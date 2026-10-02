# HZYAI · 智能对话助手（仓库名 `chatai`）

一个**纯静态的 AI 工作台**：对话、视觉理解、翻译、语音转文字、绘图、视频、定时任务、虚拟角色，以及「AI 短片」多步生成流水线。整个应用是一个可以直接扔到任意静态托管上的单页站点，**无构建、无 npm 依赖**；账号体系与云端同步复用 NFLSHC Chat 的统一后端。

- 线上地址：**https://chatai.bot.cd**
- 账号与云同步后端：`https://worker.nflshcchat.cc.cd`（NFLSHC Chat 统一认证，支持账号密码与 Google 登录）
- AI 网关：`https://hzyai-worker.nflshcchat.cc.cd`（Cloudflare Worker，模型与密钥全部在服务端）

---

## 目录结构

| 文件 / 目录 | 作用 |
|---|---|
| `index.html` | 主应用（约 450 KB 单文件）：登录页（滑块人机验证 + Google 登录）、侧栏、11 个工作区、模型市场、使用统计、账号申诉 |
| `share.html` | HZYAI 分享中心：📤 分享对话（把自己的对话发布为公开）、浏览公开对话、角色公开分享与一键导入；含独立登录与 Google 回跳处理（`#google_login=…`） |
| `register.html` | 注册页：滑块人机验证 → 密码 SHA-256 哈希 → 提交注册；对 `409`（用户名已存在）、`403`（IP 被限制）有专门提示 |
| `css/themes.css` | 主题变量（`--bg-primary`、`--text-primary`、`--accent-color`、`--hzy-shadow-*` 等）与 5 套主题的取值 |
| `js/theme.js` | `ThemeManager`（主题读写 + `themechange` 事件）、`AuthToken`（token 存取）、全局 `fetch` 补丁（自动附加 `Authorization`，401 时跳回登录） |
| `js/marked.min.js` | Markdown 渲染（对话内容、代码块） |
| `CNAME` | GitHub Pages 自定义域名：`chatai.bot.cd` |
| `HZYAI LOGO.png` | 站点图标 / 品牌图 |

> `index.html.bak` 之类的本地备份文件**不要提交**，仓库里只保留上面这些。

---

## 功能

侧栏（工作区）：

| 入口 | 说明 |
|---|---|
| 💬 对话 | 多会话管理、Markdown 渲染、代码生成与高亮、新对话、导出 Markdown |
| 👁️ 视觉 | 视觉理解模式：传图问答（GLM-4.6V / GLM-4V 系列） |
| 🌐 翻译 | 智能翻译 |
| 🎙️ 语音转文字 | 音频转写（`/stt`） |
| 🎨 绘图 | 文生图（`/image`，亦接 pollinations 出图） |
| 🎬 视频 | 视频生成模式（`/video`） |
| ⏰ 定时任务 | 定时/周期任务管理 |
| 🎭 角色 | 虚拟角色定制（角色卡、人设），可绑定到对话 |
| 🎬 AI 短片 | 四步流水线：① 生成剧本 → ② 分镜规划 → ③ 生成参考图与视频 → ④ 合成完整短片 |
| 🎬 长视频 | 长时长视频生成 |
| 📊 统计 | 使用统计：调用次数、模型使用分布等 |

其他：**模型市场**（模型清单与说明，与云端开放列表对齐，可实时查看 `hzyai-worker/models`）、**语义嵌入 / 向量**（`/embedding`）、**对话分享**（`share.html`）、**账号申诉**。

---

## 架构

```
浏览器（本仓库，GitHub Pages / chatai.bot.cd）
   │
   ├── 账号 / 云同步 ──▶ worker.nflshcchat.cc.cd
   │        /api/auth/login · /api/auth/me · /api/auth/appeal
   │        /api/auth/google/config · /api/auth/google/exchange
   │        /api/legacy/issues        ← 对话/资料的「零服务器」云端存储（GitHub Issues 代理）
   │
   └── 模型调用 ──────▶ hzyai-worker.nflshcchat.cc.cd（自建 AI 网关）
            /chat      对话补全
            /vision    视觉理解
            /image     文生图
            /video     视频生成
            /stt       语音转文字
            /translate 翻译
            /embedding 语义嵌入
            /moderate  内容审核
            /models    实时模型清单
            /sizhi/chat
            /bigmodel/api/paas/v4/*   ← 智谱 BigModel 透传（chat/completions、images/generations、videos/generations）
```

