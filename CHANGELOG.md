# 更新记录

包版本号仍是 `dsh-pet/package.json` 里的 `0.3.8`，与上游 [PC2005-cloud/dsh-pet](https://github.com/PC2005-cloud/dsh-pet) 的这一版对齐。这里只记**本仓库相对那一版多出来的事**。上游在导入本仓库时带进来的功能史，以本仓库 git 历史和上游仓库为准，不在这里逐条重写。

## 未发布

### 文档

- 重写根目录 README：本仓库是什么、与上游的六处差异、macOS 上的安装与启动、配置、环境变量、常见问题、为什么不能先构建。
- 新增 [CONTRIBUTING.md](CONTRIBUTING.md)、[docs/architecture.md](docs/architecture.md)、[docs/development.md](docs/development.md)、[docs/example-user-config.jsonc](docs/example-user-config.jsonc)。
- `dsh-pet/package.json` 补上社区补丁说明、`repository.directory`，以及 electron / macos / deepseek / standalone / animation 关键词。
- `.gitignore` 不再忽略整个 `dsh-pet/` 目录；改为忽略依赖、构建目录，以及 `standalone-ai.json`、`standalone-runtime.json` 这类本机文件。

## 0.3.8-patched（2026-10-28）

相对上游 v0.3.8 的行为补丁，写在构建产物里，`src/` 未改。细节见 [PATCHES.md](PATCHES.md)。

- 独立模式注册 `standalone` 余额提供者：读本机 `standalone-ai.json`，只接受 `baseUrl` 含 `api.deepseek.com` 的 DeepSeek 官方余额接口。
- 余额气泡改为常驻；收起态是头顶小药丸（`is-balance`），不再 10 秒后消失。
- 余额数字按 USD×1325、CNY×187 取整后仍显示为 `¥`。底层金额不变。汇率取 2026-10-27，写死在 `moneyRows`。
- 双击打开对话。Command+双击（macOS）仍呼出 DSH。
- 对话框加宽并可在内部滚动。
- 对话 system prompt 追加「20 字只约束碎碎念」；碎碎念提示词不变。长回答还要把 AI 设置里的 `maxTokens` 调到 4096（上限，默认仍是 1024）。

同一天的仓库元数据：

- `package.json` 的 `homepage`、`repository`、`bugs` 指向本仓库。
- 插件目录内的短 README 指向本仓库，并保留上游归属。

### 这份检出里没有的东西

- `dsh-pet/lib/index.js`（DSH 宿主入口）
- `dsh-pet/lib/app.js`（单进程应用入口）
- 任何 API 密钥或个人 `main-config.jsonc`

启动横幅仍打印「余额不可用」。那行文案没有跟着补丁改；独立模式在配好 DeepSeek 官方密钥后，右键「查看余额」会走上面的余额提供者。
