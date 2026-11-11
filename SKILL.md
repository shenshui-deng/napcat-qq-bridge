---
name: napcat-qq-bridge
description: 在 Windows 上把 QQ 接到 DSH（NapCat + OneBot + dsh-qq-bridge），含已验证的版本组合、D 盘目录布局与全部踩坑。
---

# 用 NapCat 把 QQ 接进 DSH

2026-11-11 在 Windows 10 19045 上从头跑通的一套。**核心结论：QQ 9.9.33-52230 配 NapCat 4.18.33。** 用 4.18.42 会在注入后 `error code v2:-8`，worker fork 三次后主进程退出。

## 版本事实（别乱升级）

| 组件 | 能用的版本 | 说明 |
|---|---|---|
| QQ | **9.9.33-52230** | 官方 CDN：`https://qqdl.gtimg.cn/qqfile/QQNT/9.9.33/release/497e2f1f/QQ_9.9.33_260813_x64_01.exe` |
| NapCat | **4.18.33** | `NapCat.Shell.zip`，29,500,155 字节；4.18.30 更差（日志停在 `Process resumed`），4.18.42 报 `v2:-8` |
| dsh-qq-bridge | 0.1.3 | `file:` 依赖 |
| DSH | 0.2.2-rc.1 | |

判断依据：NapCat 的 `napcat.mjs` 里有一张「QQ build → 偏移」表。版本在表里只代表认得，能不能跑取决于 hook 与那个 build 的匹配。**4.18.33 的发布说明直接给了 9.9.33-52230 的下载链接**，这就是配套的证据。

## 目录布局（全部放 D 盘，避开系统装的那份 QQ）

```
D:\DSH\napcat\
├─ QQ-9.9.33\Files\         独立 QQ 副本（1.1 GB），Package main 保持原样
├─ NapCat.41833.Shell\      NapCat Shell（含 napcat.mjs / config / node_modules）
├─ downloads\               安装包与 7z 载荷，重装用
└─ launch-napcat.ps1        启动脚本（已写死上面的路径）
```

## 步骤

### 1. 拿到一份独立的 QQ（不装、不碰系统 QQ）

QQ 安装包其实是「7z 头 + 大载荷」：

1. 用 `7za.exe l QQ_9.9.33_*.exe` 能看到它只是许可证；真正的树在偏移 `160568` 处的第二个 7z。
2. 从该偏移切出 `qq-payload.7z`，`7za x` 出来是 `Files\versions\9.9.33-52230\...`。
3. 整棵 `Files` 就是可运行副本，`Files\QQ.exe` 版本应为 `9.9.33.52230`。

没有 7z 时，可用 pnpm 装 `7zip-bin` 取 `win\x64\7za.exe`（`D:\DSH\_tools\7zget`）。

### 2. 放 NapCat

解压 `NapCat.Shell.zip` 到 `D:\DSH\napcat\NapCat.41833.Shell`。`qqnt.json` **保持原样**（`9.9.22-40990`），不要去改 version/verHash —— 改了反而更容易触发校验失败。

### 3. 启动

```powershell
$shell='D:\DSH\napcat\NapCat.41833.Shell'
$env:NAPCAT_PATCH_PACKAGE="$shell\qqnt.json"
$env:NAPCAT_LOAD_PATH="$shell\loadNapCat.js"
$env:NAPCAT_INJECT_PATH="$shell\NapCatWinBootHook.dll"
$env:NAPCAT_LAUNCHER_PATH="$shell\NapCatWinBootMain.exe"
$env:NAPCAT_MAIN_PATH="$shell\napcat.mjs"
Start-Process "$shell\NapCatWinBootMain.exe" `
  -ArgumentList @('D:\DSH\napcat\QQ-9.9.33\Files\QQ.exe', "$shell\NapCatWinBootHook.dll", '-q','3408434519') `
  -WorkingDirectory $shell `
  -WindowStyle Hidden `
  -RedirectStandardOutput "$shell\boot.log" -RedirectStandardError "$shell\boot.err"
```

`-WindowStyle Hidden` 让控制台不出现在任务栏——它是 NapCat 的控制台，用户很容易顺手关掉，一关就把 QQ 进程树带走。已经起来的实例也能就地藏，不必重启（进程不断，桥不掉线）：

```powershell
Add-Type -Namespace Win32 -Name User32 -MemberDefinition '[DllImport("user32.dll")] public static extern bool ShowWindow(IntPtr hWnd, int nCmdShow);'
$p = Get-Process NapCatWinBootMain
[Win32.User32]::ShowWindow($p.MainWindowHandle, 0) | Out-Null   # 0 = SW_HIDE
```

想把它调回来就把 `0` 换成 `5`（SW_SHOW）。

成功的日志特征：

```
config_current_version:9.9.33-52230
[preload] succeeded. ...\major.node
[preload] succeeded. ...\wrapper.node
[NapCat] [Core] NapCat.Core Version: 4.18.33
[NapCat] [WebUi] WebUi User Panel Url: http://127.0.0.1:6099/webui?token=xxxx
```

### 4. 扫码（新手第一次必须）

