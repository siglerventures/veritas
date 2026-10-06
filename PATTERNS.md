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

## 🃏 Answer card (Rev 6.22)
- **`buildBotCard(ex)`** returns the card element (plain answer first; evidence/sources/reasoning/disputes behind
  "Why N%?" → **`toggleDetails(btn)`**, remembered in localStorage `veritas_details_open`); **`addBotCard(ex)`** appends it.
  Re-render a card in place with `wrap.replaceWith(buildBotCard(ex))` — don't patch its innerHTML.
- **`normResult(ex)`** = the exchange's result object (parses `rawResponse` if needed); **`scoreOf(r)`** = 0–100 integer.

## 🙋 Challenges — "I disagree — here's my evidence" (Rev 6.22)
- Stored at **`veritas/challenges/{sessionId}/{exchangeId}/{challengeId}`** = `{ id, by, text, at, before:{score,verdict}, result }`
  — its OWN node, so any roster member can challenge anyone's answer without write access to their session, and a
  session save can never wipe a challenge. The original answer is never overwritten.
- **`subscribeChallenges()`** keeps **`challengeIndex`** (exchangeId → list, oldest first) live; **`renderChallenges(exId)`**
  draws the "⚖ Now N%" line + each challenge (who, their words, score change); **`latestResult(ex)`** = latest re-check
  or the original. Acknowledging after a challenge uses the latest re-check: **`acknowledgeResult(exId, chId)`**.
- **`openDebate` / `submitDebate`** (argument sent inside `<<<CHALLENGE … CHALLENGE>>>`, earlier challenges summarized),
  **`debateAddPicture` / `debateReadFiles`** (a picture as evidence → its text into the box; Ctrl+V in the box does the same),
  **`leanResult(r)`** keeps only the known answer fields before saving.
- Rules: the `veritas.challenges` block in PIECE3 — members read; a member writes only their OWN challenge (`by/uid`);
  moderators/admins can remove any. Session owners can NOT delete challenges against them.
- A follow-up typed into someone else's debate starts the asker's own session (theirs kept as AI context) — see `ask()`.

## 🔗 Links — every screen has its own URL (Rev 6.23)
- `#d/<sessionId>[/<exchangeId>]` · `#c/<categoryId>` · `#fact/<factId>` (sloppy forms accepted: `#/d/…`, `#debate/…`).
- **`setRoute(hash, replace?)`** pushes a history entry (◀ Back works) — call it from any new screen; it's a no-op while
  **`route()`** is applying a URL (popstate / hashchange / shared link on load), so screens never loop.
- **`loadSession(sid, exId?)`** fetches a debate started after page load, scrolls to + flashes `exId` (**`focusExchange`**),
  and handles a deleted debate. **`shareDebate(exId?)`** = phone share sheet, else copy link. 🔗 Share sits in the debate header.

## 🗂 History filters
- **`HIST_FILTERS`** (All · Mine · ⚖ Challenged · ✓ Acknowledged), **`setHistFilter(f)`** (remembered in localStorage),
  **`histFilterMatch(f, sid, s)`**. Lists sort by **`activityAt(sid, s)`** — a new challenge bumps a debate up.
  **`sessionChallenges[sid]`** = `{ count, lastAt }`, **`challengeSid[exId]`** = its session (built in `rebuildChallengeIndex`).

## 🔔 Notifications (in-app)
- Seen state at **`veritas/seen/{uid}`** = `{ all, ex: { exchangeId: time } }`, mirrored in localStorage (works before the
  rules paste). **`loadSeen` / `saveSeen` / `seenTsFor(exId)`**; **`markSessionSeen(sid)`** — viewing a debate reads it.
