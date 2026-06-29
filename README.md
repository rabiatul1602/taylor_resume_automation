# Taylor Resume Automation

An n8n workflow that takes a job description as input, analyses it with an LLM via OpenRouter, and rewrites your resume to better match — without fabricating anything. Output is a downloadable `.docx` file.

---

## What it does

1. Accept a job description via an n8n form
2. Read your resume from a local `.txt` file
3. Send the JD to OpenRouter to extract key skills, responsibilities, and tone
4. Rewrite your resume sections to surface relevant experience — no hallucination, existing content only
5. Return a formatted `.docx` file as a download

---

## Workflow overview

```
Form Trigger
  → Read File (resume.txt)
  → Code: build JD analysis prompt
  → HTTP Request: OpenRouter — analyse JD
  → Code: build resume tailoring prompt
  → HTTP Request: OpenRouter — tailor resume
  → Python / Flask: generate .docx
  → Respond to Webhook (binary download)
```

---

## Stack

| Tool | Purpose |
|---|---|
| n8n (self-hosted, v2.27.5) | Workflow orchestration |
| OpenRouter free tier | LLM API — `google/gemma-3-27b-it:free` |
| Python + `python-docx` | `.docx` file generation |
| Windows (local) | Host environment |

---

## Setup

### Prerequisites

- n8n self-hosted installed
- OpenRouter account — [openrouter.ai](https://openrouter.ai) (free tier works)
- Python with `python-docx` installed
- Resume saved as plain text

### 1. Resume file

Save your resume as a `.txt` file here:

```
C:\Users\<you>\.n8n-files\resume.txt
```

Use this section format:

```
SUMMARY:
Your summary here.

EXPERIENCE:
Job Title, Company Name, Date Range
- Bullet point
- Bullet point

EDUCATION:
Degree, University, Year

SKILLS:
Skill 1, Skill 2, Skill 3
```

### 2. n8n allowed file path

Set this environment variable so n8n can read local files:

```
N8N_ALLOWED_FILE_PATHS=C:\Users\<you>\.n8n-files\
```

### 3. OpenRouter API key

Get your free key at [openrouter.ai](https://openrouter.ai) and add it to the HTTP Request node headers:

```
Authorization: Bearer YOUR_KEY
Content-Type: application/json
```

### 4. .docx generation

The n8n Code node sandbox blocks `fs` and external npm modules, so `.docx` generation runs outside the main workflow. Two options:

**Option A — Python Code node**
Use the n8n Python Code node if your Python environment has `python-docx` installed.

**Option B — Local Flask API (recommended)**
Run a small Python server locally:

```bash
python generate_resume.py
```

The HTTP Request node calls `http://localhost:5000/generate`, receives the `.docx` binary, and passes it to the Respond to Webhook node.

---

## Node details

### Form Trigger
Single field: `jd_text` (Textarea, Required)

### Read/Write Files from Disk
- Operation: Read
- Path: `C:\Users\<you>\.n8n-files\resume.txt`

### Code — JD analysis prompt

```javascript
const jdText = $('n8n Form Trigger').item.json.jd_text;
const body = {
  model: "google/gemma-3-27b-it:free",
  messages: [{
    role: "user",
    content: `Analyse this job description and extract in JSON format:
1. must_have: array of must-have skills and keywords
2. preferred: array of nice-to-have skills
3. responsibilities: array of key responsibilities
4. tone: one sentence describing the company tone

Return ONLY valid JSON, no explanation.

JD:
${jdText}`
  }]
};
return [{ json: { body: JSON.stringify(body) } }];
```

### HTTP Request — Analyse JD
- Method: POST
- URL: `https://openrouter.ai/api/v1/chat/completions`
- Body: Raw → `={{ $json.body }}`

### Code — Resume tailoring prompt

```javascript
const resumeText = $('Read/Write Files from Disk').item.json.data;
const jdAnalysis = $('HTTP Request').item.json.choices[0].message.content;
const body = {
  model: "google/gemma-3-27b-it:free",
  messages: [{
    role: "user",
    content: `You are a resume writer. Rewrite this resume to better match the job description analysis below.

STRICT RULES:
- Do NOT invent any experience, skills, tools, or achievements
- Only reframe, reorder, and rephrase what already exists
- Surface keywords from the JD only where they genuinely apply
- Keep ALL original sections
- Output each section as SECTION_NAME: followed by content

Original Resume:
${resumeText}

JD Analysis:
${jdAnalysis}`
  }]
};
return [{ json: { body: JSON.stringify(body) } }];
```

### HTTP Request — Tailor resume
Same config as the first HTTP Request node.
- Body: Raw → `={{ $json.body }}`

### Respond to Webhook
- Respond With: Binary
- Binary Property: `data`

---

## Known constraints

| What was tried | Result |
|---|---|
| `require('fs')` in Code node | Blocked — sandbox disallows it |
| External npm packages in Code node | Blocked |
| File access outside `.n8n-files` | Blocked by n8n |
| "Execute Command" node | Not available in n8n v2.27.5 |

---

## Honest use policy

This tool only rephrases and reorders existing resume content. It does not invent skills, tools, or experience. The LLM prompt explicitly enforces this — the model is instructed not to fabricate anything.

---

## Status

- [x] Form trigger working
- [x] Resume file read from disk
- [x] JD analysis via OpenRouter
- [x] Resume tailoring via OpenRouter
- [ ] `.docx` generation — evaluating Python Code node vs local Flask API
- [ ] End-to-end download test

---

## License

MIT
