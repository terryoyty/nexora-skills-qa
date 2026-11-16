# 命令与机器输出

以下为 CMD 示例，`RUN_ID` 和路径替换为实际结果，需求字符串按所用 shell 正确传参。

```cmd
nexora auth status --json
nexora image options --json
nexora image generate --prompt "橘白猫坐在晨光中的窗边，自然写实摄影" --no-wait --json
nexora image generate --prompt "保留商品外观，背景改为海边日落" --reference "D:\图片\商品.png" --no-wait --json
nexora image generate --prompt "保留上一张图片的猫，背景改成花园" --reference-asset ASSET_ID --no-wait --json
nexora image wait RUN_ID --timeout 45 --json
nexora image status RUN_ID --json
nexora image download RUN_ID --output ".\results" --json
```

生成选项：

| 参数 | 用法 |
| --- | --- |
| `--prompt` | 新请求必填，最多 4000 字 |
| `--reference` | 本地 PNG/JPEG/WebP 路径，可重复；遵守 options 的限制（CLI 上限 4 张，每张 10MiB） |
| `--reference-asset` | 已有云端图片assetId，可重复，与本地图合计最多4张；本地图在前，资产ID在后，按各自参数顺序 |
| `--model` | 仅从实时 models 选择；省略使用当前自由创作默认模型 |
| `--count` | 默认 1，CLI 上限 4，同时遵守服务端 maxCount |
| `--ratio` / `--image-size` / `--quality` | 只在用户要求且 schema 支持时指定，否则使用服务端默认 |
| `--param key=value` | schema 内实际字段，可重复；数字和布尔按字段类型解析；不与别名重复指定 |
| `--brand` / `--conversation` | 用户明确提供且属于当前账号的 ID；否则省略，默认新会话 |
| `--output` | 下载目录；省略使用当前目录的 nexora-results/Run |
| `--no-wait` | 仅提交并返回 Run 和请求记录，Agent 首选此方式保存恢复身份 |
| `--timeout` | 单次等待秒数，最大 86400；Agent 应按执行时间预算选择较短值 |
| `--resume` | CLI 自动生成的记录路径，只与 timeout/no-wait/json 同用 |

`image options` 包含 `defaultModel`、`models`、`defaultCount`、`maxCount`、`maxReferences`、`maxFileBytes`。每个模型的 `parameterSchemaJson`、`defaultParamsJson` 为 JSON 字符串，使用时先解析。模型 ID、价格和参数值不写死在 Skill 中。

成功或明确受理输出：`{"ok":true,"code":0,"data":{...}}`。

失败输出：`{"ok":false,"code":6,"error":{"message":"...","recovery":{"action":"resume","record":"绝对路径"}}}`。`recovery` 只在有上下文时出现，详细含义见 [恢复规则](recovery.md)。

`image status`/`wait` 的 `data` 本身是状态对象；`generate --no-wait` 的 `data.status` 是受理状态字符串；`download` 的 `data.status` 是状态对象。部分成功退出码 5 仍有 `data`，不是普通 `error`。

下载结果含 `runId`、绝对 `output`、`files`、绝对 `manifest`。`files[].name` 为相对输出目录的安全文件名，将它与 output 连接得到图片路径。CLI 校验大小/摘要并在重试下载时跳过已完成文件；不要自行处理签名下载链接。

复用之前结果时用 `files[].assetId`，无需再次传图。只有本地路径时，CLI自动识别同目录的下载manifest；内容未改、身份和服务一致且原Run结果仍有效才复用。改过的图片作为新图上传；无清单的复制文件不会自动识别，请显式传原assetId（仅当仍要求原图内容）。归属或资产失效时报告错误，不通过删除清单或自行重传来绕过。此功能不清理OSS历史副本，也不对任意本地文件提供全局去重。
