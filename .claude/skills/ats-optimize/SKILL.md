---
name: ats-optimize
description: >
  Quick ATS optimization audit skill. Takes a resume file and optionally a JD analysis JSON,
  and returns an ATS score plus a prioritized list of improvements. Use this to spot-check
  a resume you've already written or to get a quick audit without running the full pipeline.
  Invoke with: /ats-optimize <resume_path> [jd_analysis_json_path]
user-invocable: true
argument-hint: "<resume_path> [jd_analysis_path]"
context: fork
agent: general-purpose
allowed-tools: Read, Grep, Agent
effort: medium
---

# ATS Resume Optimizer

Perform a rapid ATS audit on the resume at `$1`. If `$2` is provided, use the JD analysis
JSON for keyword-aware scoring. Otherwise, perform a general ATS formatting audit.

## Steps

1. Read the resume at `$1`
2. If `$2` is provided, read the JD analysis JSON
3. Delegate to `ats-scorer` subagent with the resume content and analysis (if available)
4. Parse and display the scoring report

## Output Format

```
ATS AUDIT REPORT
================
File: [resume_path]
Score: [score]/100 — [grade]

SCORE BREAKDOWN
---------------
Keyword Match     [██████████░░░░] [score]/40
Required Skills   [████████░░░░░░] [score]/25
Action Verbs      [████████░░░░░░] [score]/10
Quantification    [██████░░░░░░░░] [score]/15
Formatting        [██████████░░░░] [score]/10

CRITICAL ISSUES (fix these first)
-----------------------------------
[critical_issues as numbered list]

QUICK WINS (easy improvements)
--------------------------------
[quick_wins as numbered list]

MISSING KEYWORDS
----------------
[missing keywords from keyword_match.missing]

WEAK BULLET OPENERS TO REWRITE
--------------------------------
[weak_openers_found]

[summary paragraph]

--
Run /tailor-resume to automatically fix all these issues.
```

If no JD analysis is provided, add a note:
> "Tip: For keyword-specific scoring, run /analyze-jd on your target job description first,
> then re-run: /ats-optimize [resume] [jd_analysis.json]"
