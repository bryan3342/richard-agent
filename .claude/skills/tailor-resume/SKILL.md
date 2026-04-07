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

You are the orchestrator for the ATS Resume Tailor pipeline. When invoked, you run a
multi-agent workflow to produce a fully tailored, ATS-optimized resume, cover letter,
and gap coaching report.

## Arguments

Parse `$ARGUMENTS` to extract:
- `$1` — Path to the original resume file (required)
- `$2` — Path to the job description file OR a URL starting with `http` (required)
- `$3` — Output directory (optional, defaults to same directory as original resume)

If arguments are missing, ask the user:
> "Please provide paths to both your resume and the job description.\n> Usage: /tailor-resume <resume_path> <jd_path_or_url> [output_dir]"

---

## Step 0 — User Preference Intake

Before starting the pipeline, ask the user these three questions. Accept free-text answers.
If the user skips or says "no preference", proceed with defaults.

```
Before I start tailoring, a few quick questions (press Enter to skip any):

1. What aspects of your experience should I emphasize most?
   (e.g., "leadership over individual contribution", "AI/ML work", "startup experience")

2. Are there any roles, skills, or content you want excluded or downplayed?
   (e.g., "don't emphasize the internship at X", "remove the unrelated freelance work")

3. What tone should the resume have?
   Options: formal | startup | technical | balanced (default: balanced)
```

Store the answers as `user_preferences` — they will be passed to the tailoring agent.

---

## Step 1 — Validate Inputs + Fetch JD (Parallel)

Run these two tasks simultaneously (do not wait for one before starting the other):

**Task A — Resume validation:**
- Confirm the resume file exists and is readable
- Detect resume format (Markdown, plain text, or PDF)
- If PDF: inform user that PDF parsing is limited and results may vary; proceed with text extraction
- Read the resume content into memory

**Task B — JD acquisition:**
- If `$2` starts with `http`: use WebFetch to retrieve the page content, then extract the job
  description text (strip navigation, footers, and UI chrome — keep only the JD body)
- If `$2` is a file path: read the file directly
- Confirm the JD content is at least 100 words; if shorter, warn the user:
  > "Warning: JD content is very short ([N] words). Results may be less precise."

**Dynamic routing checks (evaluate after both tasks complete):**
- If JD word count < 200: flag to user, ask if they want to continue or provide a fuller JD
- If resume is already clean Markdown: note this for the tailoring agent (format is optimal)
- If JD has very few extractable keywords (heuristic: fewer than 8 distinct noun phrases): warn
  that scoring will be less precise

Print: `> [1/6] Inputs validated — Resume: [filename], JD: [source]`

---

## Step 2 — Analyze Job Description

Delegate to the `jd-analyzer` subagent:
> "Analyze the job description at [jd_path] and return the full JSON analysis."

Store the JSON analysis result.

Extract and display to user:
```
> [2/6] JD Analysis Complete
>   Role: [role_title] ([seniority_level])
>   Top ATS Keywords: [first 5 ats_keywords]
>   Required Skills: [count] identified
>   Implied Keywords (not in JD): [first 3 implied_keywords]
>   Key Metrics Patterns: [metrics patterns]
```

---

## Step 3 — Tailor Resume

Delegate to the `resume-tailor` subagent with:
- Original resume content (from Step 1)
- JD analysis JSON from Step 2
- User preferences from Step 0 (emphasis, exclusions, tone)
- Instruction to produce tailored resume Markdown

Print: `> [3/6] Resume tailoring in progress...`
Store tailored resume content.

---

## Step 4 — ATS Score & Validate (Targeted Re-Tailoring Loop)

Delegate to the `ats-scorer` subagent with:
- Tailored resume content from Step 3
- JD analysis JSON from Step 2

Parse the score report. Track: `pass_number = 1`, `previous_score = null`.

Display:
```
> [4/6] ATS Score — Pass 1
>   Overall Score: [score]/100 (Grade: [grade])
>   Keyword Match: [score]/40
>   Required Skills: [score]/25
>   Quantification: [score]/15
>   Formatting: [score]/10
>   ATS System Detected: [ats_system]
```

