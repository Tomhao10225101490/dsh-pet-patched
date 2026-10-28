# 大肥鱼桌宠 · 补丁版（新仓库说明）

这是 PC2005-cloud/dsh-pet v0.3.8 的一个补丁新仓库，不是原作者的新作，
也不是从零写的宠物。上游：https://github.com/PC2005-cloud/dsh-pet

新人从仓库根 [README.md](README.md) 安装和启动。本文件只保留许可和差异索引。
架构见 [docs/architecture.md](docs/architecture.md)。

## 许可（必须分开写，不能全写 MIT）

- 代码：MIT（上游 LICENSE，Copyright (c) 2026 PC2005-cloud）。
- 素材（动画 / 提示词 / 源视频）：允许开源使用，禁止商用。
- 角色形象：社区二创，CC BY-NC-SA 4.0，非商业。
  原型“溟月”：Bilibili @上善无形；DeepSeek 女仆鲸鱼版：@ZipZipPipe。
  因此本仓库整体不能商用，二次分发必须保留以上归属和非商业限制。

## 相对上游多出来的功能（详见 PATCHES.md）

1. 独立模式也能查 DeepSeek 余额（读本机密钥，调官方余额接口）。
2. 余额气泡常驻头顶，收起态改成贴在头顶的小药丸标签。
3. 余额显示去掉峰/谷；数字按韩元换算（USD×1325、CNY×187，2026-10-27 固定汇率）、
   符号仍用 ¥——仅显示，底层金额没变，不是真人民币，汇率会漂移。
4. 双击宠物直接开对话（Cmd+双击仍是呼出 DSH）。
5. 对话框变大、可滚动、不被窗口裁掉。
6. 对话变长（碎碎念保持短）；AI 设置里还需手动把单次输出上限设到 4096。
7. 示例配置：宠物尺寸 320、省 token 设置（碎碎念/对话配图关、历史 5 轮）。
   这是用户层选项，不在包内默认里。示例见 docs/example-user-config.jsonc。

碎碎念和对话的「本机 AI 设置页」来自这份 v0.3.8 源码，不是上面第 1 条。
第 1 条只补余额。启动横幅仍写着余额不可用，以 PATCHES.md 为准，不要以那行字为准。

## 重要：补丁在构建产物里

补丁文件是 `lib/standalone.js`、`lib/client.js`、
`runtime/electron-helper/shared-core.js`、`runtime/electron-helper/sprite.js`、
`runtime/electron-helper/events.js`、`runtime/electron-helper/index.html`，
`src/` 未动。不要直接 `npm run build`，会覆盖补丁。
见 PATCHES.md 开头和 `patches/` 里的 01–06 diff。

## 余额设置

先在宠物 AI 设置里保存 DeepSeek 密钥（只存本机），独立模式才会查余额；
密钥文件 `standalone-ai.json` 绝不能提交进仓库。
