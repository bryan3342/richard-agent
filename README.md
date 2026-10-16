# Richard Agent

## Description

A Claude Code skill and subagent system for analyzing job descriptions, tailoring an existing resume, estimating ATS coverage, and producing application documents. Its prompts explicitly instruct the agents to preserve source facts and flag gaps rather than inventing qualifications.

## Dependencies

Claude Code with support for local skills and subagents. Pandoc is optional for Word and PDF conversion. PDF and DOCX source extraction also depends on the tools available to Claude Code.

## Technologies

Markdown, YAML frontmatter, Claude Code skills and agents, JSON intermediate reports, and optional Pandoc document conversion.

## Setup

```bash
git clone https://github.com/bryan3342/richard-agent.git
cd richard-agent
claude
```

Run Claude Code from the repository so it discovers `.claude/skills/` and `.claude/agents/`. Use Markdown or plain text for the most reliable resume input. Install Pandoc only if DOCX or PDF output is needed.

## Features

- Job-description keyword, skill, action-verb, metric, and seniority analysis.
- Resume tailoring that preserves the original experience and structure.
- ATS-style scoring across keywords, skills, verbs, quantification, and formatting.
- Markdown and plain-text output, with document conversion when supported.
- Batch processing for folders of `.txt` or `.md` job descriptions.
- Per-role summaries, score comparison, and a version registry.

The score is a rubric implemented by the prompts, not a result from a hiring platform's proprietary ATS. Every generated resume should be reviewed before submission.

## Commands to run

Inside Claude Code:

```text
/analyze-jd path/to/job_description.txt
/ats-optimize path/to/resume.md path/to/jd_analysis.json
/tailor-resume path/to/resume.md path/to/job_description.txt
/generate-resume-doc path/to/tailored_resume.md path/to/output_dir
/batch-tailor path/to/resume.md path/to/jd_folder path/to/output_dir
```

There is no conventional build or test suite. The commands invoke prompt-defined workflows and write results to the chosen output location.

## Contributors

- [Bryan Mejia (@bryan3342)](https://github.com/bryan3342), credited in Git history.
