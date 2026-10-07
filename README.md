# steam-error0-fix

修复 Apple Silicon Mac 上 Steam 游戏启动报 **"error 0"** 的问题：让 Steam 以原生 arm64 启动本来就有 arm64 版本的游戏，全程不需要安装 Rosetta。

- **适用系统**：Apple Silicon（M 系列）Mac + Steam 客户端（含原生 arm64 的版本）
- **实测通过**：2026-10-07，M5 Pro / macOS 27 (Darwin 27)，Turmoil（石油骚动）与《文字遊戲》两款游戏修复后均通过 Steam 正常启动
- **形式**：一个 Claude Code skill（`SKILL.md`）+ 本说明文档（`README.md`），纯 shell 命令，无任何第三方依赖

---

## 目录

- [背景：error 0 是什么](#背景error-0-是什么)
- [为什么会这样（机制）](#为什么会这样机制)
- [你的游戏能不能修：决策表](#你的游戏能不能修决策表)
- [方式一：作为 Claude Code skill 使用](#方式一作为-claude-code-skill-使用)
- [方式二：手动修复（不用 Claude Code）](#方式二手动修复不用-claude-code)
- [回滚](#回滚)
- [FAQ](#faq)
- [局限与免责](#局限与免责)

---

## 背景：error 0 是什么

在 Apple Silicon Mac 上点 Steam 的"开始游戏"，游戏没启动，只弹一个笼统的报错：

> 启动游戏时发生错误（error 0）

这不是游戏本身的问题——Steam 日志里能看到它的真身。打开：

```
~/Library/Application Support/Steam/logs/gameprocess_log.txt
```

会看到类似：

```
[2026-10-07 19:42:53] Failed running GameID 361280: "'.../Turmoil.app/Contents/MacOS/Turmoil'" (OS Error 0)
```

即 **Steam 尝试直接 exec 游戏主程序，失败了**，但真实错误码没有传回来，界面上只显示 0。

## 为什么会这样（机制）

以下机制说明是基于本机日志与二进制结构的推断（Valve 没有官方文档），但与全部观测一致：

1. 新版 Steam 客户端已包含原生 Apple Silicon（arm64）支持，客户端本身不再需要 Rosetta。
2. 很多老游戏在 Valve 的元数据里仍被标记为"Intel Mac 游戏"，Steam 会尝试以 **x86_64 方式**去启动它们。
3. 机器上没有安装 Rosetta 2 时，x86_64 程序无法执行，exec 直接失败 → Steam 报 "error 0"。
4. 其中一部分游戏（本页两个案例都是）**主程序和全部依赖库其实都带有 arm64 切片**——它们本来就能在 Apple Silicon 上原生运行，只是 Steam 没有这么启动它。

修复思路：把主程序瘦身成纯 arm64（`lipo -thin`），Steam 的启动路径就会走原生 arm64，问题消失。同时把改写后失效的代码签名用 ad-hoc 方式重签，避免内核拒载。

## 你的游戏能不能修：决策表

先弄清楚游戏装在哪、主程序是哪个：

```bash
# 列出已装游戏与 AppID（AppID 在文件名 appmanifest_<AppID>.acf 里）
grep -H '"name"' ~/Library/Application\ Support/Steam/steamapps/appmanifest_*.acf
```

游戏本体在 `~/Library/Application Support/Steam/steamapps/common/<安装目录>/`，主程序在 `<游戏>.app/Contents/MacOS/` 下（由 Info.plist 的 `CFBundleExecutable` 指定；该目录常有多个文件，别把旁边的 dylib 当主程序）。

然后检查 .app 内**所有** Mach-O 文件的架构：

```bash
A="<游戏>.app"
find "$A" -type f -exec sh -c 'file -b "$1" | grep -q Mach-O && echo "$1"' _ {} \; \
  | while read -r f; do echo "$(lipo -archs "$f" 2>/dev/null) | ${f#"$A"/Contents/}"; done
```

| 检查结果 | 结论 | 做法 |
|---|---|---|
| 主程序含 arm64，且所有依赖库都含 arm64 | ✅ **可以修** | 见下方任一方式 |
| 主程序只有 x86_64 | ❌ 瘦身无用（没有 arm64 可提取） | 安装 Rosetta |
| 主程序有 arm64，但某个依赖库只有 x86_64 | ❌ 瘦身后游戏会闪退（原生进程加载不了 Intel-only 依赖） | 安装 Rosetta |
| 已装 Rosetta 仍报 error 0 | ⚠️ 问题不在架构路由 | 看日志找别的原因 |

安装 Rosetta 的一行命令（与官方一致，需要一次管理员授权）：

```bash
softwareupdate --install-rosetta --agree-to-license
```

## 方式一：作为 Claude Code skill 使用

一键安装到你的 skills 目录（目录名 `steam-error0-fix` 与 SKILL.md 内声明的名称一致）：

```bash
git clone https://github.com/Djhhhhhh/steam-error0-fix-skill.git ~/.claude/skills/steam-error0-fix
```

或手动：把本文件夹整个放进（或软链到）`~/.claude/skills/steam-error0-fix`。

之后对 Claude 说人话即可，例如：

> Steam 里 XX 游戏启动报 error 0，帮我修一下

skill 会按固定流程执行，**每一步都有安全闸门**：

1. 确认机器是 arm64、日志里有 `(OS Error 0)` 特征记录；
2. 定位游戏的 AppID 与主程序（经 appmanifest 与 Info.plist，不靠猜）；
3. **修改前**枚举 .app 内全部 Mach-O 做架构核对，任何一个缺 arm64 都会停下并转 Rosetta 方案；
4. 备份原文件 → `lipo -thin arm64` → `codesign` ad-hoc 重签 → 校验签名；
5. 用 `open "steam://run/<AppID>"` 走 Steam 正式启动路径验证（等价于点"开始游戏"），核对进程存活与日志中的 `adding PID ... as a tracked process`；
6. 失败则自动回滚备份，并说明 Rosetta 兜底方案。

## 方式二：手动修复（不用 Claude Code）

以主程序路径 `BIN` 为例（替换成你游戏的实际路径）：

```bash
BIN="$HOME/Library/Application Support/Steam/steamapps/common/<游戏>/<游戏>.app/Contents/MacOS/<主程序>"

# 1. 备份（可选但推荐）
cp "$BIN" /tmp/$(basename "$BIN").universal.bak

# 2. 瘦身为纯 arm64（原地覆盖写法可行）
lipo "$BIN" -thin arm64 -output "$BIN"

# 3. ad-hoc 重签名（必做：改写后原签名失效会被内核拒载；
#    且不少老游戏出厂签名本来就无效，重签顺带修正）
codesign --force --sign - "$BIN"

# 4. 校验
lipo -archs "$BIN"     # 应只输出 arm64
codesign -v "$BIN"     # 无输出即有效

# 5. 启动验证（AppID 见 appmanifest 文件名）
open "steam://run/<AppID>"
```

成功后 Steam 日志应出现 `adding PID ... as a tracked process`，而不是新的 `Failed running`。

## 回滚

```bash
cp /tmp/<主程序>.universal.bak "$BIN"
```

或者更简单：在 Steam 里对该游戏执行"**验证游戏文件的完整性**"，Steam 会重新下载原始文件（error 0 也会随之回来）。

## FAQ

**Q：会不会损坏游戏？**
A：不会。`lipo -thin` 只是从二进制里提取出 arm64 切片、丢掉 Intel 半边；arm64 一侧在原文件里就是完整自洽的（修复前已核对全部依赖库）。随时可回滚。

**Q：游戏更新后 error 0 又出现了？**
A：正常。Steam 更新或"验证完整性"会把二进制还原成 universal 版。重跑第 2、3 两行命令即可，建议把命令存个备忘。

**Q：修复后性能有变化吗？**
A：正向。游戏以原生 arm64 运行，比走 Rosetta 翻译更快、更省电。

**Q：和安装 Rosetta 冲突吗？选哪个好？**
A：不冲突。要玩的游戏里如果有"纯 Intel"老游戏（没有 arm64 切片），只能装 Rosetta；本修复的优势是免装 Rosetta 且游戏原生运行。两者可以共存。

**Q：为什么 Steam 自己不直接用 arm64 启动？**
A：Steam 大概率按游戏的"Mac 平台架构"元数据决定启动方式（详见上面的机制推断），对老游戏默认走 Intel 路径。这是 Valve 侧的行为，本修复是在本地文件层面绕开它。

## 局限与免责

- 仅针对"Apple Silicon + 未装 Rosetta + Steam 报 error 0 + 游戏实际具备 arm64 能力"这一场景，不要当作万能启动失败修复来用。
- 修改的是本地游戏文件，不影响他人；Steam 验证/更新会还原文件。
- 机制解释部分为基于实测证据的推断，如 Valve 后续修改客户端行为，本方案可能失效或不再必要（届时装 Rosetta 或等官方修复均可）。

## 实测案例记录

| 日期 | 游戏 | AppID | 结构 | 结果 |
|---|---|---|---|---|
| 2026-10-07 | Turmoil（石油骚动） | 361280 | 主程序 + 7 个依赖库全部 x86_64+arm64 | ✅ 修复并通过 Steam 启动 |
| 2026-10-07 | 《文字遊戲》 | 1109570 | 主程序 + libsteam_api.dylib 均含 arm64 | ✅ 修复并通过 Steam 启动 |

两款游戏修复后均以原生 arm64 进程运行（Steam 日志从 `Failed running ... (OS Error 0)` 变为 `adding PID ... as a tracked process`）。
