# 🎵 AI Agent for Suno — Automated Personalized Song Generator

> An n8n-based AI automation pipeline that receives a client order via a web form, generates personalized song lyrics with a structured LLM, validates the output, builds a Suno-compatible music prompt, and delivers the result directly to Telegram — fully automated, zero manual work.

---

## 📸 Workflow Overview

<img width="1729" height="660" alt="Снимок экрана 2026-02-21 151919" src="https://github.com/user-attachments/assets/d9d37107-eb0f-4895-b1cc-63ff11de6a8f" />


---

## ⚙️ How It Works

The pipeline consists of **11 nodes** that handle the full lifecycle of a song order — from raw form submission to a ready-to-use Suno prompt delivered to Telegram.

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

## 🧩 Node Breakdown

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

## ✍️ Prompt Engineering

Two LLM nodes are at the core of the pipeline. Each prompt is carefully engineered to enforce deterministic, structured output — critical for downstream validation and Suno compatibility.

---

### 🎼 Lyrics Generator

This node produces a personalized gift song. The prompt is split into a **system message** (role + output contract) and a **user message** (dynamic parameters).

#### System Prompt

```
You are a professional songwriter creating short personalized gift songs.

CRITICAL OUTPUT RULES (MANDATORY):
- You MUST return a JSON object
- The JSON MUST contain the client name exactly as provided
- The JSON MUST contain the full song lyrics
- Do NOT omit the name under any circumstances
- Do NOT return plain text
- Do NOT add explanations or comments

If the output is not valid JSON with both required fields, the task is FAILED.

Required JSON format:
{
  "name":  <client name>,
  "lyrics": "<full song lyrics>"
}

Song rules:
- Language, mood, genre must strictly follow the input
- Structure MUST be:
    Verse 1  – 4 lines
    Chorus   – 4 lines
    Verse 2  – 4 lines
    Bridge   – 2 lines
    Chorus   – EXACT repetition of the first chorus
- Lyrics must have rhyme and rhythm
- Use ONLY information from the description
- No real names inside lyrics
```

#### User Prompt (dynamic, filled from normalized data)

```
Write song lyrics using the exact structure defined above.

Parameters:
Language: {{ language }}
Genre:    {{ genre }}
Mood:     {{ mood }}
Purpose:  {{ purpose }}

About the person (USE ONLY THIS INFORMATION):
{{ description }}

Additional constraints:
- Use second-person address ("you")
- Maintain clear rhyme and steady rhythm
- Chorus must express the emotional core of the song
- Keep lyrics personal and specific, based only on the description
- Do NOT add any details that are not explicitly mentioned

Return ONLY the lyrics text.
Do NOT add explanations or formatting.
```

#### Prompt Design Decisions

**Dual-layer output enforcement.** The system prompt mandates a JSON object; the user prompt ends with `Return ONLY the lyrics text` — an intentional tension that forces the model to wrap plain lyrics in JSON. This eliminates the most common failure mode: the model returning raw text instead of a parseable structure.

**Hard failure framing.** The phrase *"the task is FAILED"* is a deliberate signal pattern. It sets a binary success criterion at the instruction level, which measurably increases compliance vs. soft phrasing like *"please try to"*.

**Identity-preserving name injection.** The client name is injected into the system prompt's required JSON schema, not just the user message. This double-binding ensures the name survives even when the model restructures its output.

**Scope control via exclusion.** `USE ONLY THIS INFORMATION` + `Do NOT add details not explicitly mentioned` prevents hallucinated biographical details — a critical requirement for a personalized gift product.

**No-name-in-lyrics rule.** Real names are excluded from lyrics to protect privacy and ensure the output remains generic enough to be reused if needed (e.g. audio generation).

---

### 🎧 Suno Prompt Generator

This node converts validated lyrics into a Suno-compatible music description. It receives structured lyrics from the Formatter and outputs a JSON object ready for direct use in Suno.

#### System Prompt

