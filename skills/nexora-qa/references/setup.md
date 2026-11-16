# 自动安装与登录

## 交付物与执行方式

定位当前 `SKILL.md` 所在目录，不从工作目录猜包路径。发布目录的 `release.json` 给出 `cli.version`、`cli.package`、`cli.sha256` 和 `node`。`cli.package` 是相对 Skill 根的 tgz 路径；缺少清单或 tgz 表示源码目录或不完整包，应获取完整发布包，不猜版本或安装同名在线包。

直接执行准备与安装，不只给用户命令，不额外询问是否安装。优先复用满足要求的运行时；需要补齐时在用户目录安装官方 Node.js 24 LTS（含 npm）与包内 CLI，不要求 pnpm、TypeScript、管理员权限或重启 Agent。安装依赖可能访问 npm registry，不承诺离线安装。宿主实际阻止操作时走宿主审批；仍不可用时报告具体阻塞。

## 先检查现有 Node.js/npm

在 Agent 实际执行命令的环境检查 Node.js 22+ 与 npm：

```powershell
node --version
npm.cmd --version
```

macOS/Linux 使用 `npm --version`。命令找不到不代表未安装：Windows 用 `Get-Command node,npm.cmd -ErrorAction SilentlyContinue`，检查 Node 所在目录、`$env:LOCALAPPDATA\Programs\nodejs` 与 `$env:ProgramFiles\nodejs` 是否有完整的 Node/npm。macOS/Linux 用 `command -v node`、`command -v npm`，检查现有版本管理器、`/opt/homebrew/bin` 与 `/usr/local/bin`。找到后将该目录加入当前进程 PATH，或使用完整入口，再验证。

仅 npm 不可用时也先检查现有完整安装，仍不满足才安装下述运行时。不删除旧 Node 目录或改系统 PATH。安装位置必须是 Agent 的实际执行环境；Windows 宿主与 WSL/容器不能混用安装。

## 自动补齐 Node.js/npm

仅在现有环境不满足要求时执行对应平台步骤。下载官方发布包及 SHA-256 清单，校验通过后解压；运行时保留版本目录。以下变量由 Agent 执行时设置，不交给用户填写。

### Windows（PowerShell）

```powershell
$ErrorActionPreference = 'Stop'
$nexoraArch = switch ([System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture.ToString()) {
    'X64' { 'x64' }
    'Arm64' { 'arm64' }
    default { throw '当前系统架构不在此安装流程支持范围内' }
}
$nexoraBase = 'https://nodejs.org/dist/latest-v24.x'
$nexoraSums = (Invoke-WebRequest "$nexoraBase/SHASUMS256.txt" -UseBasicParsing).Content
$nexoraPattern = '(?m)^([a-fA-F0-9]{64})\s+(node-v24\.[0-9]+\.[0-9]+-win-' + $nexoraArch + '\.zip)\r?$'
$nexoraMatch = [regex]::Match($nexoraSums, $nexoraPattern)
if (-not $nexoraMatch.Success) { throw '官方清单中未找到对应的 Node.js LTS 包' }
$nexoraFile = $nexoraMatch.Groups[2].Value
$nexoraRuntimeRoot = Join-Path $env:LOCALAPPDATA 'nexora\runtimes'
$nexoraTemp = Join-Path ([System.IO.Path]::GetTempPath()) ('nexora-node-' + [guid]::NewGuid())
New-Item -ItemType Directory -Path $nexoraTemp -Force | Out-Null
$nexoraZip = Join-Path $nexoraTemp $nexoraFile
Invoke-WebRequest "$nexoraBase/$nexoraFile" -OutFile $nexoraZip -UseBasicParsing
if ((Get-FileHash -LiteralPath $nexoraZip -Algorithm SHA256).Hash.ToLowerInvariant() -ne $nexoraMatch.Groups[1].Value.ToLowerInvariant()) {
    throw 'Node.js 下载摘要不匹配，停止安装'
}
New-Item -ItemType Directory -Path $nexoraRuntimeRoot -Force | Out-Null
$nexoraNodeDir = Join-Path $nexoraRuntimeRoot ($nexoraFile -replace '\.zip$', '')
if (-not (Test-Path -LiteralPath $nexoraNodeDir)) {
    Expand-Archive -LiteralPath $nexoraZip -DestinationPath $nexoraRuntimeRoot
}
$env:PATH = "$nexoraNodeDir;$env:PATH"
node --version
if ($LASTEXITCODE -ne 0) { throw 'Node.js 验证失败' }
npm.cmd --version
if ($LASTEXITCODE -ne 0) { throw 'npm 验证失败' }
```

