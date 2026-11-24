# DeepAI Code 简体中文界面补丁（非官方）

把 [DeepAI Code](https://deepai.org/chat/ai-code) 桌面版（Windows / macOS）的英文界面就地替换为简体中文的**社区非官方补丁**。

> **这是什么？** DeepAI Code 目前没有内置语言切换。本工具读取你本机已安装应用里的 `server.asar`，用简体中文词典替换其中的英文 UI 文案，重新打包后替换原归档。**不下载、不重新分发任何原始应用文件**，每个用户都在自己的安装上现场构建补丁。

本项目 fork 自 [@TomoyaSasao12 的日语补丁](https://github.com/TomoyaSasao12/deepai-code-ja-patch)，在其工具链和思路基础上编写了简体中文词典与代码补丁。

---

## 免责声明（务必阅读）

- **非官方项目**，与 DeepAI 官方无关，也未获其支持。
- 本工具会**改写已安装应用的 `server.asar`**。执行前会自动备份，`restore` 可还原。
- **应用自动更新后补丁会失效**，需要重新执行 `apply`（新版本需先在 [versions.json](versions.json) 中确认支持）。
- 请**完全退出 DeepAI Code（含托盘图标）后**再执行 `apply` / `restore`。
- 修改应用文件有风险，请自行评估；出问题可用 `restore` 或手动把备份复制回去。
- DeepAI、DeepAI Code 为各自权利人的商标。

## 支持版本

| DeepAI Code | 状态 |
| --- | --- |
| 0.3.50（Windows） | ✅ 本机实测通过 |
| 0.3.45 / 0.3.44 / 0.3.37 / 0.3.36 / 0.3.30 / 0.3.20 / 0.3.19 | 上游日语补丁已列出，理论可用（中文词典按 0.3.50 编写，低版本可能有文案对不上） |
| macOS | 🧪 未验证（代码路径已兼容，欢迎验证 PR） |
| Linux | 官方目前无 Linux 安装包；CLI 为将来预留了适配 |

新版本验证后请更新 `versions.json`。

## 快速开始

**环境要求**：Node.js 20+（无第三方依赖）；有 Edge / Chrome 时，`apply` 会自动做一次无头浏览器启动探测，探测不通过不会替换文件。

### Windows 双击脚本

仓库提供三个脚本，默认安装目录为 `D:\DeepAI Code`，如不在该路径可把安装目录作为第一个参数传入：

| 脚本 | 作用 |
| --- | --- |
| `apply-zh-patch.cmd [安装目录]` | 退出检测 → 构建 → 启动探测 → 备份 → 替换 |
| `restore-zh-patch.cmd [安装目录]` | 从最新备份恢复原版英文界面 |
| `apply-when-closed.ps1` | 后台等待应用退出后自动 `apply`（日志写入 `apply.log`） |

> **本 fork 的脚本改动**：原版脚本用应用自带的 `DeepAI Code.exe` 配合 `ELECTRON_RUN_AS_NODE=1` 运行 `patch.mjs`，但 `patch.mjs` 的进程检测会把这个自身进程误判为“应用仍在运行”，导致始终无法打补丁。本仓库的脚本改为调用 PATH 中的独立 `node`（需 20+），规避了该问题。

### 命令行

```bat
node bin\patch.mjs status  --resources "D:\DeepAI Code\resources"
node bin\patch.mjs apply   --resources "D:\DeepAI Code\resources"
node bin\patch.mjs restore --resources "D:\DeepAI Code\resources"
```

`apply` 的执行流程：

1. 自动检测安装路径（找不到时用 `--resources` 指定）
2. 校验版本是否在 `versions.json` 中（`--force` 可跳过）
3. 若已打补丁，先自动还原最新备份再重新构建（幂等，无需手动 `restore`）
4. 检测应用是否在运行，运行中则拒绝执行
5. 从**你自己的** `server.asar` 构建中文归档，逐条目重算并校验 SHA-256
6. 无头浏览器启动探测（有 Edge / Chrome 时）：实际拉起 UI 检查 JS 异常，**不通过就不替换**
7. 备份 → 替换 → 哈希比对，不一致自动回滚

常用参数：`--skip-probe` 跳过启动探测，`--force` 忽略版本警告。

开发者命令：

```bat
node bin\patch.mjs build --out test.asar        REM 只构建，不替换
node bin\patch.mjs probe --asar test.asar        REM 只对产物做启动探测
```

## 当前补丁覆盖范围

针对 0.3.50 构建：1938 条替换规则中 1918 条命中，共替换 **3592 处文案**，涉及 90 个文件；另有 15 处代码补丁处理带变量的动态文案（如“在 &lt;编辑器&gt; 中打开”“关闭 &lt;标签页&gt;”）。

词典分为 20 个文件，覆盖设置页、侧边栏、命令面板、快捷键、配对与设备、审批对话框、远程主机、浏览器预览、主题与外观、用量与诊断等界面。

未覆盖部分（保持英文）：

- 20 条在本版本未匹配到对应文案的键（多为远程主机 / SSH 页面）
- 模板中带 `${...}` 且中文语序不同的动态文案（需逐条写 `code-patches.json`）
- 智能体输出、代码块、仓库文件内容（不属于界面文案，不翻译）

## 工作原理

- UI 位于 `server.asar` 内的 `apps/server/dist/client`（React SPA），英文文案直接嵌在压缩后的 JS 里，没有 i18n 机制。
- `dictionaries/*.json` 以“英文原文 → 简体中文”为键做精确匹配替换后重新打包；编号靠后的文件优先。
- 运行时才拼出来的字符串（如快捷键标题）由 `code-patches.json` 中的代码补丁处理；`_` 开头的词典是其配套数据。
- 翻译原则：内部用于逻辑比较的标识（`Success` / `Failure` 等）、快捷键 WHEN 条件表达式、产品名 / 字体名 / 按键名原则上不翻译。

## 与上游日语补丁的区别

| 项目 | 上游 ja-patch | 本仓库 zh-patch |
| --- | --- | --- |
| 目标语言 | 日语 | 简体中文 |
| 词典 / 代码补丁 | 日语 | 自行编写的中文词典与补丁 |
| 启动脚本 | 用 `DeepAI Code.exe` 自运行 Node | 改用独立 `node`，修复自进程检测误报 |
| 工具链结构 | 相同 | 沿用 |

## 贡献

欢迎 PR 完善翻译：

1. 在 `dictionaries/*.json` 增加 `"English string": "简体中文"`（按编号选文件，靠后的优先）
2. 注意不要翻译上文中提到的“不应翻译的语”
3. 用 `node bin\patch.mjs build --out test.asar` 确认能正常构建、探测通过后再提交

## 许可证

MIT，详见 [LICENSE](LICENSE)。本仓库沿用上游日语补丁的许可证文件。

## 最后

如果 DeepAI Code 官方推出了多语言支持，本仓库将归档，届时请改用官方功能。
