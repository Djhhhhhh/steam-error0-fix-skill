---
name: steam-error0-fix
description: 修复 Apple Silicon Mac 上 Steam 游戏启动报 "error 0 / OS Error 0" 的问题。当用户报告 Steam 游戏无法启动、启动报 error 0，或 Steam 日志出现 "Failed running ... (OS Error 0)" 时使用。核心手段是 lipo 瘦身为纯 arm64 + ad-hoc 重签，让 Steam 以原生方式启动本身已具备 arm64 能力的游戏，无需安装 Rosetta。
---

# Steam on Apple Silicon "error 0" 修复

## 适用判断（三条全部满足才继续）

1. 机器是 Apple Silicon：`uname -m` 输出 `arm64`。
2. Steam 日志有特征记录。在 `$HOME/Library/Application Support/Steam/logs/gameprocess_log.txt` 中能找到目标游戏的：
   `Failed running GameID <appid> ... (OS Error 0)`
3. 目标游戏的 main 可执行文件含 arm64 切片，且 .app 内所有 Mach-O 都含 arm64（见安全检查）。

不适用时改走 Rosetta：
- `arch -x86_64 /usr/bin/true` 执行成功（Rosetta 已装）却仍报 error 0 → 问题不在架构路由，本 skill 不适用，另查日志。
- 主程序只有 x86_64，或任一依赖库缺 arm64 → 瘦身救不了（原生进程加载不了 Intel-only 依赖，会改为闪退），唯一解是 `softwareupdate --install-rosetta --agree-to-license`。

## 定位游戏

- AppID 与安装目录：`grep -H '"name"' "$HOME/Library/Application Support/Steam/steamapps/"appmanifest_*.acf`，游戏本体在 `steamapps/common/<安装目录>/`。
- main 可执行文件：`<游戏>.app/Contents/MacOS/` 下由 Info.plist 的 `CFBundleExecutable` 指定。注意该目录常有多个文件，不要把旁边的 dylib 当主程序。
- 目录名/进程名可能是繁体或英文，与 Steam 商店名不一致，以实际文件为准。

## 安全检查（修改前必做）

枚举 .app 内所有 Mach-O 并核对架构：

```bash
find "<游戏>.app" -type f -exec sh -c 'file -b "$1" | grep -q Mach-O && echo "$1"' _ {} \; \
  | while read -r f; do echo "$(lipo -archs "$f" 2>/dev/null) | ${f#*.app/Contents/}"; done
```

判读：
- 每一行都含 `arm64` → 可以修复。
- 主程序缺 `arm64`，或任一依赖（Frameworks/Plugins 下的 dylib、bundle）缺 `arm64` → 停止，转 Rosetta 方案。

## 修复

```bash
BIN="<游戏>.app/Contents/MacOS/<主程序>"
cp "$BIN" "/tmp/$(basename "$BIN").universal.bak"   # 先备份
lipo "$BIN" -thin arm64 -output "$BIN"               # 瘦身，原地覆盖可行
codesign --force --sign - "$BIN"                     # ad-hoc 重签，必做
codesign -v "$BIN" && echo valid                     # 应输出有效
```

重签不可省略：改写后原签名即失效会被内核拒载；且不少老游戏出厂签名本来就无效（codesign -v 报 "invalid Info.plist"），ad-hoc 重签顺带修正。

## 验证

```bash
open "steam://run/<AppID>"     # 等价于用户点"开始游戏"，走 Steam 正式启动路径
# 轮询最多 20 秒：pgrep -x "<进程名>"（进程名 = 主程序文件名，含空格要加引号）
tail -3 "$HOME/Library/Application Support/Steam/logs/gameprocess_log.txt"
```

- 成功标志：日志出现 `adding PID ... as a tracked process`，且没有新的 `Failed running`；进程持续存活（存活超过数秒说明依赖库加载无误）。
- 失败处理：从备份恢复 `cp /tmp/<主程序>.universal.bak "$BIN"`，改用 Rosetta 方案。

## 收尾告知用户

- Steam 的"验证游戏文件完整性"或游戏更新会把二进制还原成 universal 版，error 0 随之复现——重跑修复段的两行命令即可。
- 修复后游戏以原生 arm64 运行，性能优于 Rosetta 翻译；本修复与安装 Rosetta 不冲突。