### macOS / Linux / WSL（Bash）

```bash
set -euo pipefail
case "$(uname -s)" in
  Darwin) nexora_os=darwin ;;
  Linux) nexora_os=linux ;;
  *) echo '当前系统不在此安装流程支持范围内' >&2; exit 1 ;;
esac
case "$(uname -m)" in
  x86_64|amd64) nexora_arch=x64 ;;
  arm64|aarch64) nexora_arch=arm64 ;;
  *) echo '当前架构不在此安装流程支持范围内' >&2; exit 1 ;;
esac
nexora_base=https://nodejs.org/dist/latest-v24.x
nexora_temp=$(mktemp -d)
curl -fsSL "$nexora_base/SHASUMS256.txt" -o "$nexora_temp/SHASUMS256.txt"
nexora_file=$(awk -v suffix="-$nexora_os-$nexora_arch.tar.gz" '$2 ~ /^node-v24\.[0-9]+\.[0-9]+-/ && substr($2, length($2)-length(suffix)+1) == suffix { print $2 }' "$nexora_temp/SHASUMS256.txt")
test -n "$nexora_file"
curl -fsSL "$nexora_base/$nexora_file" -o "$nexora_temp/$nexora_file"
awk -v file="$nexora_file" '$2 == file' "$nexora_temp/SHASUMS256.txt" > "$nexora_temp/selected.sha256"
if command -v sha256sum >/dev/null 2>&1; then
  (cd "$nexora_temp" && sha256sum -c selected.sha256)
else
  (cd "$nexora_temp" && shasum -a 256 -c selected.sha256)
fi
nexora_runtime_root="$HOME/.local/share/nexora/runtimes"
mkdir -p "$nexora_runtime_root"
nexora_node_dir="$nexora_runtime_root/${nexora_file%.tar.gz}"
if [ ! -d "$nexora_node_dir" ]; then
  tar -xzf "$nexora_temp/$nexora_file" -C "$nexora_runtime_root"
fi
export PATH="$nexora_node_dir/bin:$PATH"
node --version
npm --version
```

运行时若因系统库不兼容而无法执行（例如 Alpine/musl），根据实际系统使用其包管理器安装支持的 Node22+ 与 npm，再验证。版本检查失败后不能继续。成功后可删除本次下载的临时文件，保留运行时目录。

## 自动安装包内 CLI

Node/npm 通过后检查本 Skill 专属入口的 `--version`。与清单 `cli.version` 一致则复用；缺失或不一致时自动安装到按 Skill 和版本隔离的用户目录。Agent 读取清单、解析 tgz 绝对路径并校验 SHA-256，然后执行安装，不等待用户决定是否安装或替换。不要复用 PATH 上其他环境的同名命令。

Windows PowerShell（Agent 先将 `$nexoraSkillRoot` 设置为本次 Skill 的真实绝对目录）：

```powershell
$ErrorActionPreference = 'Stop'
$nexoraRelease = Get-Content -LiteralPath (Join-Path $nexoraSkillRoot 'release.json') -Raw | ConvertFrom-Json
$nexoraPackage = Join-Path $nexoraSkillRoot $nexoraRelease.cli.package
if ((Get-FileHash -LiteralPath $nexoraPackage -Algorithm SHA256).Hash.ToLowerInvariant() -ne $nexoraRelease.cli.sha256) {
    throw 'CLI 安装包摘要不匹配，停止安装'
}
$nexoraPrefix = Join-Path $env:LOCALAPPDATA ('nexora\cli\nexora-qa\' + $nexoraRelease.cli.version)
npm.cmd install --global --prefix "$nexoraPrefix" "$nexoraPackage" --no-audit --no-fund
if ($LASTEXITCODE -ne 0) { throw 'CLI 安装失败' }
$env:PATH = "$nexoraPrefix;$env:PATH"
$nexoraCli = Join-Path $nexoraPrefix 'nexora.cmd'
$env:NEXORA_CONFIG_DIR = Join-Path $env:USERPROFILE '.nexora\qa'
$env:NEXORA_EXPECTED_SERVER = 'https://vimo.qa.aukeyit.com'
$nexoraVersion = & $nexoraCli --version
if ($LASTEXITCODE -ne 0 -or $nexoraVersion.Trim() -ne $nexoraRelease.cli.version) { throw 'CLI 版本验证失败' }
```

