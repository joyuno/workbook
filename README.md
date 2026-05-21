# Workbook — Claude Code plugin

Convert study material (`md` / `txt` / `pdf` / `docx` / `png` / `jpg`) in your **current project folder** into validated YAML quiz packs for the [StudyAndGame](https://github.com/joyuno/studyandgame) desktop app.

After you generate a pack here, head to the **studyandgame** repo to install the desktop app and drop the `.yml` onto it.

## Why this is a separate repo

The StudyAndGame desktop app intentionally ships without any LLM dependency — no API keys, no telemetry, no model downloads, install stays under ~60 MB. All content authoring happens externally, here in Claude Code, with this plugin. You pick whichever Claude model you trust; the game just consumes the validated `.yml`.

## What's inside

| Component | Purpose |
|-----------|---------|
| `workbook/skills/workbook/SKILL.md` | Ingestion strategy per file type · schema · quoting rules · quality gates · worked example. Auto-loads when a request matches. |
| `workbook/commands/workbook.md` | `/workbook <path> [title] [counts]` — convert a file or folder into a pack. |
| `workbook/commands/workbook-verify.md` | `/workbook-verify <path.yml>` — dry-run schema check before dropping into the app. |
| `workbook/examples/` | Reference `.yml` packs that pass the StudyAndGame validator. |

## Supported source formats

| Extension | Method | Requirements |
|-----------|--------|--------------|
| `.md`, `.txt`, `.markdown` | Claude Code `Read` (text) | None |
| `.pdf` | Claude Code `Read` (native multimodal) | None — vision is built in |
| `.png`, `.jpg`, `.jpeg`, `.webp` | Claude Code `Read` (vision) | None — no external OCR needed |
| `.docx`, `.pptx` | `pandoc` → `Read` | `pandoc` on PATH (preferred) or `python-docx` fallback |
| `.html`, `.htm` | `Read` or `pandoc -f html -t markdown` | Optional pandoc |
| `.epub` | `pandoc` → `Read` | `pandoc` on PATH |
| Scanned image-only PDF | `pdftoppm` → render pages → `Read` images | `poppler-utils` (or run `ocrmypdf` first) |

The plugin's "better method" for non-text formats is **Claude Code's built-in multimodal Read** — you don't need to install OCR engines or VLM models for the common cases (PDFs with selectable text, photographed slides, screenshots).

## Installation

Inside Claude Code — two steps (add the marketplace, then install the plugin from it):

```text
/plugin marketplace add joyuno/workbook
/plugin install workbook@joyuno-workbook
```

`/plugin install joyuno/workbook` (single-step) does **not** work — `/plugin install` expects a plugin name from an already-added marketplace, not a repo path. The marketplace registers under the `name` field in `marketplace.json` (`joyuno-workbook`), and the plugin name is `workbook`, so the `@` suffix targets the marketplace.

Restart Claude Code (or reload the plugin list) and the `/workbook` and `/workbook-verify` commands appear.

## Quick start

```text
# In any project folder containing your study material:
/workbook ./docs/clickhouse-mergetree.pdf "ClickHouse MergeTree" mcq 5 + ox 3
```

Claude reads the PDF, then emits:
- one-line summary
- suggested filename (e.g. `clickhouse-mergetree.yml`)
- the YAML body in a fenced block

Copy the YAML body into the suggested filename, drop the `.yml` onto StudyAndGame, play.

## Verify before drop

```text
/workbook-verify ./clickhouse-mergetree.yml
```

Reports schema issues, missing single-quotes on Korean strings, answer-index skew, etc. — without modifying the file.

## Output schema (StudyAndGame `.yml`)

Only `mcq` and `ox` types are supported.

```yaml
meta:
  title: <1-80 chars>
  version: 0.1.0
  tags: [<lowercase-with-hyphens>, ...]
questions:
  - type: mcq
    q: '<question>'                           # single-quote required for Korean/CJK
    choices: ['<2-6 items>', ...]
    answer: <0-based int>
    explanation: <required>
  - type: ox
    q: '<proposition>'
    answer: <true | false>                    # boolean, not string
    explanation: <required>
```

Full spec lives in [`workbook/skills/workbook/SKILL.md`](workbook/skills/workbook/SKILL.md).

## Korean content tip

Always single-quote `q:` and every `choices:` item when the content is Korean or contains `:` / `*` / pure digits / quotes. External LLMs frequently emit plain scalars that the YAML parser then rejects — the `workbook` skill enforces single-quoting by default.

## License

MIT — see [`LICENSE`](LICENSE).

## See also

- **[studyandgame](https://github.com/joyuno/studyandgame)** — the desktop quiz game that consumes the YAML packs you generate here.
