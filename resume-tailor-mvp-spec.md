# Resume Tailoring App — MVP Build Spec (for Google/Jules AI)

This single file is designed for an AI coding assistant to **plan, scaffold, and implement** a working MVP. It contains:
- clear scope, stack, and architecture
- exact prompts and outputs
- API contracts and data models
- acceptance criteria and a 2‑week sprint plan

> The tailoring logic is based on the user’s v9 prompt spec (formatting, ATS rules, deliverables). Follow those rules exactly when generating resume/cover letters and analysis docs【7†source】.

---

## 0) Goal

Build a **web app** that:
1) accepts a **Job Description (JD)** file/text and a **resume** file/text  
2) produces three outputs using LLM:
   - **Tailored Resume** (ATS-friendly, Google Docs compatible formatting)
   - **Cover Letter** (200–300 words)
   - **Analysis Doc** (keyword optimization, match assessment, interview prep, gap plan)
3) lets the user **preview, lightly edit, and export** (DOCX, PDF, Google Doc)  
4) stores nothing by default (or auto-deletes in 24h) unless user opts in

All formatting & validation rules MUST mirror the v9 spec (headers, bullets, no special chars, etc.)【7†source】.

---

## 1) Tech Stack

- **Frontend**: Next.js 14 (App Router), React 18, TypeScript, TailwindCSS, shadcn/ui
- **Backend**: Next.js **API routes** (Edge/Node) or /api in App Router
- **AI**: OpenAI Chat Completions (GPT‑4.1 or newer)
- **Storage**: In-memory for MVP; optional Supabase (Postgres + Storage) for opt‑in saved profiles
- **Auth**: NextAuth.js (Email magic link for MVP) — Optional for first cut
- **Docs export**: `@tiptap/react` (rich text), `html-to-docx`, `pdf-lib` (or server-side `docx` npm)
- **Deployment**: Vercel

> MVP can skip auth and persistence. Provide a `.env.local.example` for keys.

---

## 2) User Stories

- **US1**: As a user, I can upload a JD (PDF/DOCX/TXT) and my resume (PDF/DOCX/TXT) or paste text.
- **US2**: I can click **Generate** to get Tailored Resume, Cover Letter, and Analysis.
- **US3**: I can preview outputs, make light edits, and export to **DOCX** and **PDF**; link to “Open in Google Docs” via Drive API is a stretch goal.
- **US4**: I can see an **ATS & keyword match panel** that highlights incorporated JD keywords.
- **US5**: I can delete all data; the system auto-deletes in 24h by default.

---

## 3) Architecture Overview

```
/app
  /page.tsx                -> Landing + upload flow
  /tailor/page.tsx         -> Workspace (JD + Resume + Outputs)
  /api/tailor/route.ts     -> POST: runs LLM pipeline
  /api/extract/route.ts    -> POST: text extraction from PDF/DOCX
  /api/export/docx         -> POST: HTML/JSON -> DOCX
  /api/export/pdf          -> POST: HTML/JSON -> PDF
/components
  FileDrop.tsx             -> drag/drop + paste
  Editor.tsx               -> TipTap editor (resume, cover, analysis)
  KeywordPanel.tsx         -> ATS/keywords visualization
  StepsBar.tsx             -> “Analyze → Tailor → Validate”
/lib
  llm.ts                   -> OpenAI client + prompts
  parsing.ts               -> JD/resume text cleaners
  ats.ts                   -> rules checks (bullets, headers, chars)
  extract.ts               -> pdf/docx to text helpers
  exports.ts               -> docx/pdf helpers
```

---

## 4) Data & Types (TypeScript)

```ts
// JD & Resume payloads
export type UploadPayload = {
  jdText?: string;
  resumeText?: string;
  jdFileId?: string;      // if using storage
  resumeFileId?: string;
};

export type TailorRequest = {
  jd: string;
  resume: string;
  options?: {
    style?: "concise" | "balanced" | "detailed";
    coverTone?: "formal" | "friendly" | "crisp";
  };
};

export type TailorResponse = {
  tailoredResume: string; // Google-Docs-compatible plain formatting
  coverLetter: string;    // 200–300 words
  analysis: {
    keywordReport: string[];
    strengths: string[];
    concerns: string[];
    interviewNotes: string[];
    timingFollowups: string[];
    gapPlan: string[];
  };
  atsValidation: {
    passed: boolean;
    violations: string[];
  };
  keywordStats: { [keyword: string]: number };
};
```

