# AI Agent for Suno — Automated Personalized Song Generator

> An n8n-based AI automation pipeline that receives a client order via a web form, generates personalized song lyrics with a structured LLM, validates the output, builds a Suno-compatible music prompt, and delivers the result directly to Telegram — fully automated, zero manual work.

---

## Workflow Overview

<img width="1729" height="660" alt="Снимок экрана 2026-02-21 151919" src="https://github.com/user-attachments/assets/4c95ff02-6a6a-4ec3-9333-2b4cbb1e0392" />

---

## How It Works

The pipeline consists of 11 nodes that handle the full lifecycle of a song order — from raw form submission to a ready-to-use Suno prompt delivered to Telegram.

```
Tally Form Submission
       ↓
Gmail Trigger (polls every hour)
       ↓
Get Full Email
       ↓
Raw Email → Clean Text
       ↓
Remove Phone Numbers (PII cleanup)
       ↓
Tally Text Parser
       ↓
Input Normalizer  (language / genre / mood → structured JSON)
       ↓
Add Prompt Versioning
       ↓
Lyrics Generator  ← OpenRouter LLM
       ↓
Lyrics Formatter  (structure validator: 18 lines, chorus identity check)
       ↓
Suno Prompt Generator  ← OpenRouter LLM
       ↓
Final Output  (format Telegram message)
       ↓
Send to Telegram
```

---

##  Node Breakdown

| Node | Type | Description |
|------|------|-------------|
| **Gmail Trigger** | Trigger | Polls Gmail every hour for new emails from `notifications@tally.so` |
| **Get a message** | Gmail | Fetches the full email body by message ID |
| **Raw Email → Clean Text** | Code (JS) | Strips email metadata, extracts plain text |
| **Remove Phone Numbers** | Code (JS) | Removes phone numbers via regex for privacy compliance |
| **Tally Text Parser** | Code (JS) | Parses Q&A structure from the Tally form response |
| **Input Normalizer** | Code (JS) | Normalizes raw answers to structured JSON (language: `ru/en`, genre: `pop/rock/jazz`, mood: `romantic/sad/energetic/warm`) |
| **Add Prompt Versioning** | Code (JS) | Stamps each request with prompt version metadata for traceability |
| **Lyrics Generator** | LLM Chain | Generates 18-line lyrics (4+4+4+2+4) with strict structure enforcement via system prompt |
| **Lyrics Formatter** | Code (JS) | Validates 18-line count, checks chorus identity, outputs structured JSON |
| **Suno Prompt Generator** | LLM Chain | Generates a Suno-compatible music prompt (style, mood, tempo, instruments, language) |
| **Final Output** | Code (JS) | Formats the full Telegram message in HTML with lyrics + Suno prompt |
| **Send to Telegram** | Telegram | Sends the formatted message to the operator's Telegram chat |

---

## Tech Stack

