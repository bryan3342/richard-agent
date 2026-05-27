# Richard Agent — ATS Resume Tailor

An AI-powered resume tailoring system built on Claude Code agents and skills. Given an original
resume and a job description, it analyzes the JD for ATS keywords and metrics, rewrites the
resume to maximize ATS match score, and outputs submission-ready documents.

---

## What This Does

1. **Analyzes** the job description: extracts ATS keywords, required skills, action verbs,
   metrics patterns, seniority signals, and cultural indicators.

2. **Tailors** the original resume: rewrites bullet points with JD keywords, restructures
   sections for maximum relevance, surfaces quantified impact, and mirrors JD vocabulary.

3. **Scores** the result: runs a simulated ATS audit across keyword coverage, skill presence,
   action verb quality, quantification density, and formatting compliance.

4. **Generates** output documents: clean Markdown, plain-text ATS-safe `.txt` (for portal
   copy-paste), and `.docx` Word document (requires pandoc).

---

## Quick Start

```bash
# Full pipeline — analyze JD, tailor resume, score, and generate docs
/tailor-resume path/to/my_resume.md path/to/job_description.txt

# Inspect a job description only
/analyze-jd path/to/job_description.txt

# Audit an existing resume for ATS issues
/ats-optimize path/to/my_resume.md

# Audit with JD-specific keyword scoring
/ats-optimize path/to/my_resume.md path/to/jd_analysis.json

# Convert an already-tailored resume to multiple formats
/generate-resume-doc path/to/tailored_resume.md path/to/output_dir
```

---

## Architecture

```
.claude/
├── agents/                          # 4 agents — only sub-agents called >1x or returning structured JSON
│   ├── jd-analyzer.md               # Extracts keywords, skills, metrics from JD → JSON
│   ├── resume-tailor.md             # Rewrites resume preserving original structure → Markdown
│   ├── ats-scorer.md                # Scores tailored resume 0–100 → JSON report
│   └── resume-document-generator.md # Converts Markdown to .md / .txt / .pdf / .docx
│
└── skills/
    ├── tailor-resume/SKILL.md       # /tailor-resume — main orchestration (cover letter + gap coach inlined)
    ├── analyze-jd/SKILL.md          # /analyze-jd — standalone JD analysis
    ├── ats-optimize/SKILL.md        # /ats-optimize — standalone ATS audit
    ├── generate-resume-doc/SKILL.md # /generate-resume-doc — document conversion
    └── batch-tailor/SKILL.md        # /batch-tailor — multi-JD batch mode
```

### Agent Responsibilities

| Agent | Input | Output | Model |
|-------|-------|--------|-------|
| `jd-analyzer` | JD file path or text | JSON analysis | sonnet |
| `resume-tailor` | Resume + JD analysis (+ optional score report) | Tailored resume Markdown | opus |
| `ats-scorer` | Resume + JD analysis | Scoring JSON report | sonnet |
| `resume-document-generator` | Resume Markdown + output dir | .md / .txt / .pdf / .docx files | sonnet |

**Note:** Cover-letter generation and gap coaching are inlined into `tailor-resume/SKILL.md`
(not separate sub-agents) — they each ran exactly once per pipeline, so the extra Agent()
hop was pure latency overhead. PDF generation lives only in `resume-document-generator`
(previously duplicated in a now-deleted `pdf-generator.md`).

### Skill Responsibilities

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `/tailor-resume` | Main entry point | Orchestrates full pipeline |
| `/analyze-jd` | Standalone audit | Human-readable JD breakdown |
| `/ats-optimize` | Spot-check | Quick ATS score without full pipeline |
| `/generate-resume-doc` | Post-tailoring | Convert existing resume to output formats |

---

## Pipeline Flow

```
User: /tailor-resume resume.md jd.txt
         │
         ▼
[1] Validate inputs (file existence, format detection)
         │
         ▼
[2] jd-analyzer → JD analysis JSON
    - ATS keywords, required skills, action verbs
    - Metrics patterns, seniority level, section order
         │
         ▼
[3] resume-tailor → tailored resume Markdown
    - Keyword-embedded bullet rewrites
    - Skills section restructured
    - Summary rewritten to match role
         │
         ▼
[4] ats-scorer → score report (0-100)
    - If score < 70: optional re-tailoring pass
         │
         ▼
[5] resume-document-generator → output files
    - [Name]_[Role]_Resume.md
    - [Name]_[Role]_Resume_ATS.txt
    - [Name]_[Role]_Resume.docx  (if pandoc installed)
```

---

## Supported Resume Input Formats

| Format | Support |
|--------|---------|
| Markdown (`.md`) | Full support |
| Plain text (`.txt`) | Full support |
| PDF (`.pdf`) | Partial — text extraction only, formatting lost |
| Word (`.docx`) | Partial — requires pandoc for text extraction |

For best results, use a Markdown or plain-text resume as input.

---

## ATS Scoring Rubric

| Category | Max Points | What It Measures |
|----------|-----------|-----------------|
| Keyword Match | 40 | Exact JD keyword phrases present in resume |
| Required Skills | 25 | Must-have skills from JD covered |
| Action Verbs | 10 | Strong verb openers; JD verbs mirrored |
| Quantification | 15 | Bullets with concrete numbers/metrics |
| Formatting | 10 | ATS-safe structure, no tables/columns/graphics |
| **Total** | **100** | |

**Grade Scale:** A (90-100), B (80-89), C (70-79), D (60-69), F (<60)

---

## Output Files

All output goes to a `tailored_resumes/` subdirectory of your chosen output dir:

```
tailored_resumes/
├── Bryan_Mejia_Senior_Engineer_Resume.md      # Clean Markdown
├── Bryan_Mejia_Senior_Engineer_Resume_ATS.txt # Plain text for ATS portals
└── Bryan_Mejia_Senior_Engineer_Resume.docx    # Word doc (needs pandoc)
```

### When to Use Each Format

- **ATS portal** (Workday, Greenhouse, Lever, Taleo): use the `.txt` or `.docx`
- **Email to recruiter**: use `.docx`
- **LinkedIn Easy Apply**: upload `.docx`
- **Version control / editing**: use `.md`

---

## Optional: Install pandoc for Word Output

```bash
brew install pandoc       # macOS
apt install pandoc        # Ubuntu/Debian
winget install pandoc     # Windows
```

---

## Notes & Limitations

- **No content fabrication**: The tailoring agents never invent experience, credentials,
  or metrics. They only rewrite what exists using stronger language and JD-aligned vocabulary.
- **Gaps are flagged**: If the original resume is missing something the JD requires, a
  `<!-- GAPS: -->` comment is added to the tailored resume for your awareness.
- **PDF input limitation**: PDF text extraction loses formatting context. For best results,
  maintain a Markdown or `.txt` master resume.
- **Length targets**: 1 page for <5 years experience, 2 pages max for 5+ years.