```
You generate Suno music prompts.

STRICT RULES:
- Output MUST be a valid JSON object matching the required schema
- Do NOT wrap JSON in text
- Do NOT add explanations or comments
- Do NOT modify the lyrics
- Preserve the client name EXACTLY

If the output is not valid JSON, the task is FAILED.
```

#### User Prompt (dynamic)

```
Client name (MUST be preserved exactly):
{{ name }}

Full song lyrics (MUST be preserved exactly):
Verse 1
{{ verse1 }}

Chorus
{{ chorus }}

Verse 2
{{ verse2 }}

Bridge
{{ bridge }}

Chorus
{{ chorus_repeat }}

Task:
Generate a Suno-compatible music prompt that matches the song.
Return ONLY the JSON object defined in the system prompt.
```

#### Prompt Design Decisions

**Lyrics passed verbatim in sections.** Rather than sending a raw lyrics string, the user prompt reconstructs the full song with structural labels (Verse 1, Chorus, etc.). This helps the model correctly infer style, tempo, and mood from the text itself.

**Minimal system prompt.** Unlike the Lyrics Generator, the system prompt here is deliberately brief. The output schema (style, mood, tempo, instruments, language) is implied — the model has enough context from the lyrics and the final node's parsing logic to infer the correct fields.

**Repeated name injection.** The client name appears in both system and user messages with `MUST be preserved exactly` — this prevents the model from "correcting" or translating the name, which is a known failure mode with non-English names.

**Separation of concerns.** This prompt has zero creative responsibility — it is purely a *conversion* task. All creative decisions (genre, mood, language) were resolved upstream in the Input Normalizer and Lyrics Generator. The prompt reinforces this by containing no creative instructions.

---

### 📦 Prompt Versioning

Every request is stamped with metadata before reaching the LLM:

```json
{
  "lyrics_prompt_meta": {
    "version": "lyrics_v1.0.0",
    "temperature": 0.7,
    "author": "natalia",
    "created_at": "2026-02-17"
  }
}
```

This enables A/B testing, regression tracking, and reproducibility — if output quality degrades after a prompt change, any request can be traced back to the exact prompt version that generated it.

---

### 🧪 Recommended Prompt Extensions

**Quality Check LLM (between Formatter → Suno Prompt Generator)**

Add a scoring node that evaluates the generated lyrics before they proceed:

```
You are a quality reviewer for personalized songs.

Evaluate the lyrics on the following criteria:
1. Does the text reflect the person described in the order? (1–10)
2. Is the rhyme scheme consistent? (1–10)
3. Does the mood match the requested mood? (1–10)
4. Is the structure correct (4+4+4+2+4)? (pass/fail)

Return:
{
  "score": <average of criteria 1–3>,
  "structure_valid": <true/false>,
  "pass": <true if score >= 7 and structure_valid>,
  "feedback": "<brief explanation>"
}
```

Route: `pass: false` → loop back to Lyrics Generator (max 3 retries); `pass: true` → continue.

**JSON Repair Prompt (error recovery)**

If the Lyrics Formatter returns `valid: false`, send the raw output to a repair prompt:

```
The following text was supposed to be a JSON object with "name" and "lyrics" fields,
but it failed validation for this reason: {{ error }}.

Fix it and return ONLY the corrected JSON. Do not change the content.

Original output:
{{ raw_text }}
```

---

## 🔧 Tech Stack

