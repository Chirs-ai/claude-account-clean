# claude-account-clean

清理本机上某个 Claude 账号的登录凭据与文本形式的账号标识。输入一个邮箱，脚本会识别对应的账号配置，删除已确认归属的 `.credentials.json`，并把可扫描文本文件中的邮箱、账号 UUID、组织 UUID、显示名等标识替换为 `[removed]`。

仅依赖 Python 3 标准库，单文件脚本。默认只预览，加 `--apply` 才会真正修改文件。

> 这是文件级清理工具，不是“全盘擦除所有痕迹”的工具。`--apply` 不创建备份，且不可回滚。

## 特性

- 默认预览（PREVIEW），列出将被修改的文件及动作，不改动任何内容。
- 只处理目标邮箱的账号；其他账号的配置与凭据受保护，若处理会修改受保护文件则中止。
- 检测到目标配置目录正在被 Claude 进程使用时，`--apply` 拒绝执行（退出码 2）。
- 执行前预检候选文件（inode、mtime、大小）是否在扫描期间变化。
- 执行后复查本次修改的文件是否仍含已知标识。
- 不输出令牌内容；异常时也不打印异常参数。

## 环境要求

- Linux（通过 `/proc` 检测运行中的 Claude 进程）
- Python 3，仅标准库
- 以保存 Claude 配置的用户身份运行，不要用 `sudo`（默认扫描范围取决于当前用户的 HOME）

## 安装

```bash
mkdir -p ~/.local/bin
cp claude-account-clean ~/.local/bin/
chmod +x ~/.local/bin/claude-account-clean
```

确认 `~/.local/bin` 在 PATH 中，或直接使用完整路径调用。

## 快速使用

1. 预览，不修改文件：

   ```bash
   claude-account-clean 'old@example.com'
   ```

2. 核对输出的文件列表后，执行清理：

   ```bash
   claude-account-clean 'old@example.com' --apply
   ```

3. 需要额外扫描其他目录时：

   ```bash
   claude-account-clean 'old@example.com' --root /path/to/data --apply
   ```

## 参数一览

| 参数 | 说明 |
| --- | --- |
| `email` | 必填，目标账号邮箱；匹配不区分大小写，格式不合法时报错退出 |
| `--apply` | 实际删除/改写文件；不指定则只预览 |
| `--root PATH` | 额外扫描目录，可重复；不替换默认的用户主目录，目录必须已存在 |
| `-h`, `--help` | 显示帮助 |

输出动作含义：`delete-credentials`（删除凭据文件）、`remove-account`（移除配置中的账号信息及缓存字段）、`redact`（替换文本中的已知标识）。

## 清理范围

- 默认扫描整个当前用户主目录（含隐藏目录）加上 `--root` 指定的目录。
- 文件名包含 `.claude.json` 且 `oauthAccount.emailAddress` 与输入邮箱一致的配置，视为目标账号配置。
- 凭据只删除已确认归属的配置目录下的 `.credentials.json`：`~/.claude/`，以及父目录名以 `.claude` 开头的独立 `.claude.json` 所在目录。仅出现在备份中的配置不用于推断凭据归属。
- 在可读取的文本文件中，把收集到的标识替换为 `[removed]`（邮箱不区分大小写，其他标识精确匹配）。
- 跳过：`.git`、`node_modules`、`__pycache__`、符号链接、硬链接、含 NUL 的二进制文件、超过 128 MiB 的文件；`.json` / `.jsonl` 改写后无法通过 JSON 校验的文件也会跳过。

## 不做的事

- 不创建备份。
- 不执行云端账号删除，不撤销服务器端令牌，不取消订阅。
- 不调用 `claude logout`，不结束 Claude 进程，不登出其他账号。
- 不处理数据库、压缩包、系统快照、外部备份、进程内存和磁盘底层残留。
- 不删除整个 Claude 配置目录，也不清理目录名或历史记录中的旧配置路径引用。

## 安全注意事项

- 先预览再 `--apply`；预览输出的路径本身可能含个人信息。
- 默认会改写主目录下其他应用的文本日志和会话记录，请核对文件列表。
- 执行不是事务：中途出错或中断可能只完成一部分，无法自动回滚（见退出码）。
- 先退出目标账号的 Claude 会话，并让其他可能写入旧内容的程序停止。
- 保留账号依赖本地配置能正确识别归属；非标准布局的配置请自行检查。
- 清理后建议确认保留账号仍正常：`env -u CLAUDE_CONFIG_DIR claude auth status`（独立配置目录用 `CLAUDE_CONFIG_DIR="$HOME/.claude-a" claude auth status`）。

## 退出码

| 退出码 | 含义 |
| --- | --- |
| `0` | 预览完成，或执行完成且复查通过 |
| `1` | 处理异常或复查失败，可能已发生部分修改 |
| `2` | 参数错误，或 `--apply` 时发现目标 Claude 进程仍在运行 |
| `130` | 用户中断，可能已发生部分修改 |

## FAQ / 故障排查

**提示 `Target account has running Claude processes`？**
关闭使用该账号配置目录的 Claude 会话后重跑。预览模式只提示，`--apply` 会退出且不修改文件。

**输出 `Cleanup stopped (RuntimeError)`？**
为避免泄露凭据，异常信息不会打印。常见原因：目标标识与其他账号重叠；某个受保护账号文件含目标标识；候选文件在扫描期间被改动。处理后重新预览。

**`Planned file changes: 0` 是否表示已清理干净？**
不是。可能配置已被删除而无法推导 UUID/令牌，或残留位于脚本不处理的位置。

**`verification failures` 不为 0？**
说明有被修改的文件仍含已知标识（可能有程序在写回）。停止相关程序后重新预览并重跑。

**`--root` 会替换主目录吗？**
不会，主目录始终被扫描。

**普通姓名被误替换？**
显示名、全名（长度至少 5）也会被替换，预览只列文件、不展示逐处差异，请留意。

## 详细说明

完整的清理逻辑、输出字段、限制与验证记录见 [claude-account-clean.md](claude-account-clean.md)。
