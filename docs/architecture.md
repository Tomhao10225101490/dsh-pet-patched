# 架构

这份说明对着当前检出的目录写。启动步骤在仓库根 [README.md](../README.md)。补丁落在哪些文件，见 [PATCHES.md](../PATCHES.md)。

## 两条运行路径

```
独立模式（本检出可以跑）
  node lib/standalone.js
    → 本机只监听回环的 HTTP（默认端口 3080，占用则顺延）
    → apply(伪 ctx) 登记与 DSH 同一套宿主路由
    → 拉起 runtime/electron-helper/main.js
    → 每只 display 含 desktop/both 的宠物一个透明窗

DSH 插件（本检出还缺入口）
  cordis.patch.yml 指向包名 dsh-pet
    → package.json main = lib/index.js     ← 仓库里没有这个文件
    → 浏览器半侧 lib/client.js              ← 有，而且是打过补丁的
```

`src/app/main.ts` 是第三条路：同一个 Electron 进程里放 HTTP、宿主和 Helper。构建产物应是 `lib/app.js`，本检出没有。`pnpm run start:app` 会因为找不到它而失败。不要把它写成给用户的安装步骤。

## 源码怎么分

| 目录 | 运行时角色 |
| --- | --- |
| `dsh-pet/src/host/` | Cordis 插件主体。`index.ts` 的 `apply` 按固定顺序装配：配置、HTTP 路由、`/balance` `/pet` `/chat`、会话监听、余额与碎碎念轮询、最后才拉起桌面 Helper。 |
| `dsh-pet/src/host/balance/` | 余额提供者注册表。源码里的内置 id 是 `opencode-go`、`commandcode`、`deepseek-official`、`deepseek-account`。`standalone` 这个 id **只存在于** `lib/standalone.js` 的补丁，不在 `registry.ts`。 |
| `dsh-pet/src/client/` | 浏览器浮层。`index.ts` 没有顶层 ESM export，只把工厂交给 `window.__ModuleLoader__`。tsdown 把它打成 `lib/client.js`。 |
| `dsh-pet/src/shared/` | 不碰 DOM 宿主的纯逻辑：物理、字体、菜单结构、余额行文案、播放状态。浏览器 bundle 和桌面 `shared-core.js` 都从这里来。补丁改的 `moneyRows`（去掉峰/谷、显示换算）只在两份产物里，不在 `src/shared/balance/`。 |
| `dsh-pet/src/standalone/` | 不启动 DSH 的 CLI（`cli.ts`）、参数与路径（`options.ts`）、回环 HTTP、AI 设置的校验与 `0600` 落盘（`ai-config.ts`、`ai-storage.ts`）。设置页路由是 `/standalone-ai/`。 |
| `dsh-pet/src/app/` | 单进程应用的装配与桌面令牌。源码在，产物不在。 |
| `dsh-pet/runtime/electron-helper/` | 透明窗的主进程、预加载、渲染。`sprite.js` 管拖拽、双击、菜单；`events.js` 管气泡计时；`index.html` 里有 `.is-balance` 的样式挂钩；`shared-core.js` 是共享层的浏览器全局 `window.PetShared`。 |

`lib/standalone.js` 是 tsdown 把宿主半侧和独立模式打在一起的单文件，顶部直接 import `zod`、`openai`、`@anthropic-ai/sdk`、`@electron/get`。所以独立模式也要先在 `dsh-pet/` 安装依赖，不能只拷贝那一个 js 文件。

## 配置从哪读

`src/host/paths.ts` 把包根定成「当前模块所在目录的上一级」。构建产物在 `lib/` 里，所以上一级是 `dsh-pet/`，素材就是 `dsh-pet/assets/`。直接用 `node` 跑 `src/` 时，包根会落到 `src/`，包内 `config.jsonc` 会找不到。`--check` 的报错原文就是这个情况。

用户数据不在仓库里：

