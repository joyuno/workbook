---
description: Convert a study material file (md / txt / pdf / docx / image) into a StudyAndGame YAML quiz pack. Args — file path and optional title/counts.
---

You have been invoked through the `/workbook` slash command. The user wants a StudyAndGame-compatible `.yml` quiz pack generated from a file in the current project folder.

## What to do

1. Invoke the `workbook` skill — it carries the full ingestion strategy, schema, quoting rules, and quality gates.
2. Parse `$ARGUMENTS` for:
   - **Source path** (required): the file or directory to convert. Resolve relative to the user's current working directory.
   - **Title** (optional): explicit pack title, otherwise infer from filename or document heading.
   - **Counts** (optional): e.g. `mcq 5 + ox 3`. Default mcq 4-6 + ox 2-3 if absent.
   - **Figures flag** (optional): `figures` / `그림`. Forces 3D structure figures
     on for on-topic crystal-structure questions, skipping the prompt in step 4.5.
3. Pick the ingestion strategy by extension (see SKILL §1 table):
   - `.md` / `.txt` / `.pdf` / image → `Read` directly.
   - `.docx` / `.pptx` → `pandoc` → `Read`. Fallback chain if pandoc missing.
   - Folder → `Glob`, confirm before > 5 files.
4. Generate YAML following the skill's response format:
   - One-line summary (include source filename + page range if PDF)
   - Suggested filename (kebab-case, `.yml`)
   - YAML body in a single ```yaml fenced block, nothing else.
4.5. **Structure figures (user choice — see SKILL §14):** after drafting, check
   whether any question is *about* a crystal structure the app can render (SC /
   BCC / FCC / diamond / HCP / zinc blende / rock salt / CsCl / fluorite).
   - None → emit normally, say nothing about figures.
   - Some, and the user passed `figures` / `그림` → attach `figure:` to each
     on-topic structure question, then emit.
   - Some, and no flag → ask once: `"구조 그림(3D)을 N개 문항에 붙일까요?
     [예/아니오]"`. Attach `figure:` only on yes. Never invent a figure name
     outside the supported list — the app silently ignores unknown values.

## What NOT to do

- Do not invent facts not in the source. Drop unsupportable questions.
- Do not write the .yml file to disk — the user copies the YAML body into their own file for the StudyAndGame app.
- Do not propose adding `typing` / `fill` / `match` types. Only `mcq` and `ox`.
- Do not install OCR engines or VLM models. Use Claude Code's built-in multimodal `Read` first.

## User input

$ARGUMENTS