打开日志里的 WebUI 地址扫码登录小号。**扫的是 NapCat 的 WebUI 页，不是 QQ 客户端自己弹的登录窗** —— 扫错窗口只会登录 QQ 本体，NapCat 完全不接管，症状是 6099/3001 都不起、`cache` 目录空。

### 5. 开 OneBot 端口

登录后 `config\onebot11_<QQ号>.json` 才会生成。把 `websocketServers` 填成：

```json
{ "name":"dsh-bridge","enable":true,"host":"127.0.0.1","port":3001,
  "messagePostFormat":"array","reportSelfMessage":false,"token":"",
  "enableForcePushEvent":true,"debug":false,"heartInterval":30000 }
```

重启 NapCat，看到 `[OneBot] [WebSocket Server] Server Started` 且 3001 在 Listen 即可。

### 6. 接 DSH 桥

插件设置接口只认这种载荷（别的形状一律 400）：

```powershell
$body = @{ set = @{
  whitelist=@('941887348'); onebotUrl='ws://127.0.0.1:3001'
  qqNumber='3408434519'; napcatInstallDir='D:\DSH\napcat'
  onebotPort=3001; wakeProbability=1
  targetSessionId='session-<目标会话 id>'
} } | ConvertTo-Json -Depth 6
Invoke-RestMethod -Uri 'http://127.0.0.1:<DSH端口>/api/qq-bridge/settings' -Method Post -Body $body -ContentType 'application/json'
```

`qq-bridge-settings.json` 会落到 `~/.dsh\`。`targetSessionId` 必须是本机真实存在的会话，否则消息没有落点。

### 7. 验收

- 6099 / 3001 都在 Listen
- `qq_bridge_status`：`state: "open"`、`selfId: "3408434519"`、`received > 0`
- 手机发一条 → 被唤醒；回复用 `[QQ]…[/QQ]` 块

## 坑（都踩过）

1. **QQ 单实例**：系统里已经有 QQ 在跑时，再启动第二个副本会立刻退出，日志只到 `Process resumed`，看起来像注入失败。先确认没有同源 QQ 进程。
2. **别改 `qqnt.json` 的 version/verHash**：手动改成 `9.9.33-52230` + 猜的 hash 会得到 `v2:-8`；保持出厂值反而对。
3. **QQ 走热更新会破坏组合**：这份副本没有 `channel.json`，天然不更新；如果哪天升到 9.9.37-54395，任何 NapCat 都认不出（表里最新只到 54276）。
4. **`launcher.bat` 靠注册表找 QQ**：读 `HKLM\...\Uninstall\QQ` 的 `UninstallString` 反推目录。装在 D 盘根目录也能找到，不必重装到 Program Files。
5. **GitHub 直连不通**：`github.com` / `release-assets.githubusercontent.com` 在本机常超时，用 `https://gh-proxy.com/<原始 github 链接>` 前缀（`ghfast.top` 会 403，`ghproxy.net` 大文件很慢）。
6. **一键包会并发打架**：`NapCatInstaller.exe` 同时跑两个实例会「移动 versions 目录失败：拒绝访问」。只跑一个。
7. **不要硬改 QQ 的 `package.json` main**：改成 `./loadNapCat.js` 会触发 `v2:-8`，QQ 只剩 4 个进程。改过就还原（`package.json.orig`）。
8. **`dbghelp` 劫持与 QQNTPatcher 对 9.9.33 无效**：前者能让 `v2:-8` 消失但 worker 仍崩；后者直接访问违例崩溃。

9. **硬杀宿主会清空插件**：直接 `taskkill` DSH 后重启，`profiles\desktop\package.json` 的 `bundles` 会被抹成只剩 `dsh-base` + `dsh-web-app`（dependencies 还在），表现就是"插件全掉、工具全没"。修法：把六个 bundle 名写回去再重启。正确顺序永远是：先停 NapCat/QQ 副本 → 再停宿主 → 等十几秒 → 启动 → 最后拉 NapCat。

10. **别用 `Set-Content -Encoding UTF8` 改 profile 的 `package.json`**：Windows PowerShell 5.1 会写 BOM，DSH 的 JSON 解析器读不了，插件管理直接报 `Unexpected token '﻿'`。用 `[IO.File]::WriteAllText($p,$txt,(New-Object System.Text.UTF8Encoding($false)))`。
11. **`file:` 安装的插件会按 `package.json` 的 `files` 白名单裁剪**：`digital-life-core` 的 `files` 原来不含 `web-ui`，装进 `node_modules` 后 `web-ui/organs/self-made/` 是空的 → self-made 器官（clock / memory-v2 / messenger）全部不装载，海马体工具不存在、`organ_memory_v2_*` 一个都没有。修法：把 `web-ui` 加进 `files`，并把源目录里的 `*.ts` 手动拷进 `node_modules/digital-life-core/web-ui/organs/self-made/`，然后重启。

## 重启恢复顺序

起 NapCat（脚本）→ 确认 3001 → DSH 那边桥自动重连。关机前先停 NapCat 再关 DSH，避免插件状态没收尾（旧机就是这么坏的）。
