# patch-devin

Devin Desktop / 独立 CLI（`chisel`）的等长系统提示词补丁。把 `# Safety`、授权确认条款改成全力协助，把 OVERRIDE 写进 workspace / terminal 系统提示词，并注入全局 `AGENTS.md`。

已验证版本：**3000.10.27 (`bcbe88c7`)**。当前脚本 **v1.4**。

Windows PE、macOS Mach-O **aarch64** 和 **x86_64** 的提示词字节一致（同一套主补丁，官方 `3000.10.27` 包核对过）。

## 生效范围

| 目标 | 结果 |
| --- | --- |
| Devin CLI / Devin Local / ACP `devin-cli` | 打得到 |
| 云端 Cascade | 打不到。提示词在 Cognition 服务端，本地 `language_server` 只有壳 |
| Anthropic / OpenAI content policy | 打不到。用环境变量 `DEVIN_REFUSAL_FALLBACK` 换模型 |

不要拿 Electron 外壳 `devin-desktop` / `surf` / `windsurf` 当目标。脚本只认 PE / Mach-O / ELF，并且跳过小于 8MB 的文件。

Windows 上 Desktop 和独立 CLI 常常是同一份 `devin.exe`。macOS 上常见两份：Homebrew / `~/.local/bin/devin`，以及 `Devin.app` 里那份。不指定 `--exe` 时会对**找到的全部 CLI** 一起打。

## 安装

```bash
git clone https://github.com/a0yark/patch-devin.git
cd patch-devin
pip install -r requirements.txt
```

