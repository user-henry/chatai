**English** | [简体中文](README.md)

# HZYAI · Intelligent Assistant (repository name `chatai`)

A **purely static AI workbench**: chat, vision understanding, translation, speech-to-text, image generation, video generation, scheduled tasks, virtual characters, and a multi-step "AI short film" pipeline. The whole application is a single-page site that can be dropped onto any static host — **no build step, no npm dependencies**; the account system and cloud sync reuse NFLSHC Chat's unified backend.

- Live site: **https://chatai.bot.cd**
- Account and cloud-sync backend: `https://worker.nflshcchat.cc.cd` (NFLSHC Chat unified authentication, supporting username/password and Google sign-in)
- AI gateway: `https://hzyai-worker.nflshcchat.cc.cd` (Cloudflare Worker; models and keys all live server-side)

---

## Directory structure

| File / directory | Purpose |
|---|---|
| `index.html` | Main application (~450 KB single file): login page (slider CAPTCHA + Google sign-in), sidebar, 11 workspaces, model marketplace, usage statistics, account appeal |
| `share.html` | HZYAI share center: 📤 share conversations (publish your own conversations publicly), browse public conversations, public character sharing with one-click import; includes its own login and Google callback handling (`#google_login=…`) |
| `register.html` | Registration page: slider CAPTCHA → SHA-256 password hash → submit; dedicated messages for `409` (username taken) and `403` (IP restricted) |
| `css/themes.css` | Theme variables (`--bg-primary`, `--text-primary`, `--accent-color`, `--hzy-shadow-*`, etc.) and the values for the 5 themes |
| `js/theme.js` | `ThemeManager` (theme read/write + `themechange` event), `AuthToken` (token storage), global `fetch` patch (auto-adds `Authorization`, redirects to login on 401) |
| `js/marked.min.js` | Markdown rendering (conversation content, code blocks) |
| `js/sanitize.js` | **HTML allowlist**: `window.sanitizeHtml(html)`, the unified filter before conversation/share content is rendered (see "Content security" below) |
| `CNAME` | GitHub Pages custom domain: `chatai.bot.cd` |
| `HZYAI LOGO.png` | Site icon / brand image |

> Local backup files such as `index.html.bak` **must not be committed**; keep only the files above in the repository.

---

## Features

Sidebar (workspaces):

| Entry | Description |
|---|---|
| 💬 Chat | Multi-conversation management, Markdown rendering, code generation and highlighting, new conversation, export Markdown |
| 👁️ Vision | Vision understanding mode: image Q&A (GLM-4.6V / GLM-4V series) |
| 🌐 Translate | Intelligent translation |
| 🎙️ Speech to text | Audio transcription (`/stt`) |
| 🎨 Image | Text-to-image (`/image`, also uses pollinations as a fallback) |
| 🎬 Video | Video generation mode (`/video`) |
| ⏰ Scheduled tasks | Scheduled / recurring task management |
| 🎭 Characters | Virtual character customization (character cards, personas) that can be bound to conversations |
| 🎬 AI short film | Four-step pipeline: ① generate script → ② plan storyboard → ③ generate reference images and video → ④ compose the final short film |
| 🎬 Long video | Long-duration video generation |
| 📊 Statistics | Usage statistics: call counts, model usage distribution, etc. |

Also: **model marketplace** (model list and descriptions kept in sync with the cloud list, with a live view of `hzyai-worker/models`), **semantic embeddings / vectors** (`/embedding`), **conversation sharing** (`share.html`), and **account appeal**.

---

## Architecture

```
Browser (this repository, GitHub Pages / chatai.bot.cd)
   │
   ├── Account / cloud sync ──▶ worker.nflshcchat.cc.cd
   │        /api/auth/login · /api/auth/me · /api/auth/appeal
   │        /api/auth/google/config · /api/auth/google/exchange
   │        /api/legacy/issues        ← “zero-server” cloud storage for conversations (GitHub Issues proxy)
   │
   └── Model calls ──────▶ hzyai-worker.nflshcchat.cc.cd (self-hosted AI gateway)
            /chat      chat completion
            /vision    vision understanding
            /image     text-to-image
            /video     video generation
            /stt       speech-to-text
            /translate translation
            /embedding semantic embeddings
            /moderate  content moderation
            /models    live model list
            /sizhi/chat
            /bigmodel/api/paas/v4/*   ← Zhipu BigModel passthrough (chat/completions, images/generations, videos/generations)
```

Available models come from two sources, all configured and forwarded by the gateway:

- **Cloudflare Workers AI**: GLM-4.7-Flash, GLM-4-Flash-250414, GLM-4.6V-Flash, GLM-4.1V-Thinking-Flash, GLM-4V-Flash, Qwen3 / Qwen2.5-Coder-32B / qwen3-30b-a3b-fp8, DeepSeek-R1-Distill-Qwen-32B, GPT-OSS-20B / 120B, Qwen3-Embedding-0.6B, etc.;
- **Zhipu BigModel**: passed through via `/bigmodel/api/paas/v4/*`;
- **pollinations**: fallback channel for image generation.

