# DeepSeek 原生搜索（DeepSeek Native Search）

> **社区重构版 v0.2.0** · 原作者：芸 · 本项目由社区开发者重构并归档发布 · License: MIT

> Contact: kuilinan@qq.com

---

## 一、项目简介

这是一个 **基于社区重构** 的 Operit 工具包插件，调用 **DeepSeek 官方独立搜索接口**（`POST https://api.deepseek.com/v1/tools/search`），返回结构化的可引用来源列表。

**核心卖点：按次计费，不烧模型 Token。**

旧版（0.1.x）走的是 Anthropic 兼容的 Messages API + `web_search_20250305` 服务端工具，本质是“让大模型自己去搜”，每次搜索都要按**模型输入/输出 Token** 计费，而且要求模型跑一遍推理，开销很高、延迟也高。

0.2.0 改为直接调用 **官方独立的搜索接口**：

- ❌ 不再经过任何大模型
- ❌ 不再消耗模型 Token
- ✅ 只看“搜索”动作本身，按调用次数计费
- ✅ 接口返回结构化 `sources[]`，直接拿 url / 标题 / 摘要 / 发布日期

### 返回字段

| 字段 | 说明 |
|---|---|
| `url` | 来源链接 |
| `title` | 标题 |
| `snippet` | 短摘要 |
| `summary` | 长摘要（无短摘要时的回退字段） |
| `publishedAt` | 发布时间（ISO 8601 UTC，无法解析时为 null） |
| `siteName` | 站点名 / 域名 |
| `authority` | 权威度分级 |
| `rerankScore` | 相关度得分（接口返回 `rerank_score`） |

> ⚠️ 本接口**不返回模型生成的答案**（没有 `answer` 字段）。它只搜集可引用的来源；答案要由你自己的大模型基于这些来源来综述。如果你的用法要求“直接拿到答案”，请改用一个能写综述的前置流程。

---

## 二、安装与配置

### 1. 下载插件包

从右侧 Release 区域下载最新版本的：

```
operit.deepseek_native_search-v0.2.0.toolpkg
```

### 2. 导入到 Operit

在 Operit 中导入这个 `.toolpkg` 包（与导入其他本地工具包的方式一致：选择本地文件 → 选中本包 → 安装）。

**一个重要的相容性约定：** 本轮的 `toolpkg_id` **保持原值**：

```
com.operit.deepseek_native_search
```

未更改为新标识，目的是让你可以基于已有的安装**直接覆盖升级**（往本地装新包时就会替换旧版本，而不多出一个异构插件）。

### 3. 自己接 Key（本次必须）

本插件不附带任何密钥，你需要自己接入。任选一种：

**方式 A：Operit 环境变量（推荐）**

设置为：

```
DEEPSEEK_API_KEY = sk-向你自己的 DeepSeek 官方 Key
```

**方式 B：配置文件**

将自己的 Key 写入：

```
/root/.deepseek_native_search_key
```

并限制权限：

```bash
chmod 600 /root/.deepseek_native_search_key
```

⚠️ **不要把密钥写入任何会提交到 Git 的明文文件里。** 本项目已在 `.gitignore` 中将其隔离。

如果你没配置：插件**不会**默默回退到任何假结果；它会返回一段明确的错误提示和接入指南，这比拿到一堆乱数据要安全。

---

## 三、计费声明

> **本接口按次计费，裸调不享受客户端免费政策，具体价格以官方为准。**

详细说明：

| 场景 | 是否计费 |
|---|---|
| 每次调用 `POST /v1/tools/search` | ✅ **按次计费**（每次调用都产生扣费） |
| 是否消耗模型 Token | ❌ 不消耗 |
| DeepSeek 官方 App / 官方智能体中的搜索 | 可能因客户端政策免费，**与本插件所用端口无关** |
| 特定生态客户端（如 Claude Code 之类夹带搜索配额） | 可能有免费窗口，**直接 HTTP 调用不享受** |
| 定价、阶梯、免费额度变动 | 完全以 **DeepSeek 官方文档 / 官方计费页** 为准，本项目不做任何价格承诺 |

再次强调：本项目是**非商业的社区重构**。我们不持有该接口、不做代理充值、也不转售任何配额；你的**耗量与账单全部直接挂在你自己 DeepSeek 账号上**。请在下手前先看官方计费页。

---

## 四、更新日志（Changelog）

### v0.2.0

- **框架迁移**：由 Anthropic 兼容的 `Messages API + web_search_20250305` 完全迁移至 **独立 Search API**（`POST /v1/tools/search`）。
- **大幅降低 Token 开销**：旧形态每搜一次都要让模型把结果“读一遍再写一篇”，Token 与延迟双高；新版直接让搜索接口吐结构化来源，**零模型 Token**。
- **请求体重新设计**：`query`、`maxResults`、`timeoutMs`三个参数覆盖主流程；并对最大结果数做了边界限制。
- **输出契约重写**：统一返回 `sources[]`，内包含 `url`/`title`/`snippet`/`summary`/`publishedAt`/`siteName`/`authority`/`rerankScore`。
- **稳健解析**：`rerank_score` → `rerankScore`、`date` → 归一成 ISO 8601 UTC 的 `publishedAt`、`summary` 作为 `snippet` 的回退。
- **空结果不逞能**：查询无结果时返回空 `sources` 阵列并附带提示，不捏造占位内容。
- **元数据规范**：`version` 升到 `0.2.0`，`license` 定死为 MIT；保留原作者字段 `author`。

### v0.1.1

- 修正工具包装载阶段子包注册排版失误引起的潜在加载异常。

### v0.1.0

- 首次发布：基于 Messages API + 服务端搜索工具实现基本可用版本。

---

## 五、参考信息

- 官方端点：`https://api.deepseek.com/v1/tools/search`
- 鉴权：包头 `Authorization: Bearer <你的 DEEPSEEK_API_KEY>`
- 物料关系：包根含 `manifest.json`、`dist/main.js`、`dist/packages/deepseek_native_search.js`
- 依赖：纯 HTTP + JSON，**零运行时依赖**

---

## 六、隐私说明

本插件不自建服务器、不作任何缓存、不内置任何上报通道。它把你的搜索词和控制参数直接转发给 api.deepseek.com，回包原样返回给你，中间不存在第三方中转。

搜索内容会抵达 DeepSeek 侧，这是调用该服务的必然结果；本插件不额外扩大这个暴露面。

如需联系维护者，请使用 README 顶部的邮箱。
