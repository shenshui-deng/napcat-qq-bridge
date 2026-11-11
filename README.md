# napcat-qq-bridge

把 QQ 接进 [DeepSeek Harness](https://github.com/deepseek-ai) 的另一条路：**NapCat + OneBot + dsh-qq-bridge**。

这份仓库不是代码，是**一份踩过坑的现场记录**——2026-11-11 在 Windows 10 上从零跑通之后写的，包含验证过的版本组合、完整的启动步骤，以及十一个真实踩到的坑。

## 为什么会有这份记录

QQ 客户端一直在更新，NapCat 的注入补丁必须和具体的 QQ 构建号对上。对不上时，表现不是报错，而是**静默失败**：QQ 正常启动、界面正常、但没有 6099 WebUI、没有 3001 OneBot，NapCat 的日志一行都不出。

这份记录把「哪个版本配哪个」写死了，省得下次再从头试。

## 已验证组合

| 组件 | 版本 |
|---|---|
| QQ | **9.9.33-52230** |
| NapCat | **4.18.33**（Shell 版） |
| dsh-qq-bridge | 0.1.3 |
| DSH | 0.2.2-rc.1 |
| 系统 | Windows 10 19045 |

反面对照：

- NapCat **4.18.42** + QQ 9.9.33 → 能识别版本，但 worker fork 时 `error code v2:-8`，三次后主进程退出
- NapCat **4.18.30** + QQ 9.9.33 → 日志停在 `Process resumed`，连版本识别都没到
- 任何 NapCat + QQ **9.9.37-54395** → 版本表里没有这一档，注入不生效

## 怎么用

读 [`SKILL.md`](./SKILL.md)。它是按 DSH 技能格式写的，可以直接放进 `~/.dsh/skills/napcat-qq-bridge/SKILL.md`，也可以当普通技术文档照着做。

结构是：

1. 抽出一份独立的 QQ 副本（不碰系统装的那个）
2. 放 NapCat Shell（`qqnt.json` 保持原样）
3. 启动（环境变量 + 隐藏窗口）
4. 扫码（扫 WebUI 的码，不是 QQ 自己的登录窗）
5. 开 OneBot 3001
6. 接 DSH 桥

## 十一个坑

摘要，细节在 SKILL.md：

1. QQ 单实例：系统里已有 QQ 在跑时，再启动副本会立刻退出
2. 别手改 `qqnt.json` 的 version/verHash
3. QQ 热更新会破坏组合
4. `launcher.bat` 靠注册表反推 QQ 路径
5. GitHub 直连不稳，用 `gh-proxy.com` 前缀
6. OneKey 安装器不能并发跑
7. 硬改 QQ 的 `package.json` 入口会触发完整性校验
8. `dbghelp` 劫持与 `QQNTPatcher` 对 9.9.33 无效
9. 硬杀 DSH 会清空 profile 的 `bundles`，六个插件全掉
10. Windows PowerShell 5.1 写 JSON 会带 BOM，DSH 解析不了
11. `file:` 安装的插件按 `files` 白名单裁剪，器官源码可能根本没装进去

## 边界

- 只适用于 Windows 桌面环境。服务器无桌面请直接用 NapCat 的 Docker 镜像。
- 记录的是**个人号**的接法（OneBot / NapCat），不是官方机器人。
- 版本组合是 2026-11 的快照，QQ 一升级就可能失效——失效时先看 SKILL.md 里「版本事实」那一节。

## 许可

MIT。踩坑记录随便用，能少走几步就算没白写。