- **`challengesOnMyAnswers()`** = challenges by others on answers you wrote (**`answerOwner(s, ex)`**: the exchange's author,
  else the session's starter); **`updateNotifBadge()`** drives the 🔔 count and the tab title; **`toggleNotifs`** opens the list.
  NEW tags on challenges come from **`newSince`**. A new notification type → add it to `challengesOnMyAnswers` (or a sibling)
  and to the list renderer; keep "seen" per exchange.

## ⚖ Debate view — scored points (Rev 6.25)
- A debate = the session's opening exchange (`topicExchange(s)`) + every point made after it. **`renderDebate(sid, { focus, keepScroll })`**
  draws it: the pinned **`#topic-bar`** (`renderTopicBar` — picture, shared text, question, the claim, For/Against
  scoreboard, 🏅 leaderboard), then numbered turns (`buildExTurn` / `buildPointTurn`), with the point box moved to the
  bottom (`body.in-debate` reorders `#main`). **`enterDebateView(on)`** switches layout + button labels.
- **`debateTurns(sid)`** = exchanges + points, oldest first, numbered; **`debateScore(turns)`** = side totals + per-person points;
  **`topicClaim(s, turns)`** = the opening answer's `claim` (buildSystem asks for it), else the first point's.
- **`submitPoint(text, deep, shared, files)`** posts a point: shows it at once ("Scoring…"), asks askAI with **`pointSystem()`** +
  **`pointPrompt(…)`** (claim, material, opening, last 12 turns, the new point in `<<<POINT … POINT>>>`), keeps
  `stance` (for/against/neutral), `strength` 0–10, `points_reason`, `reply`, `claim` (**`leanPoint`**), saves it at
  `veritas/challenges/{sid}/{openingExchangeId}/{id}` (any member, own record — the session stays the starter's).
  Reply to a turn: **`setReplyTarget({ turn, n, side })`** (👍 I agree / 🙋 I disagree). Every message typed inside a debate goes
  through `submitPoint` (see `ask()`); a new debate is created by `ask()` outside one.
- Old challenges (no `stance`) show as "🙋 Challenge" turns; old follow-up exchanges show as "question" turns.

## 📷 Pictures kept with a debate — Firebase Storage
- **`uploadDebateImages(sid, files)`** → `[{ url, path }]` under **`veritas/debates/<topic_slug>_<last 4 of id>/<date>_<name>.jpg`**
  (downscaled JPEG, readable folder per the standing rule; `stSlug`, `debateFolder`). **`saveTopicImages(sid, files)`** puts them on
  `exchanges/0/context/images`; points keep theirs in `images`. **`topicAddPicture(sid)`** = 📷 Add the picture on an older debate.
  Only the link is in the database — never image data.
- **Storage rule (Firebase console → Storage → Rules — added by Phil, Rev 6.25).** Inside `match /b/{bucket}/o { … }`:
  ```
  // Veritas: debate pictures (Rev 6.25). Signed-in users read; create-only images ≤ 5 MB.
  match /veritas/debates/{allPaths=**} {
    allow read: if request.auth != null;
    allow create: if request.auth != null
                  && request.resource.size < 5 * 1024 * 1024
                  && request.resource.contentType.matches('image/.*');
  }
  ```
  Storage rules can't read the Realtime Database roster, so this is "any signed-in account" (same posture as AutoFlag photos);
  create-only means nobody can overwrite or delete a picture. Without the rule, saving a picture fails quietly and the debate keeps its text.

## 📱 Phone home
- **`recentDebatesHtml()`** — the latest 5 debates on the home screen (phones only; wide screens use the sidebar History).

## 👤 Who said what
- **`currentAuthor()`** → `{ uid, email, name }`; **`authorName(obj)`** for display. Sessions carry `createdBy`,
  exchanges carry `author`. **`canEditSession(s)`** = author or moderator (mirrors the `sessions/$sid` rule).

## 🔒 Access
- Roles come from `veritas/people/{emailToKey(email)}` (**`resolveRole`**); helpers `isAdmin()`, `canModerate()`,
  `canAddPeople()`, `canDataManage()`. Rules live in AutoFlag's repo: `access-model/PIECE3-rules-to-deploy.json`
  (the `veritas` block only).
