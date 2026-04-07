---
name: analyze-jd
description: >
  Standalone skill to analyze a job description and display a human-readable breakdown
  of ATS keywords, required skills, key metrics patterns, and tailoring recommendations.
  Use this when you want to inspect a JD before tailoring, or to understand what a role
  is really looking for. Invoke with: /analyze-jd <jd_path>
user-invocable: true
argument-hint: "<job_description_path>"
context: fork
agent: general-purpose
allowed-tools: Read, Glob, Agent
effort: medium
---

# Job Description Analyzer

Analyze the job description provided in `$ARGUMENTS` and produce a clear, human-readable
breakdown that a job seeker can use to understand how to position themselves.

## Steps

1. Validate the file at `$1` exists and read it
2. Delegate to the `jd-analyzer` subagent for structured analysis
3. Format and display the results in a clean, readable report

## Output Format

Display the analysis as follows:

```
JOB DESCRIPTION ANALYSIS
=========================
Role:       [role_title]
Level:      [seniority_level]
Department: [department]

MUST-HAVE REQUIREMENTS
-----------------------
Skills:     [required_skills as comma-separated list]
Experience: [years_experience]
Education:  [education requirement]

PREFERRED QUALIFICATIONS
------------------------
[preferred_skills as bullet list]

TOP ATS KEYWORDS (embed these in your resume)
----------------------------------------------
[ats_keywords — numbered list, top 10]

KEY ACTION VERBS (use these to start bullet points)
----------------------------------------------------
[action_verbs as comma-separated list]

METRICS & QUANTIFIER PATTERNS (show impact with these)
-------------------------------------------------------
[metrics patterns as bullet list]

TECHNICAL SKILLS TO HIGHLIGHT
------------------------------
[hard_skills as bullet list]

SOFT SKILLS VALUED
------------------
[soft_skills as comma-separated list]

COMPANY CULTURE SIGNALS
-----------------------
[cultural_signals as bullet list]

RECOMMENDED RESUME SECTION ORDER
----------------------------------
[recommended_resume_sections_order numbered]

TAILORING THEMES (use these in your summary)
--------------------------------------------
[suggested_resume_summary_themes as numbered list]

THINGS TO AVOID
---------------
[ats_red_flags_to_avoid as bullet list]
```

If the file does not exist, respond:
> "File not found: [path]. Please provide the correct path to your job description file."
