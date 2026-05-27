---
name: generate-resume-doc
description: >
  Converts a Markdown resume into multiple output formats: clean .md, plain-text ATS .txt,
  and .docx Word document (requires pandoc). Use this to regenerate output files from an
  already-tailored resume, or to convert an existing resume to ATS-safe formats.
  Invoke with: /generate-resume-doc <resume_md_path> [output_dir]
user-invocable: true
argument-hint: "<resume_md_path> [output_dir]"
context: fork
agent: general-purpose
allowed-tools: Read, Write, Bash
effort: medium
---

# Resume Document Generator

Convert the Markdown resume at `$1` into multiple submission-ready formats.
Output directory defaults to the same folder as the source file, or use `$2` if provided.

## Steps

1. Read the file at `$1`
2. Determine output directory from `$2` or same dir as `$1`
3. Create `tailored_resumes/` subdirectory in output dir if it doesn't exist
4. Delegate to `resume-document-generator` subagent with file content and paths

## Pandoc Check

Run: `which pandoc`
- If found: generate .docx
- If not found: skip .docx and show install instructions

## Display final output summary from the document generator agent.

If `$1` is missing or file not found:
> "Usage: /generate-resume-doc <resume_md_path> [output_dir]"
> "File not found: [path]"
