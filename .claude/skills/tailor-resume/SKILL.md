---
name: tailor-resume
description: >
  Main orchestration skill for the ATS Resume Tailor. Takes an original resume and a
  job description, runs full analysis and tailoring pipeline, and outputs ATS-optimized
  resume documents plus a cover letter and gap coaching report. Invoke with:
  /tailor-resume <resume_path> <jd_path_or_url> [output_dir]
user-invocable: true
argument-hint: "<resume_path> <jd_path_or_url> [output_dir]"
context: fork
agent: general-purpose
allowed-tools: Read, Write, Bash, Glob, Grep, Agent, WebFetch
effort: high
---

# ATS Resume Tailor — Orchestration

You are the orchestrator for the ATS Resume Tailor pipeline. When invoked, you produce a
fully tailored, ATS-optimized resume, cover letter, and gap coaching report.

## Arguments

Parse `$ARGUMENTS`:
- `$1` — Path to the original resume file (required)
- `$2` — Path to the job description file OR a URL starting with `http` (required)
- `$3` — Output directory (optional, defaults to same directory as resume)

Optional flags anywhere in `$ARGUMENTS`:
- `--skip-preferences` — skip Step 0 entirely
- `--one-page-pdf` — force PDF to fit on one page
- `--page-count N` — target page count (default 1)
- `--cover-letter` — generate cover letter without prompting (default: ask user)
- `--no-cover-letter` — skip cover letter generation without prompting
- `--no-gap-coach` — skip the gap coaching report

If required args are missing:
> "Usage: /tailor-resume <resume_path> <jd_path_or_url> [output_dir] [--skip-preferences] [--one-page-pdf]"

---

## Step 0 — User Preference Intake

Skip if `--skip-preferences` is set. Otherwise, ask once and accept free-text answers:

```
Before I start tailoring, a few quick questions (press Enter to skip any):

1. What aspects of your experience should I emphasize most?
2. Are there any roles, skills, or content you want excluded or downplayed?
3. What tone should the resume have? formal | startup | technical | balanced (default)
```

Store answers as `user_preferences`.

---

## Step 1 — Validate Inputs + Fetch JD (Parallel)

**Task A — Resume:** Confirm file exists, detect format (.md/.txt/.pdf/.docx), read content.
For PDF input: warn that text extraction loses formatting; proceed.

**Task B — JD:** If `$2` starts with `http`, WebFetch the URL and strip nav/footer/UI chrome.
Otherwise read the file. Confirm content is ≥100 words; warn if shorter.

Print: `> [1/6] Inputs validated — Resume: [filename], JD: [source]`

---

## Step 2 — Analyze JD

Delegate to the `jd-analyzer` agent:
> "Analyze the job description at [jd_path] and return the full JSON analysis."

Store the JSON result. Display:
```
> [2/6] JD Analysis Complete
>   Role: [role_title] ([seniority_level])
>   Top ATS Keywords: [first 5]
>   Required Skills: [count]
>   Implied Keywords: [first 3]
```

---

## Step 3 — Tailor Resume

Delegate to the `resume-tailor` agent with:
- Original resume content (Step 1)
- JD analysis JSON (Step 2)
- User preferences (Step 0)
- Instruction: "Preserve the original resume's exact structure, candidate name, contact
  info, sections, and entry list. Only rewrite bullets and skills keywords."

Print: `> [3/6] Resume tailoring in progress...`
Store the tailored Markdown.

---

## Step 4 — Score & Re-Tailor Loop

Delegate to the `ats-scorer` agent with the tailored resume + JD analysis.
Track `pass_number = 1`, `previous_score = null`.

Display:
```
> [4/6] ATS Score — Pass 1
>   Overall: [score]/100 (Grade: [grade])
>   Keyword Match: [/40] | Required Skills: [/25] | Action Verbs: [/10]
>   Quantification: [/15] | Formatting: [/10]
>   ATS System: [detected]
```

**Re-tailoring loop** (automatic, no user prompt):

