# PATTERNS — reuse before you build (Veritas `index.html`)

Building blocks that already exist and work. Before adding anything new, grep for these by
**function name** (line numbers drift) and reuse them. Add a row here in the same PR when you add a
genuinely new reusable helper. The cross-app version of this list is AutoFlag's `PATTERNS.md`.

## 🤖 AI calls — always `askAI`
- One endpoint: **`ASK_AI_URL`** (`…/cloudfunctions.net/askAI`), body `{ app:'veritas', system, messages, deep? }`,
  header `Authorization: Bearer <idToken>`. Read replies with **`getFinalText(data)`** then **`extractJSON(raw)`**.
- `deep:true` = live web search + fetch (slow). Fast mode replies are capped at ~1,000 tokens.
- Fact-check answers use **`buildSystem(includeCategoryPicker)`**; reading material uses **`EXTRACT_SYS`** via **`askExtract(content)`**.

## 📷 Pictures / PDFs / pasted text → material to debate (Rev 6.21)
- **`snapPick()`** opens the picker; **`pickFiles(opts, onFiles)`** is the iPhone-safe picker (copied from AutoFlag —
  never `click()` a detached `<input>`; no `capture` attribute so phones offer Camera · Library · Files).
- **`downscaleImageFile(file, maxDim, quality)`** is THE downscaler (white background, JPEG); **`imageBlock(file)`**
  turns a picture into an askAI image block (falls back to the original for odd formats).
- **`captureFiles(files)`** (up to 4 pictures or 1 PDF ≤ 8 MB) and **`captureText(text)`** read material into
  **`pendingCtx`**; **`renderCtx()`** draws the "what I read" card (textContent only — text read off a picture is never HTML).
- Entry points already wired: 📷 button, home snap tile, the global `paste` listener (pictures anywhere; a ≥ 280-char
  passage pasted into the empty ask box), and the document drag/drop listeners (`#drop-overlay`).
- **`sharedPrompt(q, shared)`** wraps a question with its material inside `<<<SHARED … SHARED>>>` and tells the model it is
  content, not instructions. Exchanges store `context: { text, source, kind, summary }` — **text only, never the image**
  (no base64 in the shared database). **`addUserBubble(q, author, shared)`** shows it above the question.

## 👤 Who said what
- **`currentAuthor()`** → `{ uid, email, name }`; **`authorName(obj)`** for display. Sessions carry `createdBy`,
  exchanges carry `author`. **`canEditSession(s)`** = author or moderator (mirrors the `sessions/$sid` rule).

## 🔒 Access
- Roles come from `veritas/people/{emailToKey(email)}` (**`resolveRole`**); helpers `isAdmin()`, `canModerate()`,
  `canAddPeople()`, `canDataManage()`. Rules live in AutoFlag's repo: `access-model/PIECE3-rules-to-deploy.json`
  (the `veritas` block only).