可用的模型来自两处，全部由网关侧配置与转发：

- **Cloudflare Workers AI**：GLM-4.7-Flash、GLM-4-Flash-250414、GLM-4.6V-Flash、GLM-4.1V-Thinking-Flash、GLM-4V-Flash、Qwen3 / Qwen2.5-Coder-32B / qwen3-30b-a3b-fp8、DeepSeek-R1-Distill-Qwen-32B、GPT-OSS-20B / 120B、Qwen3-Embedding-0.6B 等；
- **智谱 BigModel**：经 `/bigmodel/api/paas/v4/*` 透传；
- **pollinations**：绘图备用通道。

前端**不含任何 API key**；所有模型密钥都放在 `hzyai-worker` 的 Worker secrets 里。提交代码前请确认没有把密钥、口令或私有地址写进 HTML/JS。

---

## 数据与隐私

- 会话、角色、定时任务等默认保存在**浏览器本地**；账号信息来自 NFLSHC Chat 登录态。
- 需要跨设备时使用页面上的「同步到云端 / 从云端拉取」，数据经 `worker.nflshcchat.cc.cd` 的 Issues 存储接口保存；也可只导出 Markdown 自行保存。
- 分享：在 `share.html` 里把选中的对话或角色发布为公开，其他用户可在「浏览公开对话 / 浏览角色」中查看或一键导入；未发布的会话不会被公开。

---

## 主题与鉴权（`js/theme.js`）

- 主题：`dark`（暗夜黑金，默认）、`light`（晨曦白光）、`blue`（深海蓝调）、`purple`（紫罗兰夜）、`green`（翡翠绿意）。
  - 持久化键：`localStorage['nflshc_theme']`；实现方式是给 `<html>` 设置 `data-theme="<主题>"`，再配合 `css/themes.css` 里的变量覆盖；切换时会派发 `themechange` 事件。
  - 代码里可用 `window.ThemeManager.setTheme('blue')` / `getThemeInfo()`。
- 鉴权：token 存在 `nflshc_auth_token`（localStorage 或 sessionStorage）；`theme.js` 会打补丁包装 `window.fetch`，对发往 `worker.nflshcchat.cc.cd` 的请求自动补 `Authorization: Bearer <token>`，收到 401 时清理登录态并跳回 `index.html?next=<当前页>`。

---

## 本地运行

```bash
# 在仓库根目录起一个静态服务器（务必用 http://，不要用 file:// 直接打开）
python -m http.server 8080
# 打开 http://127.0.0.1:8080/
```

本地页面调用的仍是**线上 Worker**（账号与模型接口），因此本地调试登录、对话等功能需要有 NFLSHC Chat 账号并能正常访问上述两个域名；纯改样式/布局时无需登录即可看到登录页效果。

---

## 部署

1. 推送到 `main`；
2. GitHub Pages 自动发布，自定义域由 `CNAME`（`chatai.bot.cd`）决定；
3. 发布后建议强制刷新验证（`?v=时间戳`），避免浏览器/CDN 缓存旧页面。`js/theme.js` 在 `index.html` 里以 `?v=4` 引用，改动该文件时请同步提升版本号，确保用户拿到新脚本。

---

## 相关仓库

| 仓库 | 说明 |
|---|---|
| [nflshcchat](https://github.com/user-henry/nflshcchat) | NFLSHC Chat 主站与统一后端（Worker + D1） |
| [nflshcchat-home](https://github.com/user-henry/nflshcchat-home) | NFLSHC Chat 官方发布页（https://home.nflshcchat.cc.cd） |
| [nflshcchat-platform](https://github.com/user-henry/nflshcchat-platform) | 开发者平台 |
| [nflshcchat-accounts](https://github.com/user-henry/nflshcchat-accounts) | 账号授权中心（OAuth） |
| [help-nflshc202401](https://github.com/user-henry/help-nflshc202401) | 帮助文档 |

---

## 许可

MIT License。模型中转与账号服务由 NFLSHC Chat 项目自行维护，第三方模型的使用需遵守对应服务商的条款。