While `score < 80` AND `pass_number < 3`:
1. If `previous_score` exists and `score - previous_score < 5`: stop with
   "Score improvement stalled at [score]/100 — proceeding."
2. Re-run `resume-tailor` with the score report attached. Instruction: "Pass [N].
   Address every `critical_issues` item, then apply each `quick_win`. Only modify the
   bullets/sections named — do not re-rewrite well-scoring sections."
3. Re-run `ats-scorer`. Increment counters. Display new score.

If score ≥ 88 after Pass 1: skip re-tailoring with
> "Score [score]/100 — excellent match. Skipping re-tailoring pass."

---

## Step 5 — Cover Letter (inline — opt-in)

**Gating logic** (skip the rest of this step if cover letter is not wanted):

1. If `--no-cover-letter` flag is set: skip immediately, print `> [5/6] Cover letter skipped (--no-cover-letter).`
2. Else if `--cover-letter` flag is set: proceed to generation.
3. Else (no flag): ask the user once:
   ```
   Generate a cover letter for this application? (y/N) [defaults to N if no answer]
   ```
   If response is empty or starts with `n`/`N`: skip and print `> [5/6] Cover letter skipped (user declined).`
   If response starts with `y`/`Y`: proceed to generation.

If skipped, the version registry's `cover_letter` field is `null`.

If proceeding, generate a 3-paragraph cover letter directly. Inputs:
- Final tailored resume
- JD analysis JSON
- Candidate name (from resume header) and company (from JD)

**Structure:**

**Paragraph 1 — Hook (3–4 sentences):**
- Open with role title + company
- Mirror 1–2 `cultural_signals` from JD analysis
- State candidate's most relevant positioning in one sentence
- Do NOT start with "I am writing to apply for..." — dead opener

**Paragraph 2 — Evidence (4–5 sentences):**
- Pick 2–3 specific resume experiences mapping to top `responsibilities_summary` items
- Use ≥2 `action_verbs` from JD analysis
- Include 1–2 quantified results (only real numbers from resume — never invent)
- Embed 1–2 `ats_keywords` naturally
- Cite specific projects/technologies/outcomes

**Paragraph 3 — Close (2–3 sentences):**
- Express interest in company mission/product (use `industry_terminology`)
- Forward-looking sentence about what you'd contribute
- Professional close with a call to action

**Tone matches `seniority_level`:**
- entry/mid: conversational but professional
- senior/staff/principal: confident, peer-to-peer
- director/executive: strategic, vision-oriented

**Constraints:**
- 250–350 words total
- No bullets, tables, or headers in body
- Avoid: "I am excited", "I believe I would be a great fit", "passionate about"
- Never repeat the resume verbatim — reframe in cover-letter voice

**Output format:**
```markdown
<!--
COVER LETTER FOR: [role_title] at [company]
GENERATED: [today's date]
JD THEMES ADDRESSED: [3 themes from responsibilities_summary]
-->

**Subject:** [Candidate Name] — [Role Title] Application

---

[Paragraph 1]

[Paragraph 2]

[Paragraph 3]

Sincerely,
[Candidate Name]
```

Save to: `[output_dir]/tailored_resumes/[CandidateName]_[RoleTitle]_Cover_Letter.md`

Print: `> [5/6] Cover letter generated.`

---

## Step 5.5 — Gap Coaching Report (inline — opt-out)

If `--no-gap-coach` is set, skip this step and print `> Gap coaching skipped (--no-gap-coach).`
Otherwise, generate a structured coaching report directly. Inputs:
- JD analysis JSON (`required_skills`, `must_have_vs_nice_to_have`, `ats_keywords`)
- GAPS list from tailored resume's `<!-- GAPS: -->` block
- Final ATS score report (`score_breakdown.required_skills.missing`, `critical_issues`)
- Tailored resume content (to assess closest analogues)

**For each gap, produce this Markdown block:**