```
$DSH_HOME/                         未设置时 = ~/.dsh
├── electron/                      Electron 43.3.0 解压目录
└── dsh-pet/
    ├── main-config.jsonc          用户配置，覆盖包内对应字段
    ├── standalone-ai.json         独立模式密钥与模型。禁止入库
    ├── standalone-runtime.json    本次端口、令牌、设置页 URL。禁止入库
    ├── memory.json                对话记忆
    ├── logs.txt
    ├── main-animation/webm/       盖住包内同名动画
    └── pet/                       额外种类
```

没有「把某台电脑的桌面目录配进宠物」这一项。`DSH_PET_ELECTRON_PATH` 只在你想改用别处已经解压好的 Electron 时可执行文件时才设，macOS 上要指到 `Electron.app/Contents/MacOS/Electron`。

## 桌面窗口怎么连上宿主

Helper 由宿主进程拉起，不是用户单独下载的桥接器。

1. 宿主 spawn Electron，环境里带上 `DSH_PET_BRIDGE=1`（见 `src/host/plugin/desktop/spawn.ts`）。开发者手动跑 `scripts/start-desktop.mjs` 时不设这个变量，Helper 改走 HTTP 去拉 `DSH_PET_CONFIG_URL`。
2. `runtime/electron-helper/main.js` 看到 `DSH_PET_BRIDGE=1` 后注册自定义 scheme `dsh-pet-bridge:`。
3. 渲染进程的请求被收成一行 JSON，前缀是 `dsh-pet-bridge:`（`src/host/helper/bridge.ts` 的 `BRIDGE_PREFIX`），从管道交给宿主。
4. 宿主用已经登记的 `/dsh-pet-7340` 路由处理。视频不走管道，应答里给文件绝对路径，由 Electron 主进程自己读。
5. `DSH_PET_OPEN_TARGET=desktop` 时，菜单第一项是唤起 DSH 桌面应用；独立模式不设这个变量，第一项文案是「打开 AI 设置」，打开带令牌的 `/standalone-ai/#token=...`。令牌只应留在本机运行状态文件里。

macOS 差异写在 Helper 里，而不是另一套程序：托盘图标用模板图（深色菜单栏才看得见），左键即弹出菜单，窗口层级用 `floating` 而不是 `screen-saver`，Dock 图标隐藏，顶部边距按菜单栏处理。DPI 补偿在 darwin 上关闭，因为点坐标已经是 DIP。这些判断在 `tray-platform.js`、`topmost-watchdog.js`、`dpi-linearization.js`、`main/windows.js`。

## 余额的两条实现

| | DSH 内的官方余额 | 本仓库独立模式 |
| --- | --- | --- |
| 提供者 id | `deepseek-official` | `standalone`（只在 `lib/standalone.js`） |
| 密钥 | DSH 凭据 `DEEPSEEK_API_KEY` | `$DSH_HOME/dsh-pet/standalone-ai.json` 的 `apiKey` |
| 接口 | `GET https://api.deepseek.com/user/balance` | 同一个 URL |
| 前置条件 | DSH 当前服务商对得上 | `baseUrl` 必须包含 `api.deepseek.com`，否则抛错 |
| 展示 | 补丁后的 `moneyRows`：无峰/谷，数字按固定汇率换成带 `¥` 的整数 | 同一份展示函数（`client.js` 与 `shared-core.js` 各一份，必须同时改） |

气泡是否自动消失，由 `events.js` 和 `client.js` 里把 `bubbleTimer` 置空决定，不是配置项。对话和碎碎念气泡仍会暂时盖住它。

## 素材链不在启动路径上

仓库根的 `scripts/*.py`、`encode_hevc_alpha.sh`、`prompts/`、`video/` 用来从源视频做透明动画。桌宠运行时只读已经放进 `dsh-pet/assets/webm/` 的成品，以及用户目录里的覆盖文件。改宠物逻辑不必跑这条链。Safari 要的 HEVC `.mov` 也不在 npm `files` 白名单里，需要的话按 [MAKE-YOUR-OWN-PET.md](../MAKE-YOUR-OWN-PET.md) 另做。
