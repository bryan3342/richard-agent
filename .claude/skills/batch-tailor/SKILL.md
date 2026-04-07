---
name: batch-tailor
description: >
  Runs the full ATS resume tailoring pipeline against multiple job descriptions in a folder,
  producing a tailored resume for each and a comparison table of scores, grades, and top gaps.
  Use this when applying to multiple roles and you want to optimize for all of them from a
  single resume. Invoke with: /batch-tailor <resume_path> <jd_folder> [output_dir]
user-invocable: true
argument-hint: "<resume_path> <jd_folder> [output_dir]"
context: fork
agent: general-purpose
allowed-tools: Read, Write, Bash, Glob, Grep, Agent
effort: high
---

# ATS Resume Tailor — Batch Mode

You are the batch orchestrator for the ATS Resume Tailor pipeline. When invoked, you run the
full tailoring pipeline against every job description file in a folder and produce a comparison
summary across all applications.

## Arguments

Parse `$ARGUMENTS` to extract:
- `$1` — Path to the original resume file (required)
- `$2` — Path to a folder containing `.txt` or `.md` job description files (required)
- `$3` — Output directory (optional, defaults to same directory as resume)

If arguments are missing, ask the user:
> "Please provide your resume path and a folder containing job description files.\n> Usage: /batch-tailor <resume_path> <jd_folder> [output_dir]"

## Pre-Flight Checks

1. Confirm resume file exists and is readable
2. Confirm JD folder exists and contains at least one `.txt` or `.md` file
3. List all JD files found and show the user:
   ```
   Found [N] job descriptions in [jd_folder]:
     - company_a_role.txt
     - company_b_role.txt
     ...
   Starting batch tailoring — this will take approximately [N × 2-3] minutes.
   ```
4. Ask user to confirm before proceeding if N > 5 (long run)

## Batch Pipeline

For each JD file found, run the full `/tailor-resume` pipeline by delegating to the
`tailor-resume` subskill logic directly (do not re-invoke the skill — reproduce the steps):

### Per-JD Steps

1. **Analyze JD** — delegate to `jd-analyzer` agent
2. **Tailor Resume** — delegate to `resume-tailor` agent with original resume + JD analysis
3. **Score** — delegate to `ats-scorer` agent
4. **Re-tailor if score < 80** — inject `critical_issues` and `quick_wins` from score report
   into a second `resume-tailor` pass (max 2 passes per JD)
5. **Generate Documents** — delegate to `resume-document-generator` agent
   - Output to: `[output_dir]/tailored_resumes/[company_name]/`

Show progress after each JD:
```
[2/4] ✓ Stripe — Solutions Engineer — Score: 85/100 (B) — Files saved
```

## Comparison Table Output

After all JDs are processed, generate a Markdown comparison table:

```markdown
# Batch Tailoring Results

| # | Company | Role | Score | Grade | Top Gap | Output Folder |
|---|---------|------|-------|-------|---------|---------------|
| 1 | Fonoa   | Solutions Engineer | 90/100 | A | Tax domain exp. | tailored_resumes/fonoa/ |
| 2 | Stripe  | Solutions Engineer | 85/100 | B | Payment systems | tailored_resumes/stripe/ |
| 3 | Vercel  | Developer Relations | 75/100 | C | Community mgmt | tailored_resumes/vercel/ |

**Best match:** [Company] ([score]/100)
**Recommended application order:** [ranked by score, highest first]
```

Save this table to `[output_dir]/batch_summary.md`.

## Version Registry

After all runs complete, append each result to `[output_dir]/version_registry.json`.
If the file does not exist, create it as an empty JSON array `[]` first.

Each entry:
```json
{
  "date": "[today's date]",
  "jd_file": "[filename]",
  "jd_company": "[extracted company name or filename stem]",
  "role": "[role_title from JD analysis]",
  "score": 90,
  "grade": "A",
  "passes": 1,
  "top_gap": "[first item from gaps list]",
  "files": {
    "md": "[path]",
    "txt": "[path]"
  }
}
```

## Final Summary to User

```
BATCH TAILORING COMPLETE
=========================
Resume:      [resume_path]
JDs Processed: [N]
Time:        [elapsed estimate]

Results:
  A grade (90-100): [count] roles
  B grade (80-89):  [count] roles
  C grade (70-79):  [count] roles
  Below C:          [count] roles

Top match: [company] — [role] — [score]/100
Summary saved: [output_dir]/batch_summary.md

Tip: Apply to A and B grade roles first. For C grade roles,
review the gap coaching output before submitting.
```
