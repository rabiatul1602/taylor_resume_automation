# Resume Tailor Automation (n8n)

An n8n workflow that automatically tailors a resume to a specific job description using a free-tier LLM via OpenRouter — no manual copy-pasting, no fabricated content.

## What it does

1. Submit a job description through a simple form
2. Workflow reads your resume from a local file
3. JD is analysed by an LLM (must-haves, nice-to-haves, responsibilities, tone)
4. Resume is rewritten to better match the JD — **reframing only, nothing invented**
5. Output is saved as a file to disk

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
Code in JavaScript1             (builds resume tailoring request body)
      ↓
HTTP Request1                   (POST → OpenRouter: tailor resume)
      ↓
Code in JavaScript              (formats final output)
      ↓
Read/Write Files from Disk1     (writes tailored resume to disk)
```

## Stack

- **n8n** (self-hosted, Windows, no Docker)
- **OpenRouter** free-tier LLM (`google/gemma-3-27b-it:free`)
- Native n8n file nodes only — no `fs`, no external npm packages (sandbox-safe)

## Setup

- Resume file expected at a fixed local path (set in "Read/Write Files from Disk")
- OpenRouter API key set in the HTTP Request node headers
- Submit JD text via the form trigger to run the workflow

## Constraints / design notes

- n8n's Code node sandbox blocks `require('fs')` and external modules by default — this workflow avoids the issue entirely by using native **Read/Write Files from Disk** and **Extract from File** nodes instead of scripting file I/O.
- `Respond to Webhook` is not compatible with Form Trigger-based workflows — output is written to disk instead of streamed back as a download.
- Currently outputs as `.txt`. `.docx` export is a planned next step (building a minimal OOXML/zip structure in-code, since external zip libraries aren't available in the sandbox).

## Status

🚧 Work in progress — core tailoring pipeline works end-to-end; `.docx` output not yet implemented.

## Rules this project follows

- No fabricated or inflated resume content — only rephrasing/reframing of real experience
- All original resume sections are preserved
