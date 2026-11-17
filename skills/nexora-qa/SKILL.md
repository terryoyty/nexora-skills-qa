---
name: nexora-qa
description: 使用 Nexora qa 环境根据文字需求和可选本地参考图生成、查询并下载图片，自动准备运行组件，支持浏览器 OAuth 登录及原任务恢复。适用于可执行本地命令的 Agent。
---

# nexora-qa 图片生成

使用 `nexora` CLI 完成自由创作图片生成，缺少环境时先自动安装，再继续原任务。此 Skill 不提供智能套图、智能编辑或视频功能。

本 Skill 绑定 **qa**、服务 **https://vimo.qa.aukeyit.com/server**，来源为 https://github.com/terryoyty/nexora-skills-qa.git。
以下所有 `nexora` 命令均指本 Skill 精确版本的绝对入口；**每个进程**（包括 version、login、status、刷新、生成、等待、下载、登出与恢复）必须同时传入 `NEXORA_CONFIG_DIR=~/.nexora/qa`（先展开用户目录）及 `NEXORA_EXPECTED_SERVER=https://vimo.qa.aukeyit.com/server`。运行目录按 `nexora-qa/<cli.version>` 隔离，执行方式见安装文档。不复用其他 Skill 的 PATH 或凭据。

## 准备

- 首先执行 `node --version` 和 `npm --version`（Windows PowerShell 使用 `npm.cmd`），确认 Node.js 主版本至少为 22 且 npm 可用。缺失、版本不足或命令不可用时，按 [安装与登录](references/setup.md) 检查现有安装路径，必要时自动下载官方 LTS 到用户目录。安装后在当前命令环境验证并继续，不以重启 Agent 或用户手动安装为默认前置条件。
- 执行 `nexora --version`，与本 Skill 的 [release.json](release.json) 中 `cli.version` 比较。缺失或不一致时，按安装指引直接将包内 tgz 安装到用户目录并复核版本；使用该版本的绝对入口或进程 PATH 继续。
- 本 Skill 服务固定为 `https://vimo.qa.aukeyit.com/server`。执行 `nexora auth status --json`；未登录或授权失效时直接执行 `nexora login --server https://vimo.qa.aukeyit.com/server --json`，不向用户索要服务地址。切换环境应使用对应 Skill；现有凭据不符时停止，不静默改地址或复用身份。
- 环境准备和 CLI 安装是本工作流的一部分，直接执行，无需额外询问“是否安装”。若宿主确实拦截权限或网络操作，使用宿主审批机制；只有无法自动处理的阻塞才交还用户，并说明具体失败步骤。
- Token、刷新和凭据文件交给 CLI 管理；不读取、回显或要求用户粘贴凭据。CLI 与 MCP 各自登录。

## 生成并交付

1. 执行 `nexora image options --json` 获取当前默认模型、参数 schema、默认值及文件限制。只传用户要求的覆盖项，未指定的模型和模型参数由服务端决定；张数默认 1。具体参数见 [命令](references/commands.md)。
2. 将用户需求作为 `--prompt`。参考之前生成的图片时，优先将下载结果 `files[].assetId` 作为可重复的 `--reference-asset`，不下载后再上传。只有本地路径时使用 `--reference`；CLI会根据同目录下载清单、文件摘要和当前身份自动复用原资产。普通或已修改本地图正常上传。两类合计最多4张，顺序为本地路径顺序再接资产顺序；不要重复传同一图片。保留路径引号并正确传参，不转换为 base64 或由 Agent 自行上传。不要使用 fileId、下载URL或其他账号的资产ID代替assetId。
3. 用 `nexora image generate --prompt "需求" --no-wait --json` 提交，可附参考图和用户要求的其他选项。记录 JSON 中的 `data.runId`、`data.record` 和 `pollAfterSeconds`。请求记录是 CLI 自动创建的，不要求用户编写 JSON。
4. 执行 `nexora image wait RUN_ID --timeout 45 --json`。单次等待有限，长任务可在告知进度后继续原 Run；到本轮执行时间上限则交还 Run 与恢复命令。不要紧密循环查询，也不要因超时重提生成。
5. `SUCCEEDED` 或 `PARTIAL_SUCCESS` 后，用 `nexora image download RUN_ID --output "目标目录" --json` 下载。部分成功也应交付已完成文件并说明失败张数；`FAILED`/`CANCELED` 如实报告，不自动创建新收费任务。
6. 从 `data.output` 与 `data.files[].name` 得到真实图片绝对路径，确认文件存在再返回图片预览和可点击的文件链接（宿主支持时），简短说明完成情况。成功回复默认不展示 Run ID、record、manifest、assetId 等内部标识；保留它们供后续查询与恢复，仅在用户明确索要、故障排查或中断交接确有需要时提供。只有实际下载落盘才说“已下载”；受理状态和下载 URL 都不能代替已生成、已下载的事实。

## 错误与恢复

stdout 是单个 JSON envelope，stderr 是进度；同时检查进程退出码、`code`、`data.status`，不能只看 `ok`（部分成功为退出码 5）。失败后阅读 [恢复规则](references/recovery.md)，按 `error.recovery` 中的原记录或 Run 恢复，不从错误消息拼装命令。

同一任务持续使用 CLI，不切换到 MCP，也不自行调用业务 HTTP、维护 OAuth 或实现业务逻辑。缺少依赖时自动准备；需要浏览器登录和同意时说明用户应完成的网页操作，保留任务上下文，授权完成后从原位置继续。