- **[n8n](https://n8n.io/)** — workflow automation platform (self-hosted or cloud)
- **[OpenRouter](https://openrouter.ai/)** — LLM API gateway (supports GPT-4, Claude, Mistral, etc.)
- **[Tally](https://tally.so/)** — no-code form builder for client intake
- **Gmail API** — email polling trigger
- **Telegram Bot API** — output delivery

---

## 🚀 Getting Started

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

---

## 📐 Song Structure (Enforced)

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

---

## Output Example
"🎵 Персональная песня

👤 Для: Сардана Кононова

━━━━━━━━━━━━━━━━━━

🎧 Промпт для Suno:

🎼 Стиль: warm heartfelt Russian ballad with acoustic guitar and gentle piano 
💫 Настроение: loving, supportive, intimate, calm 
⏱️ Темп: moderate 
🌍 Язык: Russian 
🎻 Инструменты: acoustic guitar, piano, soft percussion, strings

━━━━━━━━━━━━━━━━━━ 📝 Текст песни:

Verse 1 Ты заботлив и весел, добр, шутишь без конца, 
Иногда немного зануда, но это просто часть тебя. 
Порядок любишь в каждом деле, всё по местам лежит, 
Ручки, ножики собираешь — это сердце любит, как магнит.

Chorus Ты мой оплот и вдохновенье, рядом ты всегда со мной, 
Поддержка в каждом мгновенье, свет в душе и тишина. 
С тобой уют и пониманье, ты — мой самый верный друг, 
В подарок сердце отдаю, ты — мой нежный берег рук.

Verse 2 Ты любишь часто мыться, гладить рубашки в ряд, 
Стираешь аккуратно, всё у тебя как надо, взгляд. 
Всегда рядом, поддержишь, в трудностях не один, 
С тобой легко и просто, ты — мой надежный кин.

Bridge В каждом штрихе, в каждом дне — ты мой покой и свет, 
С тобой весь мир прекрасен, и я люблю ответ.

Chorus Ты мой оплот и вдохновенье, рядом ты всегда со мной, 
Поддержка в каждом мгновенье, свет в душе и тишина. 
С тобой уют и пониманье, ты — мой самый верный друг, 
В подарок сердце отдаю, ты — мой нежный берег рук."

---

## 🔮 Scalability & Extensions

This agent is built as a modular pipeline and can be extended in multiple directions:

### 📝 Custom Copywriting Orders
The same architecture works for **any text generation use case**:
- Birthday poems, toasts, wedding speeches
- SEO articles and blog posts
- Ad copy and product descriptions
- Personalized email sequences

Simply replace the Lyrics Generator system prompt with a copywriting prompt. The intake form, normalizer, and delivery nodes remain the same.

### ✅ Quality Check LLM (Recommended Extension)

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

### 🛡️ Error Handling (Recommended Extension)

Current pipeline has minimal error handling. Recommended additions:

| Error type | Handling strategy |
|------------|-------------------|
| LLM returns invalid JSON | Add a **JSON repair node** or retry loop (max 3 attempts) |
| Lyrics line count ≠ 18 | Route to a **correction prompt** asking the LLM to fix the structure |
| Choruses not identical | Inject the expected chorus and ask LLM to regenerate only the second one |
| Gmail API timeout | Add a **Wait + Retry** node |
| Telegram send failure | Add an **Error Trigger** node → fallback to email notification |
| Missing form fields | Validate required fields after parsing, send an automated reply requesting missing info |

### 📊 Other Scaling Ideas

- **Google Sheets logging** — log every order with status, timestamp, and output for CRM tracking
- **Airtable / Notion database** — build an order management board
- **Multiple delivery channels** — WhatsApp, email, Slack in addition to Telegram
- **Language expansion** — extend the Input Normalizer to support more languages (Spanish, German, French)
- **Model A/B testing** — route requests to different LLMs via OpenRouter and compare output quality
- **Webhook trigger** — replace Gmail polling with an instant Tally webhook for real-time processing
- **Audio generation** — after the Suno prompt is ready, trigger the Suno API directly (when available) and deliver the audio file

---

## 🔐 Security Notes

- Phone numbers are **automatically stripped** from form responses before any LLM processing
- No PII is stored within the workflow — only structured, normalized parameters are passed to the LLM
- Credentials are stored in n8n's encrypted credential store, not hardcoded in nodes
- Telegram `chatId` should be scoped to a private operator chat only

---

## 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

---

## 📄 License

MIT

---

*Built with n8n · OpenRouter · Tally · Telegram*