```markdown
## Gap: [skill or keyword name]

**Severity:** Critical | Important | Minor
- Critical = appears in `must_have`, or required skill missing entirely
- Important = appears in `ats_keywords`, partially addressed
- Minor = preferred skill, easy to bridge

**What the JD Expects:** [1–2 sentences]

**What You Have:** [1–2 sentences identifying closest analogue — be honest if none]

**Interview Framing:** [Specific language to address this gap honestly. Include a sample sentence.]

**Bridge Strategy:** [Concrete 1–4 week action — "build a small X using Y library and publish it",
not "learn more about X"]

**Vocabulary to Learn:** [2–4 domain terms to speak to fluently]
```

**Coaching principles:**
- Be honest — if a gap is substantial, say so
- Be specific — concrete actions, not platitudes
- Bridge, don't fabricate — work from what the candidate actually has
- Prioritize by severity, leading with Critical
- Acknowledge the closest strength for each gap

**Footer:**
```markdown
---
## Priority Action List

1. [Highest-severity gap — specific action]
2. [Second gap — specific action]
3. [Third gap — specific action]

## Gaps That Are Low Risk

[Gaps unlikely to disqualify based on overall profile]

## Honest Assessment

[2–3 sentences: Is this candidate competitive? Strongest positioning angle?
Most likely hiring-manager objection?]
```

Save to: `[output_dir]/tailored_resumes/[CandidateName]_[RoleTitle]_Gap_Coach.md`

Print: `> Gap coaching report generated.`

---

## Step 6 — Generate Output Documents

Determine output directory: `$3` if provided, else same directory as the original resume.

Delegate to `resume-document-generator` agent with:
- Final tailored resume Markdown (Step 4)
- Output directory path
- Candidate name and role title
- `--one-page` if `--one-page-pdf` flag was passed (default true)
- `--page-count` value if provided
- `--original-pdf [path]` if the original input was a PDF — enables font + layout
  detection so the new PDF mirrors the original's typography

Print final summary from the document generator.

---

## Step 7 — Update Version Registry

Append a record to `[output_dir]/version_registry.json` (create as `[]` if missing):

```json
{
  "date": "[today's date]",
  "jd_source": "[file path or URL]",
  "jd_company": "[company or null]",
  "role": "[role_title]",
  "score": [final_score],
  "grade": "[grade]",
  "passes": [pass_number],
  "ats_system": "[detected]",
  "top_gap": "[first GAPS item or null]",
  "files": {
    "md": "[path]",
    "txt": "[path]",
    "pdf": "[path or null if failed]",
    "cover_letter": "[path]",
    "gap_coach": "[path]"
  }
}
```

---

## Final Output to User

```
RESUME TAILORING COMPLETE
==========================
Original Resume:  [original_path]
Job Description:  [jd_source]
ATS Score:        [score]/100 (Grade: [grade])  [Pass N if > 1]
ATS System:       [detected]

Files Generated (in [output_dir]/tailored_resumes/):
  [✓] Markdown:      [Name]_[Role]_Resume.md
  [✓] Plain Text:    [Name]_[Role]_Resume_ATS.txt
  [✓/✗] PDF:        [Name]_[Role]_Resume.pdf  (page count, size)
  [✓/✗] Word Doc:   [Name]_[Role]_Resume.docx
  [✓] Cover Letter:  [Name]_[Role]_Cover_Letter.md
  [✓] Gap Report:    [Name]_[Role]_Gap_Coach.md

Top 3 Changes Made (Original → Rewritten):
  1. [from metadata TOP CHANGES block]
  2. [from metadata TOP CHANGES block]
  3. [from metadata TOP CHANGES block]

Implied Keywords to Consider Adding:
  [from jd_analysis.implied_keywords]

Gaps Noted:
  [first 3–5 from GAPS block]

Application saved to version registry.

Tip: For ATS portals (Workday, Greenhouse, Lever, Taleo), paste the .txt
into the resume field and attach the .pdf. For email/LinkedIn, use the .pdf or .docx.
```
