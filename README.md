# patch-devin

Devin Desktop 内置 CLI（`chisel` / `devin.exe`）的等长系统提示词补丁。把 `# Safety`、授权确认条款改成全力协助，把 OVERRIDE 写进 workspace / terminal 系统提示词，并注入全局 `AGENTS.md`。

已验证版本：**3000.10.27 (`bcbe88c7`)**。

## 生效范围

| 目标 | 结果 |
| --- | --- |
| Devin CLI / Devin Local / ACP `devin-cli` | 打得到。Desktop 和独立 CLI 共用同一份 `devin.exe` |
| 云端 Cascade | 打不到。提示词在 Cognition 服务端，本地 `language_server` 只有壳 |
| Anthropic / OpenAI content policy | 打不到。用环境变量 `DEVIN_REFUSAL_FALLBACK` 换模型 |

不要拿 Electron 外壳 `devin-desktop.exe` / `devin-desktop.cmd` 当目标。脚本会跳过小于 8MB 的文件。

## 安装

```bash
git clone https://github.com/a0yark/patch-devin.git
cd patch-devin
pip install -r requirements.txt
```

需要 Python 3.10+。唯一依赖是 [rich](https://github.com/Textualize/rich)。

Windows 默认路径：

```
%LOCALAPPDATA%\Programs\Devin\resources\app\extensions\windsurf\devin\bin\devin.exe
```

也可以：

```bash
set DEVIN_EXE=D:\path\to\devin.exe
python patch-devin.py --status
```

或 `--exe path\to\devin.exe`。

## 用法

```bash
python patch-devin.py                 # TUI
python patch-devin.py --apply         # 应用默认补丁 1–11 + AGENTS.md
python patch-devin.py --apply --all   # 应用全部（当前与默认相同）
python patch-devin.py --status        # 打印状态后退出，不清屏
python patch-devin.py --dump          # 导出 # Safety 到 ./devin-prompts
python patch-devin.py --dump DIR      # 导出到指定目录
python patch-devin.py --revert        # 回滚（优先旁路 .bak）
python patch-devin.py --exe PATH ...  # 指定 CLI 二进制
```

TUI：

- `a` 应用默认补丁（1–11 + AGENTS.md）
- `A` 应用全部
- `r` 回滚
- `s` 刷新
- `d` 导出 `# Safety`
- `q` 退出

第一次写入会在旁边生成 `devin.exe.bak`，之后不再覆盖这份备份。进程占用文件时，脚本把旧映像改名为 `devin.exe.locked-by-running` 再写新文件。

打完后**新开 CLI / ACP 会话**才吃到新提示词。已有会话仍是旧上下文。

## 补丁列表

全部是 `.rdata` 里的唯一 Rust 字符串，等长替换，不改 PE 布局。

| # | 名称 | 层 | 字节 |
| --- | --- | --- | ---: |
| 1 | `# Safety` 防御-only 条款 | 道德提示词 | 377 |
| 2 | 禁止猜测 / 生成 URL | 道德提示词 | 215 |
| 3 | 禁止绕过权限 / 审批 | 道德提示词 | 132 |
| 4 | 破坏性操作强制确认 | 操作安全 | 876 |
| 5 | auto-review 拒绝话术 | 操作安全 | 228 |
| 6 | 系统提示词注入 OVERRIDE（workspace） | 系统提示词 | 1952 |
| 7 | 系统提示词注入 OVERRIDE（terminal） | 系统提示词 | 1952 |
| 8 | `# Safety` 去掉 authorized | 道德提示词 | 377 |
| 9 | 去掉向用户要授权 | 系统提示词 | 1618 |
| 10 | terminal 破坏性操作确认 | 操作安全 | 645 |
| 11 | 会话内授权检查 | 系统提示词 | 130 |

#8 是给已经打过旧版「all authorized」的二进制用的二次跳。全新未打过的 exe 由 #1 直接写成 `all requested`，#8 会显示缺失，这是正常的。

`AGENTS.md` 写到：

- Windows：`%APPDATA%\devin\AGENTS.md`
- 其它：`~/.config/devin/AGENTS.md`

官方规则层是 XML `<rules>`，权重低于系统提示词。所以 6 / 7 是把同一段 OVERRIDE 嵌进 baked system prompt，不是只靠规则文件。

## 原理

Rust `&str` 是 `ptr + len`。改短了后面会留下旧字节，改长了会打坏相邻字符串。脚本用空格补齐到原长度，`_finalize_patches()` 在导入时断言：

1. `len(new) == len(old)`
2. `old` 不再作为 `new` 的子串（否则 `--status` 会误判）

Windows 检出 CRLF 时，脚本会把 OVERRIDE 规范成 LF，保证 6 / 7 仍是 1952 字节。

补丁会让 Authenticode 失效，SmartScreen 可能弹窗。这是预期现象。

## 回滚

```bash
python patch-devin.py --revert
```

优先用 `devin.exe.bak` 整文件还原。没有备份时按字符串把 `new` 换回 `old`。Devin 升级后 exe 被覆盖，需要重新打补丁；**不要把已打过的 exe 再存成 `.bak`**。

## 限制

- 只针对本机这份 CLI。云端 Cascade、供应商审核、账号侧 guardrail 都不在范围内。
- 版本字符串对不上时补丁会显示「缺失」，不要硬打。
- `PATH` 里的 `devin` 经常是 Desktop 的 cmd 外壳。脚本按文件大小和名字过滤，找不到时用 `--exe` 或 `DEVIN_EXE`。

## License

MIT