- **[n8n](https://n8n.io/)** — workflow automation platform (self-hosted or cloud)
- **[OpenRouter](https://openrouter.ai/)** — LLM API gateway (supports GPT-4, Claude, Mistral, etc.)
- **[Tally](https://tally.so/)** — no-code form builder for client intake
- **Gmail API** — email polling trigger
- **Telegram Bot API** — output delivery

---

## Getting Started

### Prerequisites

- n8n instance (self-hosted via Docker or n8n Cloud)
- OpenRouter API key
- Gmail OAuth2 credentials
- Telegram Bot token
- Tally form with the required fields (see below)

### Required Tally Form Fields

The parser expects the following questions in the form:

| Question (Russian) | Expected answer type |
|--------------------|----------------------|
| Как вас зовут | Text — client name |
| Язык песни | Russian / English |
| Тип вокала | Женский / Мужской |
| Жанр/стиль песни | Pop / Rock / Jazz / etc. |
| Настроение песни | Романтическое / Грустное / Драйв / etc. |
| О чём песня | Free text description |
| Для чего нужна песня | Purpose / occasion |
| К какой дате нужна песня | Deadline date |

### Installation

1. Clone this repository
2. Import `AI_Agent_for_Suno.json` into your n8n instance via **Settings → Import workflow**
3. Connect your credentials:
   - Gmail OAuth2
   - OpenRouter API key
   - Telegram Bot API
4. Set your Telegram `chatId` in the **Send a text message** node
5. Activate the workflow

---

## Song Structure (Enforced)

The **Lyrics Formatter** node strictly validates the following structure:

```
Verse 1   — 4 lines
Chorus    — 4 lines
Verse 2   — 4 lines
Bridge    — 2 lines
Chorus    — 4 lines (must be IDENTICAL to the first chorus)
─────────────────
Total: 18 lines
```

If the LLM returns an incorrect number of lines or different choruses, the node flags `valid: false` with a detailed error message.

### Output Example

"🎵 Персональная песня

👤 Для: Сардана Кононова

━━━━━━━━━━━━━━━━━━

🎧 Промпт для Suno:

🎼 Стиль: warm heartfelt Russian ballad with acoustic guitar and gentle piano
💫 Настроение: loving, supportive, intimate, calm
⏱️ Темп: moderate
🌍 Язык: Russian
🎻 Инструменты: acoustic guitar, piano, soft percussion, strings

━━━━━━━━━━━━━━━━━━
📝 Текст песни:

Verse 1
Ты заботлив и весел, добр, шутишь без конца,
Иногда немного зануда, но это просто часть тебя.
Порядок любишь в каждом деле, всё по местам лежит,
Ручки, ножики собираешь — это сердце любит, как магнит.

Chorus
Ты мой оплот и вдохновенье, рядом ты всегда со мной,
Поддержка в каждом мгновенье, свет в душе и тишина.
С тобой уют и пониманье, ты — мой самый верный друг,
В подарок сердце отдаю, ты — мой нежный берег рук.

Verse 2
Ты любишь часто мыться, гладить рубашки в ряд,
Стираешь аккуратно, всё у тебя как надо, взгляд.
Всегда рядом, поддержишь, в трудностях не один,
С тобой легко и просто, ты — мой надежный кин.

Bridge
В каждом штрихе, в каждом дне — ты мой покой и свет,
С тобой весь мир прекрасен, и я люблю ответ.

Chorus
Ты мой оплот и вдохновенье, рядом ты всегда со мной,
Поддержка в каждом мгновенье, свет в душе и тишина.
С тобой уют и пониманье, ты — мой самый верный друг,
В подарок сердце отдаю, ты — мой нежный берег рук."

---

## Scalability & Extensions

This agent is built as a modular pipeline and can be extended in multiple directions:

### Custom Copywriting Orders
The same architecture works for **any text generation use case**:
- Birthday poems, toasts, wedding speeches
- SEO articles and blog posts
- Ad copy and product descriptions
- Personalized email sequences

Simply replace the Lyrics Generator system prompt with a copywriting prompt. The intake form, normalizer, and delivery nodes remain the same.

### Quality Check LLM (Recommended Extension)

Add a **QC node** between `Lyrics Formatter` and `Suno Prompt Generator`:

```
Lyrics Formatter
       ↓
[NEW] Quality Check LLM
   - Does the text match the described person?
   - Is the rhyme scheme consistent?
   - Is the mood aligned with the request?
   - Score: 1–10 with pass/fail threshold
       ↓
  IF score < 7 → loop back to Lyrics Generator (max 3 retries)
  IF score ≥ 7 → continue to Suno Prompt Generator
```

This prevents low-quality outputs from reaching the client without any human review.

### Error Handling (Recommended Extension)

Current pipeline has minimal error handling. Recommended additions:

| Error type | Handling strategy |
|------------|-------------------|
| LLM returns invalid JSON | Add a **JSON repair node** or retry loop (max 3 attempts) |
| Lyrics line count ≠ 18 | Route to a **correction prompt** asking the LLM to fix the structure |
| Choruses not identical | Inject the expected chorus and ask LLM to regenerate only the second one |
| Gmail API timeout | Add a **Wait + Retry** node |
| Telegram send failure | Add an **Error Trigger** node → fallback to email notification |
| Missing form fields | Validate required fields after parsing, send an automated reply requesting missing info |

### Other Scaling Ideas

- **Google Sheets logging** — log every order with status, timestamp, and output for CRM tracking
- **Airtable / Notion database** — build an order management board
- **Multiple delivery channels** — WhatsApp, email, Slack in addition to Telegram
- **Language expansion** — extend the Input Normalizer to support more languages (Spanish, German, French)
- **Model A/B testing** — route requests to different LLMs via OpenRouter and compare output quality
- **Webhook trigger** — replace Gmail polling with an instant Tally webhook for real-time processing
- **Audio generation** — after the Suno prompt is ready, trigger the Suno API directly (when available) and deliver the audio file

---

## Security Notes

- Phone numbers are **automatically stripped** from form responses before any LLM processing
- No PII is stored within the workflow — only structured, normalized parameters are passed to the LLM
- Credentials are stored in n8n's encrypted credential store, not hardcoded in nodes
- Telegram `chatId` should be scoped to a private operator chat only

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

---

## License

MIT

---

*Built with n8n · OpenRouter · Tally · Telegram*