---

## 5) API Contracts

### `POST /api/extract`
- body: `{ file: FormData }`
- returns: `{ text: string, meta: { pages?: number, filename: string } }`

### `POST /api/tailor`
- body: `TailorRequest`
- returns: `TailorResponse`

### `POST /api/export/docx`
- body: `{ htmlOrMarkdown: string, filename: string }`
- returns: `application/vnd.openxmlformats-officedocument.wordprocessingml.document`

### `POST /api/export/pdf`
- body: `{ htmlOrMarkdown: string, filename: string }`
- returns: `application/pdf`

---

## 6) Prompting (Use EXACT Rules from v9)

### 6.1 System Prompt (fragment)
> “You are an expert ATS optimization specialist and senior HR recruiting professional…” (see v9).  
Include the full **ANALYSIS PHASE** (steps 1–4), **TAILORING SPECIFICATIONS**, **WRITING GUIDELINES**, and **QUALITY CONTROL CHECKLIST** from the v9 prompt【7†source】.

### 6.2 Orchestration
Run **three sequential calls** with shared context:
1) **Analysis** → extract keywords, requirements, and gaps from JD; map to resume.
2) **Tailor** → generate ATS-compliant **Resume** and **Cover Letter** (follow formatting, headers, bullets, no special chars, etc.)【7†source】.
3) **Validate** → run the **MANDATORY PRE‑SUBMISSION VALIDATION** & **FINAL QUALITY GATE**; if violations exist, automatically regenerate once to fix【7†source】.

### 6.3 Output Contract (enforce)
Ask the model to return **strict JSON** matching `TailorResponse`, with `tailoredResume` and `coverLetter` as plain text (Google Docs friendly). Do **not** include markdown fences inside JSON.

---

## 7) ATS/Formatting Rules (from v9 — enforce in code)

- **Headers**: ALL CAPS & bold; contact info plain text.  
- **Bullets**: standard round `•` only; **no** arrows, pipes, or special characters.  
- **Date line** MUST be on its own line under title/company.  
- **Summary <4 lines**, bullets ≤2 lines; mix short/medium/long bullets.  
- **Cover letter 200–300 words**.  
- Run the **checklist** and **final quality gate** before returning results【7†source】.

Add a post-LLM sanitizer to:
- replace any invalid bullets/arrows/pipes,
- enforce header casing,
- verify lengths,
- compute **keywordStats** against JD terms.

---

## 8) UI Flow

1. **Upload Section**  
   - 2 dropzones (JD, Resume) + “Paste text instead” toggles
2. **Generate** → shows **StepsBar** with statuses: Analyzing → Tailoring → Validating
3. **Workspace** (3 tabs):  
   - **Tailored Resume** (Editor + export buttons)
   - **Cover Letter** (Editor + export)
   - **Analysis** (read-only + **KeywordPanel** charts)
4. **ATS Panel** (right side)  
   - Keyword match %, violations (with fix tips), word count, bullet count
5. **Privacy Notice** + Delete All

---

## 9) Example Code Stubs

### 9.1 `/lib/llm.ts`
```ts
import OpenAI from "openai";

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY! });

export async function runTailorPipeline(req: TailorRequest): Promise<TailorResponse> {
  const { jd, resume, options } = req;

  // 1) Analysis
  const analysis = await client.chat.completions.create({
    model: "gpt-4.1",
    temperature: 0.2,
    messages: [
      { role: "system", content: SYSTEM_PROMPT_ANALYST },
      { role: "user", content: buildAnalysisPrompt(jd, resume) }
    ],
    response_format: { type: "json_object" }
  });

  // 2) Tailor
  const tailor = await client.chat.completions.create({
    model: "gpt-4.1",
    temperature: 0.3,
    messages: [
      { role: "system", content: SYSTEM_PROMPT_TAILOR },
      { role: "user", content: buildTailorPrompt(jd, resume, analysis.choices[0].message.content!, options) }
    ],
    response_format: { type: "json_object" }
  });

  // 3) Validate
  const validate = await client.chat.completions.create({
    model: "gpt-4.1",
    temperature: 0,
    messages: [
      { role: "system", content: SYSTEM_PROMPT_VALIDATOR },
      { role: "user", content: buildValidatePrompt(tailor.choices[0].message.content!) }
    ],
    response_format: { type: "json_object" }
  });

  // sanitize + enforce contract
  return sanitizeAndCoerce(validate.choices[0].message.content!);
}
```