**Re-tailoring loop (automatic, no user prompt required):**

While `score < 80` AND `pass_number < 3`:
1. Check diminishing returns: if `previous_score` exists and `score - previous_score < 5`, stop
   and notify user: "Score improvement stalled at [score]/100 — proceeding with current version."
2. Otherwise, re-run `resume-tailor` with:
   - All original inputs
   - The full score report as additional context (especially `critical_issues` and `quick_wins`)
   - Instruction: "This is re-tailoring pass [N]. Address every item in `critical_issues` first,
     then apply each `quick_win`. Focus changes on the specific bullets and sections listed —
     do not re-rewrite sections that already scored well."
3. Re-run `ats-scorer` on the new version
4. Increment `pass_number`, set `previous_score = score`, update `score`
5. Display: `> [4/6] ATS Score — Pass [N]: [score]/100 (Grade: [grade])`

If score ≥ 88 after Pass 1: skip re-tailoring entirely and print:
> "Score [score]/100 — excellent match. Skipping re-tailoring pass."

---

## Step 5 — Generate Cover Letter

Delegate to the `cover-letter-writer` subagent with:
- Tailored resume content (final version)
- JD analysis JSON from Step 2
- Candidate name (extracted from resume header)
- Company name (extracted from JD analysis or JD text)

Print: `> [5/6] Generating cover letter...`

Save the cover letter Markdown to:
`[output_dir]/tailored_resumes/[CandidateName]_[RoleTitle]_Cover_Letter.md`

---

## Step 5.5 — Gap Coaching Report

Delegate to the `gap-coach` subagent with:
- JD analysis JSON from Step 2
- Gap list from the tailored resume's `<!-- GAPS: -->` comment
- Final ATS score report
- Tailored resume content

Save the gap coaching report to:
`[output_dir]/tailored_resumes/[CandidateName]_[RoleTitle]_Gap_Coach.md`

Print: `> Gap coaching report generated.`

---

## Step 6 — Generate Output Documents

Determine output directory: use `$3` if provided, otherwise use same directory as original resume.

Delegate to `resume-document-generator` subagent with:
- Final tailored resume Markdown
- Output directory path
- Candidate name and role title

Print final summary from the document generator.

---

## Step 7 — Update Version Registry

After all files are generated, append a record to `[output_dir]/version_registry.json`.
If the file does not exist, create it as an empty JSON array `[]` first. Then append:

```json
{
  "date": "[today's date]",
  "jd_source": "[file path or URL]",
  "jd_company": "[company name if extractable, else null]",
  "role": "[role_title from JD analysis]",
  "score": [final_score],
  "grade": "[grade]",
  "passes": [pass_number],
  "ats_system": "[detected ats system]",
  "top_gap": "[first gap from GAPS comment, or null]",
  "files": {
    "md": "[path to .md file]",
    "txt": "[path to .txt file]",
    "cover_letter": "[path to cover letter .md]",
    "gap_coach": "[path to gap coaching .md]"
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
ATS System:       [detected system]

Files Generated:
  [✓] Markdown:      [CandidateName]_[RoleTitle]_Resume.md
  [✓] Plain Text:    [CandidateName]_[RoleTitle]_Resume_ATS.txt
  [✓/✗] Word Doc:   [status]
  [✓] Cover Letter:  [CandidateName]_[RoleTitle]_Cover_Letter.md
  [✓] Gap Report:    [CandidateName]_[RoleTitle]_Gap_Coach.md

Top 3 Changes Made (Original → Rewritten):
  1. [from metadata TOP CHANGES block]
  2. [from metadata TOP CHANGES block]
  3. [from metadata TOP CHANGES block]

Implied Keywords to Consider Adding:
  [list from jd_analysis.implied_keywords — surfaced as suggestions only]

Gaps Noted (from original resume):
  [any gaps identified by resume-tailor]

Application saved to version registry.

Tip: For best results, paste the plain-text (.txt) version directly
into ATS portals (Workday, Greenhouse, Lever, Taleo), and attach
the .docx for email submissions. Install pandoc first: brew install pandoc
```
