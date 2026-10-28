# dsh-pet 社区补丁版

[English](#english)

这是 [PC2005-cloud/dsh-pet](https://github.com/PC2005-cloud/dsh-pet) **v0.3.8** 的社区补丁仓库，不是上游的新版本，也不是从零写的桌宠。角色是社区二创的 DeepSeek「大肥鱼」（蓝发鲸鱼女仆），住在桌面透明小窗里，也能作为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 插件跑在网页里。

**现在这份检出可以直接跑的，是不打开 DSH 的独立桌面模式。** 已提交的补丁写在构建产物里，不在 `src/` 里。克隆之后先按下面的步骤启动，不要先跑 `npm run build`。

| | |
| --- | --- |
| 上游 | https://github.com/PC2005-cloud/dsh-pet （包版本 0.3.8） |
| 本仓库 | https://github.com/Tomhao10225101490/dsh-pet-patched |
| 插件包 | `dsh-pet/`，`package.json` 的 `version` 仍是 `0.3.8` |
| 代码许可 | MIT（Copyright (c) 2026 PC2005-cloud） |
| 素材 | 允许开源使用，禁止商用 |
| 形象 | [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh-hans)，禁止商用 |

补丁逐条说明见 [PATCHES.md](PATCHES.md)，许可与归属见 [FORK-NOTES.md](FORK-NOTES.md)。

---

## 这是什么

仓库是三件套，不只是一个插件目录：

```
① prompts/     动画生成提示词
② scripts/ + video/   源视频 → 透明动画的素材链
③ dsh-pet/     真正跑起来的桌宠（源码、已打补丁的产物、素材、桌面窗口）
```

桌宠本身做这些事：一百余段透明动画首尾相接、点击和拖拽、甩出去会反弹、右键点播动作、多只同时存在、按余额档位换动画、碎碎念、对话、跟随 DSH 会话切换工作状态。桌面模式为每只宠物开一个跟着走的透明置顶小窗，不铺满屏幕。

本仓库在上游之外多出来的行为，全部记在已提交的产物里：

- `dsh-pet/lib/standalone.js`
- `dsh-pet/lib/client.js`
- `dsh-pet/runtime/electron-helper/shared-core.js`
- `dsh-pet/runtime/electron-helper/sprite.js`
- `dsh-pet/runtime/electron-helper/events.js`
- `dsh-pet/runtime/electron-helper/index.html`

`src/` 没有这些改动。`npm run build`（`scripts/prepare.js`）会用 `src/` 重新生成 `lib/` 和 `shared-core.js`，把前三个文件里的补丁盖掉。`sprite.js`、`events.js`、`index.html` 是手写运行时，构建脚本不会重写它们，但共享层一旦被盖掉，余额标签会和手写层对不齐。所以：**日常启动不要构建。** 对照用的 diff 在 `patches/`，那些 diff 的路径是当时打包机上的绝对路径，不能在本仓库里直接 `git apply`。

---

## 和上游差在哪里

上游独立模式的旧 README 写过「余额、碎碎念、对话都不可用」。本仓库的源码（`src/standalone/`）其实已经能在本机 AI 设置页里打开碎碎念和对话。补丁是在这个基础上又改了六处：

| # | 行为 | 落在哪 |
| --- | --- | --- |
| 1 | 独立模式可以查 DeepSeek 官方余额。读本机 `$DSH_HOME/dsh-pet/standalone-ai.json` 里的密钥，请求 `https://api.deepseek.com/user/balance`。`baseUrl` 不含 `api.deepseek.com` 就直接报错，不拿别的接口去试。返回形状与上游 `deepseek-official` 相同，所以原有余额动画和气泡还能用。 | 只在 `lib/standalone.js`。DSH 宿主模式仍走 DSH 凭据，不读这个文件。 |
| 2 | 余额气泡不再 10 秒后消失。下一轮刷新（默认 1800 秒）就地改数字。对话或碎碎念气泡优先级更高，挡完之后余额标签回来。收起态改成头顶小药丸（CSS 类 `is-balance`），位置更低，贴着头顶。展开三窗口读数时仍是大盒子。 | `lib/client.js`、`shared-core.js`、`events.js`、`index.html`、`sprite.js` |
| 3 | 余额文案去掉「峰/谷」。数字按固定汇率换成整数后仍画成 `¥`：USD × 1325、CNY × 187（2026-10-27 的显示用汇率）。底层钱包金额和币种不变，这不是人民币余额，汇率也不会自己更新。 | `lib/client.js` 与 `shared-core.js` 的 `moneyRows` |
| 4 | 双击宠物打开对话。对话已经开着时不重复打开，也不关闭。拖拽过程中不触发。macOS 上 **Command + 双击** 仍是呼出 DSH，不会被抢成对话。双击前那两下单击的点击动画仍会各播一次，这是 DOM 事件顺序，不是故障。 | `lib/client.js`、`sprite.js` |
| 5 | 对话框加宽到大约 320–480px，输入区最高 320px，内部滚动，长文本不再被裁掉；变大之后会重新夹回窗口或屏幕里。 | `lib/client.js`、`shared-core.js` |
| 6 | 只在对话的 system prompt 末尾追加一段说明：「20 字以内」只约束碎碎念；对话按完整回答来写。碎碎念提示词本身没改。独立模式单次输出上限默认 1024、最高 4096，长回答要在 AI 设置里把上限调到 4096 才出得来。 | `lib/standalone.js` 的 `chatWithPet` |

另外两件容易误当成「代码功能」的事，**不在仓库里**：

- 有人把宠物宽度从包内默认 462 改成 320，并关掉碎碎念配图、对话配图、把记忆轮数设为 5。那是用户目录里的 `main-config.jsonc`，不是本仓库的默认值。可选示例见 [docs/example-user-config.jsonc](docs/example-user-config.jsonc)。
- 启动横幅里仍有一行 `不可用      余额 / 会话工作状态 / DSH 系统通知`。余额这一词是补丁没改到的旧文案。独立模式在保存了 DeepSeek 官方密钥之后，右键「查看余额」会走上面的第 1 条。工作状态和 DSH 系统通知确实仍要 DSH 宿主。

`patches/01`–`06` 是上述改动的原始 diff。细节、以及「不要把密钥提交进来」的约束，以 [PATCHES.md](PATCHES.md) 为准。

---

## 功能亮点

下面是上游就已经有、本仓库照常保留的能力。和补丁冲突的地方以上一节为准。

- **动画链**：每段（含待机）播完按权重选下一段。默认 idle 10 / turn 5 / move 5，再加分类权重，都在 `dsh-pet/assets/config.jsonc`。
- **事件动画**：余额、碎碎念、工作状态按档位播专属动画。档位可以写成候选数组，触发时档内随机，循环播放时轮换，避免连着同一段。
- **多开**：设置页或配置文件里增加 `pets[]`。每只独立大小、位置、开关。
- **屏幕漫游**：朝面朝的方向走，先看边界。多屏按各屏工作区判断。`confineToScreen: true` 时只留在当前这块屏。
- **点击 / 拖拽 / 甩抛**：点击有回应动画和 Q 弹。拖拽过阻尼弹簧跟手。甩出后抛物线飞行，碰边反弹，落地摩擦停住再弹一下。轻放则停在原地。物理在 `dsh-pet/src/shared/physics.ts`，浏览器和桌面共用。
- **右键菜单**：动作 → 分类 → 某一段。移动类点播会真的走一段。桌面端工具项包含「打开 AI 设置」（DSH 桌面应用宿主里这一项会变成「打开软件」）、查看余额、碎碎念、对话、回到初始位置、重载配置。
- **朝向**：动画可以镜像。脚底线对齐，站在同一地面上。
- **切换**：双缓冲交叉淡入。`prefers-reduced-motion` 时跳过 Q 弹和淡入。
- **余额**：按已用百分比分六档（钱袋满溢 → 金袋叮当 → 钱袋如常 → 数金皱眉 → 袋空如洗 → 分文不剩）。DeepSeek 官方看账户余额；OpenCode Zen Go 与 Command Code 看最紧迫的额度窗口。没登记余额接口的服务商不静默，改弹文字说明。每只宠物用 `balanceEnabled` 单独开关。本仓库的显示层见上一节第 2、3 条。
- **碎碎念与对话**：碎碎念按 `eventsRefreshSec.whisper`（默认 300 秒）自动来一句，也可右键手动触发。对话记忆写在 `$DSH_HOME/dsh-pet/memory.json`，浏览器和桌面读同一份。可以附图（上传、拖拽、粘贴、截屏）；图只跟这一轮请求，不进记忆、不进气泡。独立模式的开关在本机 AI 设置页，不走 DSH 凭据。
- **工作状态**：只在 DSH 里有会话可听。六档：思考 / 工作 / 整理 / 等待 / 成功 / 出错。独立模式没有这条。
- **系统通知**：DSH 窗口失焦时的系统 toast（完成、失败、截断、权限、需要选择）。独立模式没有这条。
- **桌面小窗**：`display` 为 `desktop` 或 `both` 的每只宠物一个窗口。数据走宿主进程和 Electron 之间的管道，不依赖 DSH 的 HTTP 闸门。管道由程序自己打开（环境变量 `DSH_PET_BRIDGE=1`，行前缀 `dsh-pet-bridge:`）。这不是一条要用户去填的本机路径，也不存在单独的 live-bridge 安装包。
- **pet pack**：在用户数据目录的 `pet/` 下放 `种类名-config.json` 和 `种类名-animation/`，就是一套新的动画池。见 [CONFIG.md](CONFIG.md)。
- **自定义动画**：把 VP9-Alpha 的 `.webm` 放到 `$DSH_HOME/dsh-pet/main-animation/webm/`，文件名与配置里的动画名一致（不含扩展名）。主宠物优先用这里的文件，没有再回落到包内 `dsh-pet/assets/webm/`。
- **对外接口**：DSH 页面里还有 HTTP（前缀 `/dsh-pet-7340`，只监听本机）和 Cordis 服务 `petApi7340`。契约在 [API.md](API.md)、[openapi.yaml](openapi.yaml)、[CORDIS.md](CORDIS.md)、[CORDIS-API.md](CORDIS-API.md)。

---

## 运行效果

下面八张是宠物在 DSH 网页里的实拍，文件在 `assets/screenshots/`。

<p>
  <img src="assets/screenshots/dsh-pet-running-1.png" width="380" alt="dsh-pet 在界面中的运行效果 1">
  <img src="assets/screenshots/dsh-pet-running-2.png" width="380" alt="dsh-pet 在界面中的运行效果 2">
  <img src="assets/screenshots/dsh-pet-running-3.png" width="380" alt="dsh-pet 在界面中的运行效果 3">
  <img src="assets/screenshots/dsh-pet-running-4.png" width="380" alt="dsh-pet 在界面中的运行效果 4">
  <img src="assets/screenshots/dsh-pet-running-5.png" width="380" alt="dsh-pet 在界面中的运行效果 5">
  <img src="assets/screenshots/dsh-pet-running-6.png" width="380" alt="dsh-pet 在界面中的运行效果 6">
  <img src="assets/screenshots/dsh-pet-running-7.png" width="380" alt="dsh-pet 在界面中的运行效果 7">
  <img src="assets/screenshots/dsh-pet-running-8.png" width="380" alt="dsh-pet 在界面中的运行效果 8">
</p>

透明动画的 GIF 预览在 `dsh-pet/assets/preview/`（文件名是拼音），播放用的 VP9-Alpha 视频在 `dsh-pet/assets/webm/`。GIF 里的透明区域会显示成页面底色，实际播放是透明的。分类预览见文末。

---

## 系统要求

本说明按 **macOS** 来写启动步骤。同一套代码也有 Windows（`electron.exe`）和 Linux（单文件 `electron`）分支；Linux 没有 `DISPLAY` / `WAYLAND_DISPLAY` 时会跳过桌面小窗，只打日志。macOS 不看这两个变量，有图形会话就可以拉窗。

| 项目 | 要求 |
| --- | --- |
| 系统 | macOS，能打开普通图形窗口。Intel 与 Apple 芯片都可以，Electron 按 `process.arch` 下载对应包。 |
| Node.js | `>= 22.12.0`（`dsh-pet/package.json` 的 `engines`）。发版流水线用 Node 24。 |
| 包管理器 | pnpm。锁文件是 `dsh-pet/pnpm-lock.yaml`。本机用 pnpm 10 可以按锁安装；GitHub Actions 发版用 pnpm 12。没有 pnpm 时再用 npm。 |
| 磁盘 | 仓库自带 webm 与 GIF。第一次启动另下一份 Electron 43.3.0（约一百兆这一档，视架构而定）到 `$DSH_HOME/electron`。 |
| 网络 | 首次需要能访问 Electron 镜像。默认 `https://npmmirror.com/mirrors/electron/`。 |
| 浏览器浮层 | 桌面小窗走 Electron（Chromium），macOS 上直接播包内 `.webm`。Safari / WKWebView 不认 webm 的 alpha，会变成黑底；那只影响「用 Safari 打开 DSH 网页浮层」。要在 Safari 里透明，需另备 HEVC `.mov`，见 [MAKE-YOUR-OWN-PET.md](MAKE-YOUR-OWN-PET.md)。 |
| 签名 | 自动下载的 Electron 没有苹果签名。第一次打开若被系统拦住，见下面的常见问题，不要换一条别人机器上的绝对路径来绕。 |

不需要预先安装 DSH，也不需要把某台电脑的桌面路径写进配置。数据目录只有一个规则：`$DSH_HOME`，没设置时就是用户主目录下的 `.dsh`。macOS 上即 `~/.dsh`。

---

## 安装与启动

在终端里做。路径按你克隆的位置来，不要写死某个人的主目录。

### 1. 确认 Node

```sh
node -v
```

需要 v22.12 或更高。macOS 上可以用官网安装包或 nvm。版本不够就不要继续，`@electron/get` 的引擎声明卡在这一档。

### 2. 克隆本仓库

```sh
git clone https://github.com/Tomhao10225101490/dsh-pet-patched.git
cd dsh-pet-patched/dsh-pet
```

不要克隆上游再指望得到这里的补丁。npm 上的 `dsh-pet` 也是上游包，`npm install -g dsh-pet` 不会装到本仓库。

### 3. 安装依赖

```sh
pnpm install --frozen-lockfile
```

没有 pnpm 时：

```sh
npm install
```

这一步只装 `zod`、`openai`、`@anthropic-ai/sdk`、`@electron/get` 等运行依赖。**不要**接着跑 `pnpm build` 或 `npm run build`。

### 4. 体检

```sh
node lib/standalone.js --check
```

等价写法：`pnpm run standalone -- --check`。

正常时退出码为 0，并打印类似下面的行（路径会换成你的机器）：

```text
插件            dsh-pet@0.3.8  <你的克隆目录>/dsh-pet
DSH_HOME        ~/.dsh
用户配置        ~/.dsh/dsh-pet/main-config.jsonc（不存在 → 用包内默认）
内置默认配置    <克隆目录>/dsh-pet/assets/config.jsonc
Electron        未发现（首次运行会下载到 ~/.dsh/electron）
配置条目        1 个：main
  条目 main：宠物 1 只  main(size=462, display=both)
桌面小窗        1 只
结论：配置与 Electron 就绪，可以直接启动（npm run standalone）。
```

怎么读这张报告：

- 第一行若出现 `「main-config.jsonc」的 pets 缺失或为空，已取默认宠物列表`，表示你还没有用户配置，程序改用包内 `assets/config.jsonc`。这是第一次运行的正常情况。
- `Electron 未发现` 只说明还没下载。最后那句「Electron 就绪」是固定文案，**不代表磁盘上已经有 Electron**。看上面那一行。
- `桌面小窗 0 只` 才是问题：`display` 必须是 `desktop` 或 `both`。包内默认是 `both`，所以第一次应为 1 只。
- 退出码非 0：多半是包内 `assets/config.jsonc` 读不到。确认当前目录是 `dsh-pet/`，并且跑的是 `lib/standalone.js`，不是直接跑 `src/`。

### 5. 启动

```sh
node lib/standalone.js
```

指定端口（默认 3080，被占用则向后最多试 25 个）：

```sh
node lib/standalone.js --port 3100
```

启动后终端会写下这些事实：

- 本机路由：`http://127.0.0.1:<端口>/dsh-pet-7340/`
- 实际用的配置文件
- 有几只宠物、几只会开桌面小窗
- AI 是否已经在本机设置里启用
- 那行过时的「余额不可用」（见上面的差异表，不要被它吓退）

第一次启动会下载 Electron 43.3.0 并解压到 `$DSH_HOME/electron`。macOS 上可执行文件是：

```text
~/.dsh/electron/Electron.app/Contents/MacOS/Electron
```

若你设置了 `DSH_HOME`，把 `~/.dsh` 换成那个目录。下载过程打在终端里。下载失败只影响桌面小窗；把镜像或版本改掉再启动即可，见环境变量一节。

窗口起来之后：宠物默认在屏幕右上角附近（`corner: top-right`，边距 24 / 100，宽度 462，高度按 16:9）。macOS 菜单栏里有托盘图标（`trayEnabled` 默认 true），Dock 里不放图标。左键点托盘弹出菜单。

退出：终端里 Ctrl+C，或托盘菜单里的退出，或对本机服务 `POST /shutdown`。

### 6. 打开 AI 设置并保存密钥

右键宠物，第一项是「打开 AI 设置」。它打开的是这次进程自己的本机页面（`/standalone-ai/`，地址里带一次性令牌），不是某个写死的桌面文件夹。

在页面里填写：

| 字段 | 约束 | 说明 |
| --- | --- | --- |
| protocol | `openai-compatible` 或 `anthropic` | DeepSeek 官方用前者 |
| baseUrl | 最长 2048，只能是 http/https，不能带账号、查询串或片段 | 要查余额的话，字符串里必须包含 `api.deepseek.com`，例如 `https://api.deepseek.com` |
| model | 最长 200 | 原样传给服务商，例如 `deepseek-chat` |
| apiKey | 最长 4096，不能有空白 | 只写进本机 `standalone-ai.json`，权限收成仅当前用户可读（非 Windows 是 `0600`）。页面回读时只告诉你有没有密钥，不把密钥显示回来 |
| enabled | 布尔 | 打开之后才发模型请求。打开时地址、模型和密钥必须齐全 |
| automaticWhispers | 布尔 | 自动碎碎念。默认关 |
| whisperIntervalSec | 60–86400 的整数 | 默认 300 |
| maxTokens | 128–4096 的整数 | 默认 1024。想要补丁所说的长对话，设为 4096 |
| timeoutMs | 1000–60000 的整数 | 默认 30000 |

保存之后文件在：

```text
$DSH_HOME/dsh-pet/standalone-ai.json
```

不要把这个文件、`standalone-runtime.json`（里面有本次运行的令牌）、`memory.json`、`logs.txt` 复制进仓库。`.gitignore` 已经按文件名忽略它们。文档和截图里也不要贴真实密钥。

查余额：右键「查看余额」。成功时头顶出现常驻小标签，数字是上一节说的显示换算，不是账户里的原始金额。没配密钥、或 `baseUrl` 不是 DeepSeek 官方时，气泡里会写原因，而不是静默。

自动刷新间隔是配置里的 `eventsRefreshSec.balance`，包内默认 1800 秒。手动「查看余额」不受 `balanceEnabled` 开关限制。

---

## 配置

有三层，写错的字段会告警并回落，不会静默用残缺配置：

| 层 | 位置 | 谁改 |
| --- | --- | --- |
| 包内默认 | `dsh-pet/assets/config.jsonc` | 不要为了自己的机器改这个文件并提交 |
| 用户配置 | `$DSH_HOME/dsh-pet/main-config.jsonc` | 设置页保存，或自己编辑。整体替换你写到的顶层字段 |
| 额外种类 | `$DSH_HOME/dsh-pet/pet/<名>-config.json` 与同名 `-animation/` | pet pack |

`$DSH_HOME` 默认是 `~/.dsh`。独立模式和 DSH 插件读的是同一套路径（`src/standalone/options.ts` 的 `configPathsFor` 跟宿主对齐）。

常用字段：

- `pets[]`：`id`、`name`、`size`（宽度 px，高度 = 宽 × 9/16）、`display`（`web` / `desktop` / `both` / `none`）、`position.corner` 与 `marginX` / `marginY`、`balanceEnabled`、`whisperEnabled`、`workStatusEnabled`、`fixedEnabled`。
- `eventsRefreshSec.balance` / `.whisper`：秒。余额默认 1800，碎碎念默认 300。
- `chatMemoryRounds`：每次对话带上的历史轮数。包内默认 5。`memory.json` 仍全量保存。
- `whisperImageEnabled` / `chatImageEnabled`：表情包。包内默认都是 true。
- `physics`：重力、弹性、摩擦、顶部反弹、甩抛力度、宠物互撞。
- `font`：`candy`（包内软糖体）等预设 id，加字号、是否跟着宠物缩放。
- `trayEnabled`：桌面托盘。macOS 上是菜单栏图标。
- `confineToScreen`：多屏时是否锁在当前屏。
- `clickSoundEnabled`：点击音效，包内默认 false。

独立模式只显示 `display` 含 `desktop` 或 `both` 的宠物。没有 DSH 网页，所以 `web` 那一半不会出现；`both` 仍然会开桌面小窗。

字段的完整合并规则、pet pack 目录形状，见 [CONFIG.md](CONFIG.md)。包内 `config.jsonc` 自己的注释也是契约的一部分，改配置前先读文件头那三段「字段归属」。

想用补丁作者常用的那组用户层选项（宽度 320、关掉配图、记忆 5 轮、只放桌面），复制 [docs/example-user-config.jsonc](docs/example-user-config.jsonc) 到 `$DSH_HOME/dsh-pet/main-config.jsonc` 再改。那不是程序默认值。`pets` 写了就是整表替换，示例里因而带上了必填开关。

改完用户配置后，桌面端右键「重载配置」，或重启 `node lib/standalone.js`。

---

## 环境变量

只列运行时真正会读的。没列出的不要自己发明。

| 变量 | 默认 | 作用 |
| --- | --- | --- |
| `DSH_HOME` | `~/.dsh` | 用户数据与 Electron 解压目录的根 |
| `DSH_PET_ELECTRON_PATH` | 空 | 直接指向 Electron 可执行文件。macOS 上应指向 `Electron.app/Contents/MacOS/Electron`，不是 `.app` 包本身 |
| `DSH_PET_ELECTRON_VERSION` | `43.3.0` | 缺失时下载的版本 |
| `DSH_PET_ELECTRON_MIRROR` | `https://npmmirror.com/mirrors/electron/` | 下载镜像 |
| `DSH_PET_DESKTOP_FORCE` | 未设 | 设为 `1` 时，Linux 上即使没有 `DISPLAY` 也尝试开窗。macOS 不需要 |
| `DSH_PET_STANDALONE_DEBUG` | 未设 | 设为 `1` 时独立模式打 DEBUG 日志 |
| `DSH_PET_CONFIG_URL` | `http://127.0.0.1:3080/dsh-pet-7340/config` | 只给手动启动桌面 Helper 的开发脚本用（`pnpm run start:desktop`） |
| `DSH_PET_SCALE` | `1` | 同上，开发脚本塞给 Helper 的缩放 |
| `ELECTRON_RUN_AS_NODE` | — | 启动脚本会删掉它。设着的话 Electron 会当成纯 Node，`require('electron')` 直接失败 |

`DSH_PET_BRIDGE` 由宿主在拉起 Helper 时设为 `1`。不要为了「接上某台机器」自己 export 一个路径。桥只是同一台电脑上两个进程之间的行协议：渲染进程把 `/dsh-pet-7340/...` 请求写成一行 JSON，宿主回答；视频文件回的是绝对路径，由 Electron 自己读盘，不把二进制塞进管道。

---

## 装进 DeepSeek Harness

本检出**不能**直接 `dsh plugin add file:./dsh-pet` 当完整插件用。

`package.json` 的 `main` 是 `lib/index.js`，`cordis.patch.yml` 也写明宿主入口是这个文件。当前 `dsh-pet/lib/` 里只有两份已打补丁的产物：

- `client.js`：浏览器浮层
- `standalone.js`：独立模式入口，也是 `bin.dsh-pet-standalone`

没有 `lib/index.js`（DSH 宿主半侧），也没有 `lib/app.js`（`src/app/main.ts` 那个单进程应用的构建结果）。`pnpm run start:app` 和 `pnpm dist` 因此也还不能在「不构建」的前提下使用。`scripts/dist.mjs` 目前只打 Windows 解包目录。

`pnpm run build` 会生成缺失的入口，同时用未打补丁的 `src/` 覆盖 `lib/client.js`、`lib/standalone.js` 和 `runtime/electron-helper/shared-core.js`。补丁还没移植回 `src/` 之前，不要为了装进 DSH 去构建，除非你准备好丢掉本仓库的余额、标签、双击和长对话改动。

若你已经在用上游 npm 包 `dsh-pet` 作为 DSH 插件，那是另一份代码，不包含本仓库的补丁。

DSH 兼容声明（给将来移植进 `src/` 之后用）：`0.2.0-rc.1`、`0.2.0-rc.2`、`0.2.1-rc.2` 标为 compatible，`0.1.7-rc.2` 为 incompatible。上游 README 写明开发时对着 `0.2.1-rc.2` 测过。

---

## 常见问题

**桌面上没有宠物。**  
先看 `--check` 的「桌面小窗」是不是 0。确认 `display` 是 `desktop` 或 `both`。再看终端有没有 Electron 下载失败。macOS 上若系统拦住了未签名的 `Electron.app`，到「隐私与安全性」里允许打开，或在终端对**你自己的** Electron 目录去掉隔离属性（目录跟 `DSH_HOME` 走，不要抄别人的家目录）：

```sh
xattr -dr com.apple.quarantine "${DSH_HOME:-$HOME/.dsh}/electron"
```

然后重新 `node lib/standalone.js`。

**余额数字大得不像话，前面却是 ¥。**  
这是补丁的显示换算（USD×1325、CNY×187，取整，符号仍是 ¥）。接口返回的金额没有被改写。不想要这层换算，需要改 `moneyRows` 里的 `toKRW` 并接受那是在改已打补丁的产物；见 [PATCHES.md](PATCHES.md) 第 3 节。

**右键查看余额只弹出说明，没有数字。**  
打开 AI 设置，确认已保存密钥、`enabled` 为真，且 `baseUrl` 含 `api.deepseek.com`。密钥文件损坏或为空时，错误文案是「独立模式未配置 DeepSeek 密钥」。启动横幅那行「余额不可用」可以忽略。

**对话只有一句。**  
碎碎念仍限制在大约 20 字。对话要长回复时，把 AI 设置里的单次输出上限调到 4096，并确认走的是补丁后的 `lib/standalone.js`（没有被重新构建盖掉）。

**双击先跳了两下才出对话框。**  
单击动画会先收到那两下 `click`，然后才是 `dblclick`。对话已经开着时再双击不会再开一个，也不会把它关掉。

**Command + 双击没有打开对话。**  
这是呼出 DSH 的快捷键（Windows / Linux 上是 Ctrl + 点击）。普通双击才打开对话。

**`npm run build` 之后余额标签或独立余额没了。**  
构建用 `src/` 覆盖了补丁产物。用 git 把 `lib/client.js`、`lib/standalone.js`、`runtime/electron-helper/shared-core.js` 恢复回来。不要把重新构建的未打补丁文件提交上来。

**Safari 里宠物是黑底。**  
桌面小窗不受影响。Safari 需要 HEVC-with-Alpha 的 `.mov`。仓库有 `.github/workflows/hevc-alpha.yml` 和 `scripts/encode_hevc_alpha.sh`（macOS AVFoundation）。步骤见 [MAKE-YOUR-OWN-PET.md](MAKE-YOUR-OWN-PET.md)。

**端口被占用。**  
默认从 3080 向后找空端口，最多 25 次。也可以 `--port` 指定。不要和另一个正在跑的桌宠抢同一个用户数据目录：两边会共用 `$DSH_HOME` 下的 Electron 缓存目录，上游原话是可能互相踩缓存，程序只告警。

**日志在哪。**  
`$DSH_HOME/dsh-pet/logs.txt`。独立模式没有自己的控制台窗口时也写这里。

---

## 项目结构

```
├── README.md  PATCHES.md  FORK-NOTES.md  CONFIG.md  API.md  CORDIS.md
├── CONTRIBUTING.md  CHANGELOG.md
├── docs/                  架构、开发说明、可选的用户配置示例
├── patches/               补丁参考 diff（01–06，不能直接 git apply）
├── prompts/               动画与通知图标的提示词
├── scripts/               素材链（去水印、抠像、归一化、GIF、HEVC）。不是桌宠启动脚本
├── video/                 素材链用的水印遮罩
├── tools/                 API 调试页、Cordis 探针、DPI 探针
├── assets/screenshots/    运行截图
└── dsh-pet/
    ├── src/host/          DSH 宿主半侧：配置、路由、余额、会话、拉起桌面进程
    ├── src/client/        浏览器浮层，构建后才是 lib/client.js
    ├── src/shared/        两端共用的纯逻辑（物理、菜单、余额展示、状态）
    ├── src/standalone/    独立模式 CLI、本机 AI 设置、回环 HTTP
    ├── src/app/           单进程 Electron 应用的源码（本检出没有对应的 lib/app.js）
    ├── lib/               已提交：client.js、standalone.js（含补丁）。没有 index.js
    ├── runtime/electron-helper/   透明窗口。含已打补丁的 sprite / events / shared-core
    ├── assets/            webm、preview GIF、字体、表情包、config.jsonc
    └── scripts/           构建、下载 Electron、测试注册
```

模块怎么串起来，见 [docs/architecture.md](docs/architecture.md)。测试命令和「为什么不能先构建」，见 [docs/development.md](docs/development.md)。

---

## 许可

必须分开看，不能把整个仓库当成 MIT：

- **代码**：MIT。[LICENSE](LICENSE) 的版权人是 PC2005-cloud（2026）。
- **素材**（`dsh-pet/assets` 里的动画、`prompts/`、`video/` 及由其生成的预览）：允许开源使用，禁止商用。
- **形象**：社区二创，CC BY-NC-SA 4.0。原型「溟月」为 B 站 [@上善无形](https://space.bilibili.com/4456176)；DeepSeek 女仆鲸鱼版为 B 站 [@ZipZipPipe](https://space.bilibili.com/4168597)。[声明原文](https://www.bilibili.com/opus/1247156596114456581)。不可以商业化。
- **二次公开分发**：保留上述归属、非商业限制，以及上游仓库地址 https://github.com/PC2005-cloud/dsh-pet 。本仓库是衍生作品，公开页面同时保留本仓库地址即可。

---

## 贡献

见 [CONTRIBUTING.md](CONTRIBUTING.md)。短版本：先开 issue 说你要改的行为；不要提交密钥和用户配置；不要用一次无说明的 `npm run build` 覆盖 `lib/`。问题请报到本仓库的 [Issues](https://github.com/Tomhao10225101490/dsh-pet-patched/issues)。与补丁无关、属于上游产品的功能请求，请去上游仓库。

---

## 动画预览

每一类放几张代表。全部 GIF 在 `dsh-pet/assets/preview/`，全部透明视频在 `dsh-pet/assets/webm/`。

**待机 / 转向 / 移动**

<p>
  <img src="dsh-pet/assets/preview/daiji-huxi-xiuxian.gif" width="160" alt="待机呼吸休闲">
  <img src="dsh-pet/assets/preview/dongzhangxiwang.gif" width="160" alt="东张西望">
  <img src="dsh-pet/assets/preview/pangxie-zoulu.gif" width="160" alt="螃蟹走路">
  <img src="dsh-pet/assets/preview/yuandi-piaofu-tabu.gif" width="160" alt="原地漂浮踏步">
  <img src="dsh-pet/assets/preview/yuandi-zuozhuan-benpao.gif" width="160" alt="原地左转奔跑">
</p>

**小动作**

<p>
  <img src="dsh-pet/assets/preview/youxian-hengga.gif" width="160" alt="悠闲哼歌">
  <img src="dsh-pet/assets/preview/chaoda-shenlanyao.gif" width="160" alt="超大伸懒腰">
  <img src="dsh-pet/assets/preview/xie-daima.gif" width="160" alt="写代码">
  <img src="dsh-pet/assets/preview/nvpu-quxi-liyi.gif" width="160" alt="女仆屈膝礼仪">
  <img src="dsh-pet/assets/preview/da-keshui-bei-jingxing.gif" width="160" alt="打瞌睡被惊醒">
</p>

**玩耍**

<p>
  <img src="dsh-pet/assets/preview/yuandi-zhuanxin-wan-mofang.gif" width="160" alt="原地专心玩魔方">
  <img src="dsh-pet/assets/preview/shuan-huoguo.gif" width="160" alt="涮火锅">
  <img src="dsh-pet/assets/preview/youya-nvpuwu.gif" width="160" alt="优雅女仆舞">
  <img src="dsh-pet/assets/preview/sanqiu-paojie.gif" width="160" alt="三球抛接">
  <img src="dsh-pet/assets/preview/pingkong-shenghua.gif" width="160" alt="凭空生花">
</p>

**点击 / 拖拽**

<p>
  <img src="dsh-pet/assets/preview/dianji-huiying-kaixin-yuedong.gif" width="160" alt="点击回应-开心跃动">
  <img src="dsh-pet/assets/preview/dianji-huiying-haixiu-jingya.gif" width="160" alt="点击回应-害羞惊讶">
  <img src="dsh-pet/assets/preview/beishubiao-tuozhuai-xuankong-fankui.gif" width="160" alt="被鼠标拖拽悬空反馈">
</p>

**余额六档**（显示标签是本仓库补丁；动画本身来自上游）

<p>
  <img src="dsh-pet/assets/preview/qian-dai-man-yi.gif" width="160" alt="余额-钱袋满溢">
  <img src="dsh-pet/assets/preview/jin-dai-ding-dang.gif" width="160" alt="余额-金袋叮当">
  <img src="dsh-pet/assets/preview/qian-dai-ru-chang.gif" width="160" alt="余额-钱袋如常">
  <img src="dsh-pet/assets/preview/shu-jin-zhou-mei.gif" width="160" alt="余额-数金皱眉">
  <img src="dsh-pet/assets/preview/dai-kong-ru-xi.gif" width="160" alt="余额-袋空如洗">
  <img src="dsh-pet/assets/preview/fen-wen-bu-sheng.gif" width="160" alt="余额-分文不剩">
</p>

**碎碎念 / 工作状态**

<p>
  <img src="dsh-pet/assets/preview/suisuinian-duiping-suisuinian.gif" width="160" alt="碎碎念-对屏碎碎念">
  <img src="dsh-pet/assets/preview/suisuinian-fadai-suisuinian.gif" width="160" alt="碎碎念-发呆碎碎念">
  <img src="dsh-pet/assets/preview/gongzuozhuangtai-sikao-maopao.gif" width="160" alt="工作状态-思考冒泡">
  <img src="dsh-pet/assets/preview/gongzuozhuangtai-manglu-dianan.gif" width="160" alt="工作状态-忙碌点按">
  <img src="dsh-pet/assets/preview/gongzuozhuangtai-queyue-qingzhu.gif" width="160" alt="工作状态-雀跃庆祝">
  <img src="dsh-pet/assets/preview/gongzuozhuangtai-chuitou-tanqi-maohan.gif" width="160" alt="工作状态-垂头叹气冒汗">
</p>

做自己的角色：从提示词到透明 webm 的步骤在 [MAKE-YOUR-OWN-PET.md](MAKE-YOUR-OWN-PET.md)。那条管线会改素材，和「把现成补丁跑起来」是两条路。

---

<a id="english"></a>

## English

Community patch of [PC2005-cloud/dsh-pet](https://github.com/PC2005-cloud/dsh-pet) v0.3.8. It is a desktop whale-maid pet (animations, click, drag, throw, balance, chat). This checkout runs **without DeepSeek Harness**, via the standalone CLI. Patches live in committed build artifacts, not in `src/`.

```sh
git clone https://github.com/Tomhao10225101490/dsh-pet-patched.git
cd dsh-pet-patched/dsh-pet
pnpm install --frozen-lockfile
node lib/standalone.js --check
node lib/standalone.js
```

Requirements: macOS with a GUI, Node.js `>= 22.12`. Electron 43.3.0 is downloaded on first launch to `$DSH_HOME/electron` (default `~/.dsh/electron`). Do not run `npm run build`; it regenerates `lib/` from unpatched `src/` and drops the standalone balance provider, the pinned balance pill, the display-only KRW conversion, double-click chat, and the longer chat prompt.

Right-click the pet and choose **打开 AI 设置**. The API key is stored only in `$DSH_HOME/dsh-pet/standalone-ai.json` (mode `0600`). Never commit that file. Balance works only when `baseUrl` contains `api.deepseek.com`. The `¥` figure is a display conversion (USD × 1325, CNY × 187, rounded), not the raw wallet balance.

`lib/index.js` is not in this checkout, so `dsh plugin add` against this tree is not a working install path until the patches are ported back into `src/`. See [PATCHES.md](PATCHES.md) and [docs/architecture.md](docs/architecture.md).

License: MIT for code; assets are non-commercial; the character is CC BY-NC-SA 4.0. Keep the upstream attribution: https://github.com/PC2005-cloud/dsh-pet
