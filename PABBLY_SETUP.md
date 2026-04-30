# Pabbly + Telegram + Claude + Sheets — setup recipe

> **Goal**: chat your bot ("baby drank 3oz", "wet diaper", "pumped 4oz left 20m") → row appears in Google Sheets → web app shows it after sync.
>
> **Time**: ~45 min for first-time setup. ~5 min for repeats.
>
> **Cost**: free (Pabbly task quota + ~$0.0003/message in Anthropic Haiku).

---

## 1. Telegram bot

1. Open Telegram → search `@BotFather` → `/newbot`.
2. Pick a name and a unique username ending in `bot`. Save the **bot token**.
3. Send any message to your new bot from **your** Telegram and your **wife's** Telegram (one message each).
4. Open in a browser:
   ```
   https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
   ```
   Copy `result[].message.from.id` for each parent. These are the **chat IDs** you'll allowlist.

---

## 2. Google Sheet — "Baby Tracker"

Create one workbook, 5 tabs. **Header row of each tab must be exactly the column names below.** Order matters — Pabbly will map by position when you "Add Row".

### `feeds`
```
id | user_id | created_at | source | notes | subtype | side | duration_min | amount_oz | milk_kind
```

### `sleeps`
```
id | user_id | created_at | source | notes | start_at | end_at | duration_min
```

### `diapers`
```
id | user_id | created_at | source | notes | subtype
```
`subtype` = `wet` | `dirty` | `both`.

### `poos`
```
id | user_id | created_at | source | notes | consistency | color
```

### `pumps`
```
id | user_id | created_at | source | notes | side | amount_oz | duration_min
```

### Publish each tab as CSV
For every tab:
1. **File → Share → Publish to web**.
2. Pick this tab. Format: **Comma-separated values (.csv)**.
3. Click **Publish**. Copy the URL — it ends in `output=csv`.

You'll have 5 URLs. Paste them into `index.html` → `CLOUD = { ... }`.

> **Privacy reminder**: published CSVs are public to anyone who has the URL. That's fine for your family while validating; if you later open this to other families, follow the Phase-3 Supabase migration in the plan file.

---

## 3. Pabbly Connect workflow

### Trigger — Telegram Bot → New Message
- Connect your bot using the token from step 1.
- Test by sending a message; you should see the payload preview.

### Step 1 — Filter (allowlist)
- Condition: `Trigger > message > from > id` **equals** `<YOUR_CHAT_ID>` **OR** `<WIFE_CHAT_ID>`.
- If false, stop the workflow.

### Step 2 — API by Pabbly (call Claude)
- Method: `POST`
- URL: `https://api.anthropic.com/v1/messages`
- Headers (key → value):
  - `x-api-key` → `<YOUR_ANTHROPIC_KEY>`
  - `anthropic-version` → `2023-06-01`
  - `content-type` → `application/json`
- Body type: **Raw (JSON)**:
  ```json
  {
    "model": "claude-haiku-4-5",
    "max_tokens": 256,
    "system": "<PARSER_PROMPT below — paste as a single line, escaped>",
    "messages": [
      { "role": "user", "content": "{{Trigger.message.text}}" }
    ]
  }
  ```

### Step 3 — Data Transformer (parse Claude's JSON reply)
- Input: `Step 2 > content > 0 > text`
- Operation: **JSON Parse**.
- Now you can reference `type`, `subtype`, `amount_oz`, `duration_min`, `side`, `milk_kind`, `consistency`, `color`, `notes`, `confidence` in later steps.

### Step 4 — Filter on confidence
- Condition: `Step 3 > confidence` **>= 0.5** AND `Step 3 > type` **!=** `unknown`.
- If false → **Telegram Bot → Send Message**: "❓ I didn't catch that. Try: `drank 3oz`, `pumped 4oz left 20m`, `wet diaper`, `nursed 15 min right`." Then stop.

