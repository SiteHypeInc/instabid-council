# InstaBid AI Council — UI

A single-file web app that calls the n8n "Council" webhook (4 frontier models answer in parallel, Opus synthesizes) and renders the result — consensus, dissent flags, and raw per-model answers — with thread history, file attachments, and copy/download.

No build step. No framework. One HTML file.

---

## Files

- `index.html` — the entire app (HTML + CSS + vanilla JS inline). This is the codebase.
- `vercel.json` — Vercel static deploy config.
- `Procfile` + `package.json` — Railway/Heroku-style static serve.

## Dependencies (CDN, no install)

- [`marked`](https://cdn.jsdelivr.net/npm/marked@12) — markdown → HTML
- Google Fonts (Inter, JetBrains Mono, Newsreader)

That's it. Everything else is vanilla JS.

---

## Run it locally

```bash
# simplest
open index.html

# or serve (avoids file:// quirks)
python3 -m http.server 8080
# → http://localhost:8080/
```

---

## Configuration

One value to set, top of the `<script>` block in `index.html`:

```js
const WEBHOOK_URL = 'https://n8n.instabid.pro/webhook/<id>/chat';
```

The webhook expects:
```json
{ "chatInput": "<prompt + any attached file text>", "sessionId": "<stable id>" }
```
and streams NDJSON lines of shape `{ "type": "item", "content": "..." }`, terminated by `{ "type": "end" }`. The UI also falls back to a non-streaming full-text read if streaming is blocked.

Response parsing relies on markdown headings:
- `## Consensus answer`
- `## Dissent flags`
- `## Raw answers` → `### Claude...` / `### GPT...` / `### Gemini...` / `### Grok...`

If you change the synthesis prompt in n8n, keep those headings or update `parseResponse()`.

---

## Deploy

### Option A: Host inside n8n itself (no extra service — recommended)

The UI is just static HTML. n8n can serve it directly from the same Railway service that already runs `n8n.instabid.pro` — no second deploy, no DNS change, no CORS config (same origin as the chat webhook).

1. In n8n: **Workflows → Import from File** → pick `n8n-council-ui.workflow.json` from this repo.
2. Open the imported workflow, click **Active** (top right toggle).
3. Visit `https://n8n.instabid.pro/webhook/council` — that's the UI, serving the chat through your existing webhook.

The workflow is two nodes: a GET webhook at `/webhook/council` and a Respond-to-Webhook node that returns `index.html` with `Content-Type: text/html`. To update the UI, edit `index.html` here, re-run the build below to regenerate `n8n-council-ui.workflow.json`, and re-import (n8n's import overwrites on same name).

Regenerate after editing `index.html`:
```bash
python3 -c "
import json
html = open('index.html').read()
wf = json.load(open('n8n-council-ui.workflow.json'))
wf['nodes'][1]['parameters']['responseBody'] = html
json.dump(wf, open('n8n-council-ui.workflow.json','w'), indent=2)
"
```

### Option B: Vercel

```bash
npm i -g vercel
vercel --prod
```

Then in Vercel dashboard:
- Domains → add `council.instabid.pro`
- DNS: CNAME `council` → `cname.vercel-dns.com`

### Option C: Railway (separate service)

```bash
railway up
```

Or connect this GitHub repo in Railway dashboard. `Procfile` + `package.json` serve `index.html` via `serve`.

Then:
- Settings → Networking → add custom domain `council.instabid.pro`
- DNS: CNAME `council` → Railway-provided CNAME target

### n8n CORS (required for either)

Add `https://council.instabid.pro` to the webhook's allowed origins. The webhook already returns CORS headers for `hyperagent.com`; just add the new origin.

Preflight (`OPTIONS`) must return:
- `Access-Control-Allow-Origin`
- `Access-Control-Allow-Methods: OPTIONS, GET, POST`
- `Access-Control-Allow-Headers: content-type`

---

## Code map (`index.html`)

| Section (in `<script>`) | What it does |
|---|---|
| `WEBHOOK_URL` | n8n endpoint (the one config value) |
| `safeStorage` | localStorage wrapper w/ in-memory fallback (sandbox-safe) |
| `safeConfirm` | confirm() wrapper, falls through if blocked |
| `attachments` block | file attach: read-as-text, chips, fold into prompt as labeled context |
| `composePrompt()` | merges question + attached file text into `chatInput` |
| `askCouncil()` | fetch → try streaming reader → fall back to full-text read |
| `parseResponse()` | splits the markdown into consensus / dissent / per-model |
| `renderAnswer()` | renders the three result zones via marked |
| `renderThreadList()` | sidebar thread history |
| `buildMarkdown()` | export format for copy/download |
| `showDownloadModal()` | guaranteed copy/save fallback when sandbox blocks downloads |

---

## Post-deploy simplification (optional)

Once it's on a real origin (not a sandbox iframe), these shims become unnecessary:
- `safeStorage` → plain `localStorage`
- `safeConfirm` → plain `confirm`
- `showDownloadModal()` → native blob download works directly

Streaming reliability also improves at a real origin.

---

## Roadmap candidates (not built)

- Cloud-backed thread history (cross-device) — currently localStorage, per-browser
- PDF/DOCX attachment parsing — currently text-based files only
- Per-model "ask just one model" mode (skip the council)
- Auth gate if exposed publicly

---

*Built in-thread with Atlas. Single-file by design — keep it that way unless it earns the complexity.*
