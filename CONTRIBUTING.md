# 贡献

本仓库是 [PC2005-cloud/dsh-pet](https://github.com/PC2005-cloud/dsh-pet) v0.3.8 的补丁检出。代码 MIT，素材与形象禁止商用。公开衍生作品请同时保留上游地址和本仓库地址。

## 先看什么

- 行为差异：[PATCHES.md](PATCHES.md)
- 怎么跑起来：仓库根 [README.md](README.md)
- 模块怎么接：[docs/architecture.md](docs/architecture.md)
- 测试与构建陷阱：[docs/development.md](docs/development.md)

和补丁无关的新产品功能，请到上游仓库讨论。本仓库接收的是：文档与真实代码不一致、补丁本身的缺陷、把补丁移植回 `src/` 的工作。

## 开口之前

用本仓库的 [Issues](https://github.com/Tomhao10225101490/dsh-pet-patched/issues) 写清：

- 你看到的行为，以及你期望的行为
- 是独立模式（`node lib/standalone.js`）还是 DSH 插件
- macOS / Windows / Linux，Node 版本，`dsh-pet` 的 `package.json` version
- 有没有跑过 `npm run build`。跑过的话，补丁产物可能已经不是仓库里那一版

不要贴 `standalone-ai.json`、`standalone-runtime.json`、`memory.json` 或任何 API key。日志里如果有密钥，先删掉再贴。

## 改代码时的约束

1. **不要把 `npm run build` 的结果直接提交。** `scripts/prepare.js` 会用 `src/` 重写 `lib/` 和 `runtime/electron-helper/shared-core.js`。余额查询、韩元显示换算、常驻药丸、双击对话、加宽对话框、对话加长，都在这些产物里，`src/` 里没有。
2. 若要移植补丁，单独做一次改动：改 `src/`，构建，再对照 `patches/` 与 [PATCHES.md](PATCHES.md) 核对六条行为还在。`patches/*.patch` 的路径是打包机上的绝对路径，不能在本仓库根目录 `git apply`。
3. 手写运行时 `runtime/electron-helper/sprite.js`、`events.js`、`index.html` 不由 tsdown 生成。改它们时要和 `shared-core.js` 一起看，避免只改了一边。
4. 不要新增根目录 `package.json` 来「方便启动」。入口就是 `dsh-pet/lib/standalone.js`。
5. 不要提交用户数据，也不要在示例里写真实密钥。可选用户配置只用 [docs/example-user-config.jsonc](docs/example-user-config.jsonc) 这种空密钥示例。
6. 源码注释沿用现有模块头：这个文件负责什么、不负责什么、哪条不变量必须成立。不要写大段复述代码的注释。

## 本地检查

在 `dsh-pet/` 目录：

```sh
pnpm install --frozen-lockfile
node lib/standalone.js --check
pnpm test
```

`pnpm test` 用 Node 自带测试运行器跑 `src/**/*.test.ts`，不启动窗口。UI 脚本是 `pnpm run test:ui`；其中 macOS 桌面用例在非 darwin 或没设置 `DSH_PET_ELECTRON` 时会跳过。

只改文档时不必跑完整测试，但启动步骤必须和 `package.json` 的 `scripts`、`src/standalone/cli.ts` 对得上。

## 提交说明

写清改了哪条用户可见的行为。如果动了 `lib/*.js`，说明这是手补构建产物，还是移植进 `src/` 之后的构建结果。后者必须在说明里写「已重新构建」，并列出仍保留的补丁。

## 许可

贡献的代码按仓库现有 MIT 许可提交。不要把商业化素材、去掉署名的形象、或上游 LICENSE 的替换稿放进来。