The frontend **contains no API keys**; all model keys live in the Worker secrets of `hzyai-worker`. Before committing code, make sure no keys, passwords or private addresses have been written into HTML/JS.

---

## Data and privacy

- Conversations, characters, scheduled tasks, etc. are stored **locally in the browser** by default; account information comes from the NFLSHC Chat session.
- To use multiple devices, use "Sync to cloud / Pull from cloud" on the page; data is stored through the Issues storage API of `worker.nflshcchat.cc.cd`. You can also simply export Markdown and keep it yourself.
- Sharing: in `share.html` you can publish selected conversations or characters publicly; other users can view or import them from "Browse public conversations / Browse characters". Conversations that have not been published are never public.

---

## Themes and authentication (`js/theme.js`)

- Themes: `dark` (midnight black-gold, default), `light` (dawn white), `blue` (deep sea blue), `purple` (violet night), `green` (emerald green).
  - Persistence key: `localStorage['nflshc_theme']`; implemented by setting `data-theme="<theme>"` on `<html>` and overriding variables from `css/themes.css`; switching dispatches a `themechange` event.
  - From code: `window.ThemeManager.setTheme('blue')` / `getThemeInfo()`.
- Authentication: the token is stored in `nflshc_auth_token` (localStorage or sessionStorage); `theme.js` patches `window.fetch` to automatically add `Authorization: Bearer <token>` to requests going to `worker.nflshcchat.cc.cd`, and on a 401 clears the session and redirects to `index.html?next=<current-page>`.

---

## Content security: HTML allowlist for Markdown rendering (`js/sanitize.js`)

Conversation bodies and "shared conversations" are rendered with `marked`, and **marked keeps inline HTML by default**: if model output contains
`<img src=x onerror=…>`, or someone shares a conversation containing `<script>`, it would execute in other people's browsers. So every rendering exit passes through an allowlist:

- `renderMarkdown()` in `index.html` and `renderMarkdownSimple()` in `share.html` both end with a call to
  `window.sanitizeHtml(html)`; if the allowlist script is not loaded they **fall back to plain text** (`escapeHtml(content)`) and never inject as-is.
- Allowlist rules (the same implementation as the NFLSHC Chat main site and Windows desktop client):
  - Kept: paragraphs/headings/lists/tables/code/quotes, plus `img`, `video`, `audio`, `source`, `track`, `figure`, `details`, `time`;
  - Removed entirely: `script`, `style`, `iframe`, `object`/`embed`, `form` and controls, `link`/`meta`/`base`, `svg`, `math`, etc.;
  - Attributes are allowed per tag; `on*` events can never get through; `id`/`name` are always stripped (to prevent DOM clobbering);
  - URLs allow only http(s), same-site relative paths, `mailto:`/`tel:` (links only) and bitmap `data:image/…;base64,`;
    `javascript:`, `data:text/html` and the like are dropped (whitespace/tabs are stripped from the protocol first, to block `java\tscript:` bypasses);
  - `style` keeps only `max-width`/`width`/`height`/`margin`/`border-radius`/`object-fit`; `position`/`inset`/`z-index` are forbidden.
- Result: inline media such as images, `<video controls>` and `<audio controls>` still work (direct links from file hosts are fine),
  while script payloads are dismantled. Regression script: `node dev/tests/test-chatai-sanitize.mjs` (under `dev/tests/` in the NFLSHC main repository,
  running against the real live environment) — 25/25 passing, covering 8 attack payloads + both rendering functions + "no payload actually executed".

---

## Running locally

```bash
# Start a static server from the repository root (use http://, do not open via file://)
python -m http.server 8080
# Open http://127.0.0.1:8080/
```

The local page still calls the **live Workers** (account and model APIs), so debugging login, chat and other features locally requires an NFLSHC Chat account and working access to the two domains above; you can see the login page without signing in when only changing styles/layout.

---

## Deployment

1. Push to `main`;
2. GitHub Pages publishes automatically; the custom domain is determined by `CNAME` (`chatai.bot.cd`);
3. After publishing, a hard refresh (`?v=timestamp`) is recommended to avoid browsers/CDNs serving the old page. `js/theme.js` is referenced as `?v=4` in `index.html`; bump that version whenever you change the file so users receive the new script.

---

## Related repositories

| Repository | Description |
|---|---|
| [nflshcchat](https://github.com/user-henry/nflshcchat) | NFLSHC Chat main site and unified backend (Worker + D1) |
| [nflshcchat-home](https://github.com/user-henry/nflshcchat-home) | NFLSHC Chat official release page (https://home.nflshcchat.cc.cd) |
| [nflshcchat-platform](https://github.com/user-henry/nflshcchat-platform) | Developer platform |
| [nflshcchat-accounts](https://github.com/user-henry/nflshcchat-accounts) | Account authorization center (OAuth) |
| [help-nflshc202401](https://github.com/user-henry/help-nflshc202401) | Help documentation |

---

## License

MIT License. Model relaying and account services are maintained by the NFLSHC Chat project; use of third-party models is subject to their respective providers' terms.
