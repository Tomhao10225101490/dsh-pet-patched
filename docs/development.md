# 开发说明

在 `dsh-pet/` 里操作。Node.js 需要 `>= 22.12`。依赖锁是 `pnpm-lock.yaml`。

```sh
pnpm install --frozen-lockfile
```

## 不要先构建

`package.json` 的 `build` 指向 `scripts/prepare.js`，它依次跑：

1. `pnpm run bundle`（tsdown → `lib/`）
2. `pnpm run build:desktop-core`（`src/shared` → `runtime/electron-helper/shared-core.js`）
3. `pnpm run types`（`lib/types`，git 不跟踪）

然后把 `package.json` 的 `files` 重写成发布白名单。这一步是幂等的，但 **tsdown 和 desktop-core 会覆盖已提交的补丁**：

- `lib/standalone.js` 里的 `standaloneBalance`、对话加长提示
- `lib/client.js` 里的双击、对话框尺寸、余额药丸、`moneyRows` 换算
- `shared-core.js` 里的同一套 `moneyRows` 与气泡几何

`runtime/electron-helper/sprite.js`、`events.js`、`index.html` 不在这条生成链上，构建不会把它们还原。只构建、不还原共享层的话，手写层和共享层会不一致。

日常验证用已提交产物：

```sh
node lib/standalone.js --check
node lib/standalone.js
```

`--check` 不下载 Electron、不开窗。退出码 0 只说明包内配置读得到。报告里「Electron 未发现」仍表示第一次真正启动才会下载。固定文案「结论：配置与 Electron 就绪」不要理解成磁盘上已经有二进制。

## 脚本

| 命令 | 实际做什么 | 本检出 |
| --- | --- | --- |
| `node lib/standalone.js` | 独立模式。`package.json` 里叫 `pnpm run standalone` | 可以，这是用户路径 |
| `node lib/standalone.js --check` | 打印 Node、`DSH_HOME`、配置来源、桌面宠物数量 | 可以 |
| `pnpm test` | `node --experimental-strip-types --import ./scripts/test-register.mjs --test src/**/*.test.ts` | 测的是 `src/`，不覆盖只存在于 `lib/` 的补丁 |
| `pnpm run test:ui` | `scripts/tests/*.test.mjs` | macOS 几何用例需要 `process.platform === 'darwin'` 且设置 `DSH_PET_ELECTRON` |
| `pnpm run ensure:electron` | 把 Electron 43.3.0 解压到 `$DSH_HOME/electron` | 可以。启动独立模式时也会按需下载 |
| `pnpm run start:desktop` | 只拉 Helper，配置 URL 默认 `http://127.0.0.1:3080/dsh-pet-7340/config` | 要已有服务在提供配置。不设 `DSH_PET_BRIDGE` |
| `pnpm run start:app` | 用本机 Electron 跑 `lib/app.js` | **本检出没有 `lib/app.js`** |
| `pnpm run build` | 见上，覆盖补丁 | 移植补丁进 `src/` 之前不要跑 |
| `pnpm dist` | `scripts/dist.mjs`，electron-builder，只要 `--win dir` | 面向 Windows 打包，且依赖完整 `lib/` |
| `pnpm run dev:mock` | `scripts/dev/mock-server.mjs` | 本地假服务，不是用户启动方式 |
| `pnpm run lint` / `format` / `typecheck` | ESLint、Prettier、`tsc --noEmit` | 改 `src/` 时再用 |

`prepack` 会构建并跑 lint、循环依赖、运行时 ESLint、openapi 检查。发布 npm 包不是本补丁仓库的当前路径；npm 上的 `dsh-pet` 仍是上游。

## 测试覆盖不到的地方

`src/host/balance/registry.ts` 的内置列表没有 `standalone`。因此 `pnpm test` 全绿也不表示独立模式余额还在。改完产物后至少：

- `node lib/standalone.js --check` 退出码 0
- 在 `lib/standalone.js` 里还能搜到 `standaloneBalance` 和 `api.deepseek.com/user/balance`
- 在 `lib/client.js` 与 `shared-core.js` 里还能搜到 `toKRW` 与 `is-balance`
- `events.js` 里余额气泡不是 `setTimeout` 之后自动收起（补丁把 `bubbleTimer` 置为 `null`）

macOS 桌面几何、HiDPI 菜单和点击穿透在 `scripts/tests/desktop-macos.test.mjs`。它跳过的条件是：不是 darwin，或没给 `DSH_PET_ELECTRON`。这个用例使用隔离的假配置服务，不读用户的 `standalone-ai.json`。

## 环境

与用户文档相同的变量见 README。开发时多出来的只有这些：

- `DSH_PET_ALLOW_ELECTRON_DOWNLOAD=1`：`scripts/dist.mjs` 在本机没有解压目录时，允许 electron-builder 自己下载。不设则直接失败退出。
- `DSH_PET_ELECTRON_DIST`：打包时指向已解压的 Electron 目录。
- `ELECTRON_MIRROR`、`ELECTRON_BUILDER_BINARIES_MIRROR`：`dist.mjs` 默认 npmmirror。
- `CSC_IDENTITY_AUTO_DISCOVERY=false`：`dist.mjs` 会自己设上，避免本机没有证书时签名探测失败。
- `DSH_PET_DPI_PROBE`、`DSH_PET_SMOKE`：Helper 内部探针，不是用户配置。

删掉 `ELECTRON_RUN_AS_NODE` 是启动脚本的责任（`start-app.mjs`、`start-desktop.mjs`）。空字符串也会让 Electron 中止，必须是删除该键。

## 素材与 CI

- `.github/workflows/release-app.yml`：tag `v*` 时在 `windows-latest` 上打包 `dsh-pet-win-x64.zip`，挂到**已经存在**的 GitHub Release。不负责写更新说明。
- `.github/workflows/hevc-alpha.yml`：手动触发的 macOS HEVC 透明转码，产物给 Safari，不进默认 npm `files`。

根目录 `scripts/` 是视频管线（水印、绿幕、归一化、GIF、HEVC），工作目录和中间文件（`step01/` 一类）不在插件启动路径上。中间产物不要提交。
