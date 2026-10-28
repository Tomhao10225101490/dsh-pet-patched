# 本仓库相对上游的改动（已用 diff 核对，不是凭印象）

上游：PC2005-cloud/dsh-pet v0.3.8，代码 MIT，素材（动画/提示词/源视频）允许开源使用、禁止商用。
角色素材另有社区二创归属：原型“溟月” Bilibili @上善无形，DeepSeek 女仆鲸鱼版 @ZipZipPipe，CC BY-NC-SA 4.0，非商业。

## 重要提醒：补丁在构建产物里，不在 src/ 里

本次改动全部在以下构建产物中，`src/` 未动：
- `lib/standalone.js`
- `lib/client.js`
- `runtime/electron-helper/shared-core.js`
- `runtime/electron-helper/sprite.js`
- `runtime/electron-helper/events.js`
- `runtime/electron-helper/index.html`

因此 **不要直接 `npm run build`**，重构建会把补丁覆盖掉。
本目录 `patches/` 里是原始版 → 补丁版的完整 diff（01–06），
先构建、再对构建产物打补丁，或者直接用仓库里已打好补丁的构建产物。
长期正确做法是把这些补丁移植回 `src/` 再构建，本次打包未做这一步，
README 里必须如实写明。

## 1. 独立模式也能查 DeepSeek 余额（lib/standalone.js）
- 新增 `standaloneBalance` provider，登记进 BUILTIN_PROVIDERS。
- 不走 DSH 凭据，直接读本机 `$DSH_HOME/dsh-pet/standalone-ai.json` 里保存的
  DeepSeek 密钥 + baseUrl，调 `https://api.deepseek.com/user/balance`。
- baseUrl 不是 `api.deepseek.com` 时直接报错，不乱查。
- 返回格式和 `deepseek-official` 一模一样（money 形状 + wallets + tier），
  所以宠物自带的余额动画 + 气泡照常工作。
- 密钥只在本机读，不打印、不外发。`standalone-ai.json` 本身绝不能进仓库。
- 上游独立模式原文：余额、碎碎念、对话“不可用”；本补丁只解决了余额，
  碎碎念/对话在独立模式下的行为仍以上游说明为准。

## 2. 余额常驻头顶 + 小药丸标签（lib/client.js、shared-core.js、events.js、index.html、sprite.js）
- 余额气泡不再 10 秒自动收起：三处定时器全部改成 `bubbleTimer = null`。
  下一轮 1800 秒刷新直接在原地更新数字；对话/碎碎念气泡优先级更高，
  弹出来暂时盖住余额，结束后余额自动回来。
- 收起态余额不再用大白泡泡，改成头顶小药丸：`is-balance`，无小尾巴，
  padding 3px/10px、字号 0.8×、圆角 pill。
- 位置从 `100% - 尺寸×0.108` 改成 `100% - 尺寸×0.22`，320 体型约低 36px，
  标签压在头顶上而不是悬在头顶上方。展开态（三窗口完整读数）仍用大盒子。

## 3. 余额显示：去掉峰/谷，数字按韩元换算、符号仍用 ¥（仅显示）
- `moneyRows`（client.js、shared-core.js 两处同步改）：
  不再显示（峰/谷）档位括号，只显示 `余额 ¥x.xx`。
- 数字换算（2026-10-27 固定汇率，取整数，显示用）：USD × 1325、CNY × 187，
  符号一律 ¥，不是 ₩，也不是 💰。
- 底层钱包金额和币种完全没变，这只是“看着多”的显示把戏，不是真的人民币。
  汇率是写死常量，会随市场漂移；想改回真实金额，把 `toKRW` 换回原版
  `"$"/"¥" + w.total` 即可。

## 4. 双击宠物直接开对话（lib/client.js、runtime/electron-helper/sprite.js）
- 浏览器端：命中区 `onDoubleClick` → `mountChatDialog`；
  桌面端：sprite `dblclick` → `showChatFromMenu`。
- 已开着对话时不重复打开、也不关掉；拖拽中不触发；
  Cmd+双击仍是呼出 DSH，不抢成对话。
- 注意：双击前两次单击动画照样会播一次，这是 DOM 固有顺序，不是故障。

## 5. 对话框变大、可滚动、不被裁掉（lib/client.js、shared-core.js）
- 宽度 160–340px → 320–480px；输入框 `max-height: 320px`，内部滚动
  （`overflow-y: auto`），不再 `overflow: hidden` 被裁。
- 对话框长大时重新夹在窗口/屏幕内，不会伸出屏幕外。

## 6. 对话变长，碎碎念保持短（lib/standalone.js）
- 只在 `chatWithPet` 里给 system prompt 追加一段对话特别说明：
  “20字以内”只适用于碎碎念，对话要像专业 Agent 一样详细、完整、
  结构清晰，展开步骤、理由和例子，除非用户明确要求简短。
- 碎碎念本身的提示词没改，仍保持短。
- 另注意：独立模式 AI 的单次输出上限默认 1024、最高 4096，
  想让长回复真正出来，还要在宠物 AI 设置里把单次输出上限设到 4096，
  这一步是用户在设置页手动做的，不是代码改的。

## 7. 不是代码改动、不要误当成代码功能（用户层配置）
以下来自 `$DSH_HOME/dsh-pet/main-config.jsonc`，不在仓库代码里，
打包时只应给一份示例配置，不能上传用户本人的配置文件：
- 宠物尺寸 462 → 320（约 2/3，气泡字号随之缩小）。
- 省 token 设置：`whisperImageEnabled=false`、`chatImageEnabled=false`、
  `chatMemoryRounds=5`（历史只在手动对话时发，不是自动发）。
- `balanceEnabled=true`。

## 8. 不属于本仓库的东西（打包时排除）
- `node_modules/`、嵌套的重复 `dsh-pet/` 目录。
- `~/.dsh/dsh-pet/standalone-ai.json`（DeepSeek 密钥）、`memory.json`（对话记忆）。
- 用户本人的 `main-config.jsonc`、LaunchAgent 自动启动配置、
  桌面启动器（`~/Desktop/大肥鱼.*`）、Electron 缓存。
- 上游 `package.json` 里的 `homepage` / `repository` 仍指向
  PC2005-cloud/dsh-pet， fork 或新仓库发布前必须改成自己的地址，
  否则会误导别人以为这是上游原仓库。