macOS/Linux：同样从清单读取真实 tgz 路径，用 `sha256sum` 或 `shasum -a 256` 校验摘要。将 `nexora_prefix` 设为 `$HOME/.local/share/nexora/cli/nexora-qa/<cli.version>`（版本取自清单），执行 `npm install --global --prefix "$nexora_prefix" "实际tgz绝对路径" --no-audit --no-fund`。每次调用使用绝对入口，并传 `NEXORA_CONFIG_DIR="$HOME/.nexora/qa" NEXORA_EXPECTED_SERVER="https://vimo.qa.aukeyit.com" "$nexora_prefix/bin/nexora" --version` 复核版本。

用户目录 prefix 可直接安装所需版本，无需改写系统级旧版本。若 Agent 每次命令启动独立进程，每次都传入同一运行时目录与 CLI 目录的 PATH（包括 npm 安装阶段），或使用绝对入口；不能假设上一条命令的 PATH 修改会自动保留。Windows PowerShell 优先 `.cmd`，避免 `npm.ps1` 执行策略造成假失败；CMD 使用 `set "PATH=实际Node目录;实际CLI目录;%PATH%"`。安装失败先处理退出码和错误，不能当作已完成进入登录。

## OAuth（qa 环境）

本 Skill 已配置 origin **`https://vimo.qa.aukeyit.com`**，无需用户提供地址。每条独立进程命令都要重新设置本环境变量和绝对入口。Windows PowerShell 示例：

```powershell
$env:NEXORA_CONFIG_DIR = Join-Path $env:USERPROFILE '.nexora\qa'
$env:NEXORA_EXPECTED_SERVER = 'https://vimo.qa.aukeyit.com'
# $nexoraCli 为上文按 nexora-qa 和清单版本解析的绝对入口。
& $nexoraCli login --server https://vimo.qa.aukeyit.com --json
& $nexoraCli auth status --json
```

本环境要求 HTTPS，不允许降级协议。 地址不含 `/server` 或 `/app-api`；不从前端业务 API 路径猜测 OAuth origin。连接失败报告 qa 服务不可达，不切换服务。

CLI 自动发现 OAuth、登记公共客户端、生成 PKCE、监听本地回调并打开浏览器。Agent 直接启动并保持登录进程运行，等待用户在网页完成账号登录和同意，无需再问是否启动登录。浏览器未弹出时将 stderr 的授权地址提供给用户打开，不代替用户确认授权。登录完成后复核 status 的 server 和 identity，继续此前的生成或原 Run 恢复，不要求用户重复需求。不要把 Token 当作登录参数，也不读取凭据文件。

已有登录且服务与目标一致时直接复用；用户明确要求切换账号时重新登录。现有登录属于其他服务时停止并报告配置不符，保留原连接，不为排错执行 logout 或删除凭据。Windows PowerShell 后续每条 CLI 调用均设置：

```powershell
$env:NEXORA_CONFIG_DIR = Join-Path $env:USERPROFILE '.nexora\qa'
$env:NEXORA_EXPECTED_SERVER = 'https://vimo.qa.aukeyit.com'
```

macOS/Linux 每条进程调用都前置 `NEXORA_CONFIG_DIR="$HOME/.nexora/qa" NEXORA_EXPECTED_SERVER="https://vimo.qa.aukeyit.com"` 并使用该 Skill/版本的绝对入口。不得迁移或覆盖旧 `~/.nexora/credentials.json`。配置目录不是图片输出目录；CLI 不读取 MCP 授权信息。