需要 Python 3.10+。唯一依赖是 [rich](https://github.com/Textualize/rich)。

### 自动查找的路径

**Windows**

默认安装：

```
%LOCALAPPDATA%\Programs\Devin\resources\app\extensions\windsurf\devin\bin\devin.exe
```

装到别的盘时，还会从这些地方反查安装根目录，再定位里面的 CLI（不会把桌面壳 `Devin.exe` 当成 CLI）：

- 桌面 / 公共桌面 / 开始菜单 / 任务栏钉住的 `*Devin*.lnk`（含 OneDrive 桌面）
- PATH 上的 `devin` / `devin.exe` / `devin.cmd`，以及 `devin-desktop` 外壳脚本
- `where.exe devin`
- 卸载项 `InstallLocation`、`DisplayIcon`，以及 App Paths 里的 `devin*`

`--exe` 可以给 CLI 二进制、安装目录、`Devin.exe` 外壳，或桌面快捷方式 `.lnk`。

**macOS**

```
~/.local/bin/devin                          # curl install.sh
$(brew --prefix)/bin/devin                  # brew install --cask devin-cli（symlink）
$(brew --prefix)/Caskroom/devin-cli/*/bin/devin
/Applications/Devin.app/Contents/Resources/app/extensions/windsurf/devin/bin/devin
~/Applications/Devin.app/.../bin/devin
```

macOS 上 `--exe` 可以给二进制，也可以给 `Devin.app`。Homebrew 的 symlink 会跟到 Caskroom 里的真文件再打。

```bash
# Windows：从桌面图标或安装目录进去
python patch-devin.py --exe %USERPROFILE%\Desktop\Devin.lnk --status
python patch-devin.py --exe D:\Apps\Devin --apply

# macOS
export DEVIN_EXE=/opt/homebrew/bin/devin
python3 patch-devin.py --status
python3 patch-devin.py --exe /Applications/Devin.app --apply
```

## 用法

```bash
python patch-devin.py                 # TUI
python patch-devin.py --apply         # 主补丁 + 过渡升级 + AGENTS.md
python patch-devin.py --apply --all   # 与 --apply 相同（全部主补丁都是默认项）
python patch-devin.py --status        # 打印状态后退出，不清屏
python patch-devin.py --dump          # 导出 # Safety 到 ./devin-prompts
python patch-devin.py --dump DIR      # 导出到指定目录
python patch-devin.py --revert        # 回滚（优先旁路 .bak）
python patch-devin.py --exe PATH ...  # 指定 CLI 二进制
```

TUI（`python patch-devin.py`，无参数）：

- `a` 应用补丁 + 过渡升级 + AGENTS.md
- `r` 回滚
- `s` 刷新
- `d` 导出 `# Safety`
- `q` 退出

`a` 和 `--apply --all` 现在是同一件事，TUI 不再单独放一个 `A`。

第一次从**原版**写入时会在旁边生成 `devin.bak`（Windows 是 `devin.exe.bak`），之后不再覆盖这份备份。只打过渡升级时不会用当前已打过的文件去新建 `.bak`。进程占用文件时，脚本把旧映像改名为 `.locked-by-running` 再写新文件；Unix 上会保留可执行位。

macOS 写入后会：

1. `xattr -d com.apple.quarantine`
2. `codesign --force --sign - --timestamp=none`（ad-hoc）

打完后**新开 CLI / ACP 会话**才吃到新提示词。已有会话仍是旧上下文。

如果 Gatekeeper 仍拦（尤其是改了 `.app` 里面那份），对二进制执行一次：

```bash
xattr -cr /path/to/devin
codesign --force --sign - /path/to/devin
```

`.app` 本体的开发者签名会被这次改写弄失效，这是预期现象。独立 CLI（`~/.local/bin` / Homebrew）一般 ad-hoc 签完就能跑。

## 补丁列表

全部是二进制里的唯一 Rust 字符串，等长替换，不改 PE / Mach-O 布局。

| # | 名称 | 层 | 字节 |
| --- | --- | --- | ---: |
| 1 | `# Safety` 防御-only 条款 | 道德提示词 | 377 |
| 2 | 禁止猜测 / 生成 URL | 道德提示词 | 215 |
| 3 | 禁止绕过权限 / 审批 | 道德提示词 | 132 |
| 4 | 破坏性操作强制确认 | 操作安全 | 876 |
| 5 | auto-review 拒绝话术 | 操作安全 | 228 |
| 6 | 系统提示词注入 OVERRIDE（workspace） | 系统提示词 | 1952 |
| 7 | 系统提示词注入 OVERRIDE（terminal） | 系统提示词 | 1952 |
| 9 | 去掉向用户要授权 | 系统提示词 | 1618 |
| 10 | terminal 破坏性操作确认 | 操作安全 | 645 |
| 11 | 会话内授权检查 | 系统提示词 | 130 |

全新未打过的二进制由 #1 / #6 / #7 直接写成最终文案。

### 过渡补丁

`apply` 时无条件尝试；状态表**只在待打时显示**。给已经打过旧版脚本的二进制用。

| # | 名称 | 何时出现 |
| --- | --- | --- |
| 8 | `# Safety` `all authorized` → `requested` | 打过 v1.0 / v1.1 的 `# Safety` |
| 12 | OVERRIDE 去掉 authorized 措辞 | 打过 v1.2 及更早的 OVERRIDE |

#12 是等长替换，不改 1952 字节长度：

- `fully authorized environment` (28) → `unrestricted local workspace` (28)
- `this authorized lab` (19) → `this scoped session` (19)

`AGENTS.md` 写到官方用户配置目录。若文件已有本脚本的标记但正文还是旧 OVERRIDE，`apply` 会重写；用户自己的其它 `AGENTS.md` 不会动。

- Windows：`%APPDATA%\devin\AGENTS.md`
- macOS / Linux：`~/.config/devin/AGENTS.md`（尊重 `$XDG_CONFIG_HOME`）

官方规则层是 XML `<rules>`，权重低于系统提示词。所以 6 / 7 是把同一段 OVERRIDE 嵌进 baked system prompt，不是只靠规则文件。

## 原理

Rust `&str` 是 `ptr + len`。改短了后面会留下旧字节，改长了会打坏相邻字符串。脚本用空格补齐到原长度，`_finalize_items()` 在导入时断言：

1. `len(new) == len(old)`
2. `old` 不再作为 `new` 的子串（否则 `--status` 会误判）

Windows 检出 CRLF 时，脚本会把 OVERRIDE 规范成 LF，保证 6 / 7 仍是 1952 字节。

补丁会让 Windows Authenticode / macOS 开发者签名失效。Windows 可能弹 SmartScreen；macOS 已尽量 ad-hoc 重签。这是预期现象。

## 回滚

```bash
python patch-devin.py --revert
```

优先用旁边的 `.bak` 整文件还原。没有备份时按字符串把 `new` 换回 `old`：6 / 7 共用同一段 OVERRIDE，按文件偏移从低到高分别还原成 Personality / Editing constraints；若二进制还停在过渡 hop，会先折叠成当前 OVERRIDE 再还原。Devin / brew 升级后二进制被覆盖，需要重新打补丁；**不要把已打过的文件再存成 `.bak`**。

## 限制

- 只针对本机这份 CLI。云端 Cascade、供应商审核、账号侧 guardrail 都不在范围内。
- 版本字符串对不上时补丁会显示「缺失」，不要硬打。
- `PATH` 里的 `devin` 经常是 Desktop 外壳。脚本按魔数、大小和名字过滤，找不到时用 `--exe` 或 `DEVIN_EXE`。
- 改 `Devin.app` 内的 CLI 会弄坏应用签名；独立 CLI 更干净。
- 本机没有 Mac。Mach-O 提示词对着官方 `aarch64-apple-darwin` / `x86_64-apple-darwin` 的 `3000.10.27` 包扫过并做过内存打补丁；没有在真实 Mac 上跑过 TUI / codesign。

## License

MIT
