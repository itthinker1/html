# Netlify 三种部署模式完全指南

> 面向「想用 AI 助手自动把 HTML 页面托管到 Netlify」的开发者：本文带你吃透 **匿名部署、Draft 草稿部署、Deploy Preview 部署预览** 三种部署模式，并把 **生产部署（--prod）** 作为对比参照，帮你分辨"什么时候该用哪种、哪些免费、哪些扣积分"。

---

## 目录

1. [一、概述与前置准备](#一概述与前置准备)
2. [二、三种部署模式详解](#二三种部署模式详解)
   - [模式一：匿名部署（Anonymous Deploy）](#模式一匿名部署anonymous-deploy)
   - [模式二：Draft 草稿部署（CLI 手动）](#模式二draft-草稿部署cli-手动)
   - [模式三：Deploy Preview 部署预览（Git 集成自动）](#模式三deploy-preview-部署预览git-集成自动)
3. [三、三模式对比表](#三三模式对比表)
4. [四、生产部署与积分说明](#四生产部署与积分说明)
5. [五、AI 自动化完整流程示例](#五ai-自动化完整流程示例)
6. [六、常见问题与坑](#六常见问题与坑)

---

## 一、概述与前置准备

### 1.1 三种模式一句话速览

| 模式 | 触发方式 | 是否需要账号 | 是否扣积分 | URL 形态 |
|------|----------|--------------|------------|----------|
| **匿名部署** | CLI 手动 | ❌ 不需要 | ❌ 免费 | 随机临时 URL |
| **Draft 草稿** | CLI 手动 | ✅ 需要 | ❌ 免费 | 随机独立 URL（可固定） |
| **Deploy Preview** | Git 集成自动 | ✅ 需要 | ❌ 免费 | 每个 PR 独立 URL |

> ⚠️ **注意**：三种模式**都不扣积分**。只有「生产部署（--prod）」才消耗积分。这是选择"临时预览"套路时最爽的一点。

### 1.2 安装 Netlify CLI

本机已装 Node.js / npx，二选一即可：

```bash
# 方式一：全局安装（推荐）
npm i -g netlify-cli

# 方式二：不全局安装，直接用 npx 调用
npx netlify-cli deploy ...
```

验证安装：

```bash
netlify --version
```

### 1.3 认证：无需浏览器，用 Personal Access Token

AI 自动化场景下，**不要**走 `netlify login` 的浏览器交互登录（会卡住）。改用 API Token：

1. 登录 Netlify 后台 → **User settings** → **Applications** → **New access token**，生成一个 Personal Access Token。
2. 导出到环境变量：

```bash
export NETLIFY_AUTH_TOKEN=你的token
```

3. 之后所有命令即可自动读取该 token；也可以逐条显式传：

```bash
netlify deploy --auth $NETLIFY_AUTH_TOKEN ...
```

> 💡 **关于 `--json`**：所有命令都建议加 `--json`，让输出变成 JSON 而不是人类可读的横幅（banner），这样 AI/脚本能稳定地解析出 `deploy_url`、`url`、`site_id` 等字段。**不要解析 stdout 打印的横幅**，格式不可靠。

---

## 二、三种部署模式详解

## 模式一：匿名部署（Anonymous Deploy）

### 是什么

**无需登录、无需账号**即可部署的模式。命令会生成一个**临时实时 URL**，适合一次性分享、AI 测试、临时预览。

### 步骤命令

```bash
netlify deploy --dir=输出目录 --no-build --allow-anonymous --json
```

### 特点

- **零门槛**：不需要任何账号、不登录，一条命令直接出 URL。
- **临时性**：生成的临时 URL **1 小时内有效**，1 小时内可认领，超时未认领的站点会被删除。
- **可认领**：1 小时内通过 `netlify claim --site <site-id> --token <drop-token>` 认领，或登录 Netlify 认领。
- **防滥用限制**：每个 IP 最多创建 **3 个匿名站点**。
- **默认密码保护**：匿名站点默认受密码保护，**直到有人认领并绑定身份后才公开**。
- **不消耗积分**：无账号，走匿名 Drop 通道，实时 URL 免费。

### 适用场景

- 一次性分享给朋友/客户看效果。
- AI 测试环境、快速验证输出。
- 临时预览、演示。

### 注意事项

- ⚠️ **仅 1 小时有效**，不要把它当长期托管用。
- ⚠️ **含敏感信息的内容不要用匿名部署**（临时且默认不公开，但仍不建议）。
- 不适合长期保留、不适合正式发布。

---

## 模式二：Draft 草稿部署（CLI 手动）

### 是什么

**纯 CLI 手动产物**：不带 `--prod` 的部署，生成一个**随机独立临时 URL**，不占用主站 URL、不影响生产站点。

### 步骤命令

```bash
netlify deploy --dir=输出目录 --no-build --json
```

> 关键：**不带 `--prod`**，就是 Draft 草稿部署。

### 特点

- **独立随机 URL**：每次生成一个全新随机 URL（JSON 里的 `deploy_url` 字段）。
- **不影响生产**：不占用主站 URL、不碰生产站点。
- **可固定地址**：加 `--alias=xxx` 可固定子域名（最多 **37 字符**），反复部署更新同一地址：

```bash
netlify deploy --dir=输出目录 --no-build --alias=my-staging --json
```

- **不消耗积分**：Draft 免费。

### 适用场景

- 本地快速验证、看效果、迭代。
- 用 `--alias` 固定地址时，可反复更新同一预览地址，方便持续迭代。
- 想先看效果、不想污染生产站点的场景。

### 注意事项

- **`--dir` 必须指向成品输出目录**（`dist` / `build` / `out`），**不是项目根目录**。
- **`--no-build` 跳过构建**——如果你已经构建好，就加上它，避免 Netlify 再跑一次构建。

---

## 模式三：Deploy Preview 部署预览（Git 集成自动）

### 是什么

**Git 工作流（GitHub / GitLab / Bitbucket）的自动产物**：每当你打开 Pull Request（PR）或 Merge Request（MR）时自动触发，为每个 PR 生成独立预览 URL，供团队评审。

### 触发方式

- 打开 PR / MR 时**自动触发**，无需手动命令。
- 需要先 `netlify init` 或完成 Git 集成连接设置。

### 特点

- **独立预览 URL**：每个 PR/MR 有独立 URL，形如 `<pr号>--<site>.netlify.app`。
- **与 PR 绑定**：预览 URL 与对应 PR 关联，PR 关闭后预览随之失效（随分支机构生命周期）。
- **团队评审友好**：让评审者在真实 URL 上看改动效果。
- **免费、无限**：官方文档明确 **Deploy Previews or branch deploys = 0 credits**（免费、无限量）。

### 适用场景

- 团队协作、Code Review 场景。
- 想给每个 PR 一个"真实可点"的预览地址。
- 需要 Git 工作流驱动、自动化的场景。

### 注意事项

- 必须**配合 Git 连接**使用，需先 `netlify init` 或完成 Git 集成设置。
- 这是"自动化产物"，不是通过一条 CLI 命令直接生成的——它依赖 Git 远端集成。

---

## 三、三模式对比表

| 维度 | 匿名部署 | Draft 草稿 | Deploy Preview | 生产部署（参照） |
|------|----------|------------|----------------|------------------|
| **触发方式** | CLI 手动 | CLI 手动 | Git 集成自动（PR/MR） | CLI 手动 |
| **是否需要账号** | ❌ 否 | ✅ 是 | ✅ 是 | ✅ 是 |
| **场景** | 一次性分享 / AI 测试 | 本地快速验证 / 迭代 | 团队评审 / Code Review | 正式发布 |
| **URL** | 随机临时 URL | 随机独立 URL（可 `--alias` 固定） | `<pr号>--<site>.netlify.app` | 稳定生产 URL（`site.netlify.app` 或自定义域名） |
| **是否绑定 Git** | ❌ 否 | ❌ 否 | ✅ 是（与 PR 绑定） | 视情况 |
| **影响生产站点** | ❌ 否 | ❌ 否 | ❌ 否 | ✅ 是（发布到主站） |
| **消耗积分** | ❌ 0 | ❌ 0 | ❌ 0（免费、无限） | ✅ 每次 15 credits |
| **生命周期** | 1 小时（可认领，超时删除） | 独立临时，可反复更新 | 随 PR 生命周期 | 长期保留 |
| **适合** | 一次性、临时、匿名 | 本地迭代、看效果 | 团队协作、自动评审 | 正式对外发布 |

---

## 四、生产部署与积分说明

### 4.1 生产部署命令

```bash
netlify deploy --dir=输出目录 --no-build --prod --json
```

把内容发布到**稳定生产 URL**（`site.netlify.app` 或自定义域名）。

### 4.2 积分规则（重点）

- **只有生产部署才扣积分**：每次 **15 credits**。
- **Free 计划每月 300 credits** ≈ 约 **20 次生产部署**。
- **失败部署、回滚不扣积分**。
- 匿名、Draft、Deploy Preview 都**不消耗积分**。

### 4.3 省钱建议

> 迭代期用 **Draft（免费）** 反复看效果，确认就绪后再用 `--prod` 正式推送。这样既能频繁迭代，又不浪费宝贵的生产积分。

---

## 五、AI 自动化完整流程示例

下面三种套路，覆盖从"一次性匿名"到"有状态持久化"到"无状态容器"的典型场景。

### 5.1 一次性匿名部署（零账号、有手就行）

```bash
netlify deploy --dir=dist --no-build --allow-anonymous --json
```

一条命令拿到临时 URL，适合 AI 快速验证/分享。

### 5.2 创建站点并部署（有状态，初期一次建站）

```bash
# 第一次：创建站点（一次即可）
netlify sites:create --name my-project --account-slug 你的团队slug

# 关联当前目录到该站点
netlify link --name my-project

# 之后每次：生产部署
netlify deploy --dir=dist --no-build --prod --json | jq -r '.url'
```

用 `jq -r '.url'` 从 JSON 里精准取出生产 URL。

### 5.3 无状态环境（每次全新容器，不依赖 link）

如果每次部署都在**全新容器**里跑（AI 自动化 agent 常如此），**不要依赖 `netlify link`**（它是本地状态，新容器里不存在）：

```bash
# 第一次：创建站点，用 --json 拿到并保存 site_id
netlify sites:create --json   # 解析出 site_id，存进环境变量/配置文件

# 之后每次部署：显式传 --site <site-id>
netlify deploy --dir=dist --no-build --prod --json --site=<site-id>
```

这样每次部署都明确指定站点，不依赖任何本地 `.netlify` 状态文件，天然适合无状态 CI / AI 容器。

---

## 六、常见问题与坑

| # | 坑 | 原因 | 解决 |
|---|----|------|------|
| 1 | **`--dir` 指向了项目根目录** | 部署的应是成品目录 | 指向 `dist` / `build` / `out` 等输出目录 |
| 2 | **忘记 `--no-build`** | 会尝试跑构建，慢且可能失败 | 已构建好的产物部署时加上 `--no-build` |
| 3 | **首次 deploy 交互式提问卡住** | 无关联站点时 CLI 会交互询问 | 用 `sites:create` 先建好站点，或加 `--site` |
| 4 | **解析 stdout 横幅不可靠** | 横幅格式会变 | 一律用 `--json`，读 `deploy_url` / `url` / `site_id` 字段 |
| 5 | **缺 `NETLIFY_AUTH_TOKEN` 时卡住** | 无 token 会回退到交互登录 | 先确认环境变量已设置，或加 `--auth` 显式传 token |
| 6 | **匿名站点打不开** | 匿名站点默认密码保护，未认领不公开 | 用 `netlify claim --site <site-id> --token <drop-token>` 认领 |
| 7 | **匿名 URL 失效** | 1 小时有效，超时未认领被删除 | 匿名部署仅用于临时场景，长期用 Draft/生产 |

---

## 附：快速决策树

```
你要部署一个 HTML 页面到 Netlify，问自己：

1. 需要账号吗？不想注册 → 匿名部署（--allow-anonymous）
2. 只是本地看效果、迭代？ → Draft（不带 --prod，免费）
3. 团队要评审 PR？ → 靠 Git 集成的 Deploy Preview（免费）
4. 要正式对外发布？ → 生产部署（--prod，每次扣 15 credits）
```

**核心口诀**：临时免费随便用（匿名/Draft/Preview），正式发布才扣分（--prod）。