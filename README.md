# Game of Thrones Trivia — Kaltura Conversational Avatar

A Game of Thrones trivia challenge built on the Kaltura Conversational Avatar platform. The avatar, **Ashara Dayne** (a book character who never appeared in the HBO show), guides users through five questions of escalating difficulty, provides in-character feedback, and delivers a results summary at the end.

---

## What This Does

- Embeds a Kaltura/eself Conversational Avatar on a GOT-themed webpage
- Collects the user's name before starting the session
- Injects user context (name, time of day, page content) into the avatar at session start via Dynamic Page Prompt (DPP)
- Avatar runs a 5-question adaptive trivia game with image clues on the first three questions
- On session end: POSTs a summary to a webhook and redirects to a score card page

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | Main page — GOT lore, name input, avatar embed, DPP injection, session tracking |
| `results.html` | Score card — reads session data from localStorage and displays results |

---

## How to Run

```bash
python3 -m http.server 8080
```

Open `http://localhost:8080` in Chrome. Must be served over HTTP — the avatar won't load from a `file://` URL due to browser security restrictions.

---

## The Assignment

1. Create a webpage with information on it
2. Embed a Kaltura Conversational Avatar
3. When a user starts the conversation, inject data about the user, the current time, and the page content
4. Build an avatar that greets the user based on time of day
5. Run adaptive trivia — difficulty increases on correct answers, decreases on wrong
6. Avatar shows images with visual questions
7. End-of-session schema summary with questions, answers, and scores
8. POST the summary to a webhook

---

## The DPP Injection Problem (and Everything We Tried)

### What DPP Is

The avatar accepts a **Dynamic Page Prompt** (DPP) — a message injected from the parent page into the avatar iframe — that provides context the avatar can use in the conversation. The mechanism is a `postMessage` with type `eself-dynamic-prompt-message`.

The goal: inject the user's name so the avatar opens with *"Good morning, Corey — I am Ashara Dayne..."* instead of a generic greeting.

### What We Discovered About This Avatar

This avatar is hosted at `meet.avatar.us.kaltura.ai` and runs on the **eself.ai** platform under the hood. This matters because:

- The **Kaltura Iframe SDK** (`KalturaAvatarSDK`) does **not** work with this avatar — it logs `"Message type not allowed: undefined"` and gets rejected
- The correct approach is a **raw iframe embed** with a direct `postMessage` using `eself-dynamic-prompt-message`
- The avatar has a **Dynamic Opening Phrase** feature (enabled in eself Studio) that generates a personalized AI greeting using the DPP as context

### The Attempts

**Attempt 1: Plain text DPP**
```
"User name: Corey. Time of day: morning..."
```
Result: Avatar said **"Good morning, User name"** — it read the label literally.

**Attempt 2: Plain text with natural language**
```
"The visitor's name is Corey. It is morning..."
```
Result: Promising direction but we moved to JSON before fully testing this.

**Attempt 3: JSON with `user.first_name` (official SDK format)**
```json
{"v":"2","user":{"first_name":"Corey"},"inst":["..."]}
```
Result: Avatar said **"Good morning, user name"** — it read the JSON key names `user` and `first_name` as literal words.

**Attempt 4: JSON with explicit name in `inst` array**
```json
{"v":"2","user":{"first_name":"Corey"},"inst":["The person you are speaking with is named Corey. Call them Corey.",...]}
```
Result: Avatar said **"Good morning, users first name"** — still parsing JSON key names.

**Attempt 5: JSON without `user` key (name only in `inst`)**
```json
{"v":"2","inst":["Greet Corey by name. Their name is Corey.",...]}
```
Result: Avatar said **"Good morning, name"** — parsed "name" from the sentence.

**Attempt 6: URL parameters on the iframe src**
```
?first_name=Corey&name=Corey&time_of_day=morning
```
Result: `formLeadInfo` changed from `{}` to `Object` (data got in), but still no name in greeting.

### Where Things Stand

The DPP injection **works** — the avatar receives the data and knows the user's name when asked mid-conversation. The problem is specific to the **Dynamic Opening Phrase** feature: its generation LLM reads the DPP as "user memory" and outputs JSON key names as literal words instead of extracting values.

**The confirmed fix** (not yet applied) is a change to the **Opening Phrase Generation Prompt** in eself Studio:

> "The user memory contains a JSON object. Extract the value of `user.first_name` and use it as the person's name in the greeting. Never say 'user', 'first name', or any JSON key as words — only use the actual value."

This is a Studio-side configuration change, not a code change. Everything else — the DPP injection, the knowledge base, the trivia flow, the webhook, and the results page — is functional.

---

## Technical Notes

### Why Not the Kaltura Iframe SDK?

The Kaltura Conversational Avatar SDK (`kaltura-avatar-sdk.min.js`) uses a different message protocol than the eself-origin avatar. When loaded against this avatar, every SDK message is rejected with `"Message type not allowed"`. The correct integration is:

```javascript
iframe.contentWindow.postMessage({
  type:    'eself-dynamic-prompt-message',
  content: JSON.stringify({ v: '2', user: { first_name: userName }, inst: [...] })
}, '*');
```

### Webhook

Session data is POSTed to `https://webhook.site/97746e0e-6905-498a-afde-3e5fb39fd3a4` on session end. The payload includes the user's name, session timestamps, transcript, and score.

### Results Page

`results.html` reads from `localStorage` and displays the score card. It loads automatically when `conversation-ended` fires from the avatar iframe.
