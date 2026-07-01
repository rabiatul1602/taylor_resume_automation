# Resume Tailor Automation (n8n)

An n8n workflow that automatically tailors a resume to a specific job description using a free-tier LLM via OpenRouter — no manual copy-pasting, no fabricated content.

> 🎥 Demo video available — see below.

---

## What it does

1. Submit a job description through an n8n form
2. Workflow reads your resume from a local file
3. LLM analyses the JD (must-haves, nice-to-haves, responsibilities, tone)
4. Resume is rewritten to better match the JD — **reframing only, nothing invented**
5. Output is saved as a `.docx` file to disk via a Python watchdog script

---

## Workflow

```
On form submission
      ↓
Read/Write Files from Disk      (reads resume.txt)
      ↓
Extract from File               (Extract From Text File)
      ↓
Code in JavaScript2             (builds JD analysis request body)
      ↓
HTTP Request                    (POST → OpenRouter: analyse JD)
      ↓
Code in JavaScript1             (splits header/body, builds tailoring request)
      ↓
HTTP Request1                   (POST → OpenRouter: tailor resume)
      ↓
Code in JavaScript              (prepends header back, formats final output)
      ↓
Read/Write Files from Disk1     (writes tailored_text.txt to disk)
      ↓
[watchdog picks up file]
      ↓
watch_and_convert.py            (converts txt → tailored_resume.docx)
```
<img width="1844" height="811" alt="image" src="https://github.com/user-attachments/assets/7f11803d-56ec-4529-ab7b-6ff3f5151b83" />

---

## Stack

- **n8n** self-hosted, Windows, no Docker — v2.27.5
- **OpenRouter** free-tier LLM (`nvidia/nemotron-3-super-120b-a12b:free`)
- **Python** — `watchdog`, `python-docx` for post-processing
- Native n8n file nodes only — no `fs`, no external npm packages required

---

## Setup

### n8n side
- Place your resume at `C:\Users\<you>\.n8n-files\resume.txt`
- Add your OpenRouter API key to the HTTP Request node headers
- Submit a job description via the form trigger to run the workflow

### Python watchdog
```bash
pip install watchdog python-docx
python watch_and_convert.py
```

The script watches the `.n8n-files` folder. When `tailored_text.txt` appears, it converts it to `tailored_resume.docx` and archives the original as `tailored_text_0.txt`, `tailored_text_1.txt`, etc.

> ⚠️ Keep the docx closed before running — Windows will throw a permission error if the file is open in Word.

---

## LLM Rules (enforced via prompt)

- Do NOT invent any experience, skills, tools, or achievements
- Only reframe, reorder, and rephrase what already exists
- Surface keywords from the JD only where they genuinely apply
- Keep ALL original sections
- Format output to be ATS-friendly: plain bullet points, no tables or special characters
- Header block (name, contact, links) is hardcoded — never sent to LLM

---

## Constraints & design notes

| Challenge | How it's handled |
|---|---|
| `require('fs')` blocked in Code node sandbox | Use native Read/Write Files from Disk + Extract from File nodes instead |
| External npm packages blocked in sandbox | Python watchdog script handles docx conversion outside n8n |
| `Respond to Webhook` incompatible with Form Trigger | Output written to disk; user retrieves the docx directly |
| LLM removing name/contact header | Header extracted before sending to LLM, prepended back in final Code node |
| LLM adding meta-commentary to output | Explicit rule in prompt: output resume content only |
| File locked error on docx | Close the file in Word before triggering a new run |

---

## Demo

📹 [[Watch the demo video](https://drive.google.com/file/d/1F8LmnYtczvZjInMlfENSMEPhIwCS4_HQ/view?usp=sharing)]

---

## Status

✅ Core pipeline works end-to-end  
✅ Outputs valid `.docx`  
✅ ATS-friendly formatting  
✅ Header block preserved  
⚠️ Workflow JSON not exportable from this n8n version — manual recreation required

---

## Honest disclaimer

This tool does **not** fabricate or inflate resume content. It only reframes and rephrases real experience to better match the language of a job description.