### Step 5 — Router by `type`
Five branches, one per type. In every branch, **Google Sheets → Add Row** to the matching tab. Common fields you set on **every** branch:
- `id` → `{{Trigger.message.date}}` × 1000 (or use Pabbly's "Date Time → Get Current Timestamp (ms)" formatter)
- `user_id` → `{{Trigger.message.from.id}}`
- `created_at` → current ISO timestamp (Pabbly Date Time formatter)
- `source` → `telegram`
- `notes` → `{{Step 3.notes}}`

Branch-specific fields:

| Branch (type) | Tab | Fields |
|---|---|---|
| `feed`   | feeds   | `subtype`={{Step 3.subtype}}, `side`={{Step 3.side}}, `duration_min`={{Step 3.duration_min}}, `amount_oz`={{Step 3.amount_oz}}, `milk_kind`={{Step 3.milk_kind}} |
| `sleep`  | sleeps  | `start_at`=ISO of (now − duration_min), `end_at`=now ISO, `duration_min`={{Step 3.duration_min}} |
| `diaper` | diapers | `subtype`={{Step 3.subtype}} |
| `poo`    | poos    | `consistency`={{Step 3.consistency}}, `color`={{Step 3.color}} |
| `pump`   | pumps   | `side`={{Step 3.side}}, `amount_oz`={{Step 3.amount_oz}}, `duration_min`={{Step 3.duration_min}} |

### Step 6 — Telegram reply (confirmation)
**Telegram Bot → Send Message** to `{{Trigger.message.chat.id}}`:
```
✅ Logged: {{Step 3.type}} · {{Step 3.subtype}}{{#if amount_oz}} · {{Step 3.amount_oz}}oz{{/if}}{{#if duration_min}} · {{Step 3.duration_min}}m{{/if}}
```
(Pabbly's syntax differs slightly; build the string with its inline conditions.)

---

## 4. Claude parser system prompt (`PARSER_PROMPT`)

Paste this as the `system` in Step 2. Single string, JSON-escape the newlines (`\n`):

```
You parse baby and breastfeeding tracker messages into strict JSON. Reply with ONE JSON object, no prose, no code fences.

Schema (omit fields that don't apply):
{
  "type": "feed" | "sleep" | "diaper" | "poo" | "pump" | "unknown",
  "subtype": "breast" | "bottle" | "wet" | "dirty" | "both",
  "side": "L" | "R" | "both",
  "amount_oz": number,
  "duration_min": number,
  "milk_kind": "formula" | "ebm",
  "consistency": "soft" | "normal" | "hard" | "watery",
  "color": string,
  "notes": string,
  "confidence": number
}

Rules:
- "drank/bottle/fed Xoz" -> type=feed, subtype=bottle, amount_oz=X.
- "nursed/breastfed X min [left|right|both]" -> type=feed, subtype=breast.
- "pumped Xoz [left|right|both] [Y min]" -> type=pump.
- "wet/pee" alone -> diaper, subtype=wet. "dirty/poo/poop" -> poo. "wet and dirty" -> diaper, subtype=both.
- "slept X[h]Ym" or "nap X min" -> sleep with duration_min.
- Convert ml to oz when needed (1 oz = 30 ml, round to 1 decimal).
- Accept Malay (susu, kencing, berak, tidur) and informal English.
- If type unclear, type="unknown" and confidence < 0.5.

Examples:
"baby drank 3oz" -> {"type":"feed","subtype":"bottle","amount_oz":3,"confidence":0.95}
"pumped 4oz left side, 20 min" -> {"type":"pump","side":"L","amount_oz":4,"duration_min":20,"confidence":0.95}
"wet diaper" -> {"type":"diaper","subtype":"wet","confidence":0.98}
"nursed 15 min right" -> {"type":"feed","subtype":"breast","side":"R","duration_min":15,"confidence":0.95}
"susu botol 90ml" -> {"type":"feed","subtype":"bottle","amount_oz":3,"notes":"converted from 90ml","confidence":0.9}
"hello" -> {"type":"unknown","confidence":0.1}
```

---

## 5. Wire the web app

1. Open `baby-tracker/index.html`.
2. Find the `CLOUD` constant at the top of the `<script>`.
3. Replace each `<paste-...-csv-url>` placeholder with the published CSV URL of the matching tab.
4. Save. Open the app — the sync chip in the header should turn from "cloud not configured" to "synced just now" after a moment.

---

## 6. End-to-end test

| Send to bot | Expected |
|---|---|
| `baby drank 3oz` | ✅ reply, row in `feeds` (subtype=bottle, amount_oz=3) |
| `pumped 4oz left 20m` from wife | ✅ reply, row in `pumps` (side=L, amount_oz=4, duration_min=20) |
| `wet diaper` | ✅ reply, row in `diapers` (subtype=wet) |
| `slept 1h 20m` | ✅ reply, row in `sleeps` (duration_min=80) |
| `hello` | ❓ reply, no row written |
| (from a third Telegram) | silently ignored |

After each: open the web app → tap the sync chip → entry appears.

---

## Gotchas

- **CSV cache**: Sheets CSV publishing has a ~5 min delay. New rows may not appear immediately on the web app even with hard sync. Pabbly writes the row instantly; only the *published* CSV view lags.
- **Schema drift**: if you ever rename a column, also update the matching field name in `index.html`'s `rowTo*` mapper functions.
- **Anthropic rate limits**: free tier on Haiku handles ~50 messages/minute — way more than two parents will ever send.
- **Bot token leaks**: the token lives only in Pabbly. Don't paste it into Sheets or HTML.