### 9.2 `/app/api/tailor/route.ts` (Next.js App Router)
```ts
import { NextRequest, NextResponse } from "next/server";
import { runTailorPipeline } from "@/lib/llm";

export async function POST(req: NextRequest) {
  const body = await req.json();
  const result = await runTailorPipeline(body);
  return NextResponse.json(result);
}
```

### 9.3 ATS Validator (outline)
```ts
export function validateAts(text: string) {
  const violations: string[] = [];
  if (/[|→←▪■★✓]/.test(text)) violations.push("Contains prohibited special characters.");
  if (!/^\*\*[A-Z ]+\*\$/m) {/* ensure headers ALL CAPS & bold if using markdown */}
  // …more checks per v9 spec
  return { passed: violations.length === 0, violations };
}
```

---

## 10) Environment & Setup

**.env.local.example**
```
OPENAI_API_KEY=sk-...
NEXT_PUBLIC_APP_NAME=Resume Tailor
AUTO_DELETE_HOURS=24
```

**Scripts**
```
pnpm dlx create-next-app@latest resume-tailor --typescript --tailwind --eslint
pnpm add openai zod tiptap react-hook-form @tanstack/react-query
pnpm add html-to-docx pdf-lib docx mammoth       # export + doc parsing
pnpm add shadcn-ui                                # or install per docs
```

---

## 11) Security & Privacy

- Do not log resume/JD content.  
- In-memory processing by default; if persistence is enabled, encrypt at rest and provide **Delete**.  
- Add a background job (cron) to purge items older than `AUTO_DELETE_HOURS`.

---

## 12) Acceptance Criteria (MVP)

- Upload JD & resume → click Generate → receive **all 3 outputs** within one flow.
- **Tailored Resume** conforms to v9 formatting rules and passes validator (no pipes/arrows, bullets = `•`, headers ALL CAPS, date line isolated)【7†source】.
- **Cover Letter** length is **200–300 words**, tone professional, no boilerplate clichés per v9【7†source】.
- **Analysis Doc** includes: keyword list & placement, strengths/concerns, interview prep, timing/follow‑ups, gap plan【7†source】.
- **Keyword panel** shows match % and per‑keyword counts.
- Export to **DOCX** and **PDF** works for resume & cover letter.

---

## 13) Two‑Week Sprint Plan

**Week 1**
- D1: Next.js scaffold, Tailwind, shadcn, basic layout
- D2: FileDrop + paste; `/api/extract` with `mammoth` (docx) + `pdf-parse`
- D3: LLM pipeline stub + env setup; strict JSON schema with `zod`
- D4: Workspace UI (tabs, editors); StepsBar progress
- D5: ATS validator + keyword extractor; keyword panel

**Week 2**
- D6: DOCX/PDF exports; file naming; download buttons
- D7: Polishing prompts per v9; automatic re‑validate & fix loop
- D8: Error states, loading skeletons, privacy info, delete flow
- D9: Manual QA with multiple JDs/resumes; edge cases
- D10: Deploy to **Vercel**, write README & troubleshooting

---

## 14) Test Checklist

- [ ] Pipes/arrows never present in outputs  
- [ ] Bullets render as `•` everywhere  
- [ ] Headers ALL CAPS & bold; date line separated below title/company  
- [ ] Summary < 4 lines; bullets ≤ 2 lines  
- [ ] Cover letter 200–300 words  
- [ ] JSON adheres to `TailorResponse` (no markdown fences)  
- [ ] Exports open in MS Word & Google Docs without layout issues  
- [ ] Data auto-deletes after 24h (or opt-out saved profiles)

---

## 15) Stretch Ideas

- LinkedIn URL scraper to auto‑pull JD text
- Saved profiles & template styles (concise vs narrative)
- Multiple cover letter variants (formal/friendly/crisp)
- OAuth Google Drive export (“Open in Google Docs”)

---

## 16) Implementation Notes for the AI Assistant

- Generate **strictly typed** code (TypeScript).  
- Keep prompts in `/lib/prompts/` with clear comments referencing v9 rule sections.  
- Build a **retry** on JSON parse failure.  
- Add **unit tests** for `ats.ts` and `parsing.ts`.  
- Prefer **edge‑safe** libraries for Vercel deployment.
