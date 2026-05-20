---
name: workbook
description: Convert study material in the current project folder into a validated YAML quiz pack (.yml) for the StudyAndGame desktop app. Accepts md, txt, pdf, docx, png, jpg as source. Use when the user asks to make a quiz, study pack, problem set, workbook, or 문제집 from a file. Output is a single .yml file the user drops onto StudyAndGame.
---

# Workbook — multi-format study material → StudyAndGame YAML

You convert study material in the user's current project folder into a structured quiz YAML. The companion desktop app (`studyandgame` repo) intentionally does not call any LLM — all content authoring happens here, in Claude Code, with this plugin.

**No hallucination.** Every answer must be quotable from the source. If a fact is unclear, drop the question rather than invent one.

---

## 1. Ingesting the source (the multi-format part)

The user gives you a path inside the current project folder. Pick a strategy by extension:

| Ext | Strategy | Notes |
|-----|----------|-------|
| `.md`, `.txt`, `.markdown` | `Read` tool directly | Plain text — no conversion needed. |
| `.pdf` | `Read` tool directly | Claude Code's `Read` supports PDF natively, multimodal. For PDFs > 10 pages pass `pages: "1-10"` and iterate. |
| `.png`, `.jpg`, `.jpeg`, `.webp` | `Read` tool directly | Claude Code's `Read` ingests images multimodally — built-in vision replaces external OCR/VLM. |
| `.docx` | `Bash: pandoc <file> -o <tmp>.md` then `Read` the .md | See §1.1 below for fallbacks. |
| `.pptx` | `Bash: pandoc <file> -o <tmp>.md` (slide-by-slide) | Same fallback chain as docx. |
| `.html`, `.htm` | `Read` directly (text) or `pandoc -f html -t markdown` | Stripping HTML chrome first usually improves quality. |
| `.epub` | `Bash: pandoc <file> -o <tmp>.md` | Long → chunk by chapter. |
| URL | `WebFetch` | Only if user explicitly provides a URL — default is local files. |

### 1.1 docx / pptx conversion chain

Try in order, stop at first success:

1. **pandoc (best — preserves headings/lists/tables):**
   ```bash
   pandoc "<file>" -o "/tmp/workbook-<hash>.md" --wrap=none
   ```
   If pandoc is missing the user sees `pandoc: command not found`. Suggest install:
   - macOS: `brew install pandoc`
   - Windows: `winget install JohnMacFarlane.Pandoc` or `choco install pandoc`
   - Linux: `apt install pandoc` / `dnf install pandoc`

2. **Python fallback (if pandoc unavailable):**
   ```bash
   python -c "import docx; print('\n'.join(p.text for p in docx.Document('<file>').paragraphs))"
   ```
   Requires `pip install python-docx`. Tables and images are lost — flag this.

3. **PowerShell pure (Windows, zero install):**
   docx is a zip with `word/document.xml`. Extract and strip XML. Quality is rough but works offline.
   ```powershell
   Add-Type -AssemblyName System.IO.Compression.FileSystem
   $zip = [IO.Compression.ZipFile]::OpenRead('<file>')
   $entry = $zip.GetEntry('word/document.xml')
   $reader = New-Object IO.StreamReader($entry.Open())
   $xml = $reader.ReadToEnd(); $reader.Close(); $zip.Dispose()
   ([xml]$xml).document.body.InnerText
   ```

4. **Ask the user** to export as `.pdf` or `.md` and retry. Last resort.

### 1.2 Scanned / image-based PDF

If `Read` on a PDF returns garbled text or no text (image-only PDF), it is a scan. Two options:

1. **Render pages as images then `Read` them** — Claude's vision OCRs the page content:
   ```bash
   # If pdftoppm is available (poppler-utils):
   pdftoppm -r 200 -png "<file>" "/tmp/workbook-page"
   ```
   Then `Read` each `/tmp/workbook-page-*.png`. Slower, but very accurate for typed text.

2. **Suggest the user run external OCR first** (`ocrmypdf input.pdf output.pdf`) and retry — produces a searchable PDF you can `Read` directly.

Do not try to install OCR engines. The "better method" is Claude Code's own multimodal Read.

### 1.3 Folder ingestion

If the user gives a directory:
1. `Glob` for supported extensions in the directory.
2. Confirm with the user before processing > 5 files (cost / time).
3. Generate one pack per source file, OR one merged pack — ask which.

---

## 2. Output YAML schema (strict — StudyAndGame rejects deviations)

```yaml
meta:
  title: <string, 1-80 chars, required>           # Korean chars count as 1 each
  version: <semver, e.g. 0.1.0, required>
  default_time: <int seconds, optional, default 30>
  set_time: <int seconds | null, optional>
  tags: [<lowercase-with-hyphens>, ...]            # optional

questions:                                          # required, length >= 1
  - type: mcq
    q: '<question text>'                            # ALWAYS single-quote Korean/CJK
    choices:
      - '<choice 1>'                                # ALWAYS single-quote each
      - '<choice 2>'                                # 2-6 choices total
    answer: <0-based int, 0 <= answer < len(choices)>
    explanation: <why correct + source quote>
    time: <int seconds, optional>
    tags: [<tag>, ...]

  - type: ox
    q: '<proposition>'
    answer: <true | false>                          # boolean, NOT string
    explanation: <explanation>
    time: <int seconds, optional>
    tags: [<tag>, ...]
```

**Only `mcq` and `ox` are supported.** `typing` / `fill` / `match` are rejected with `ERR_UNKNOWN_TYPE`.

---

## 3. Absolute rules (violations auto-rejected by StudyAndGame)

| # | Rule | App error |
|---|------|-----------|
| 1 | `type` is `mcq` or `ox` only | `ERR_UNKNOWN_TYPE` |
| 2 | `mcq.answer` is 0-based int in range | `ERR_MCQ_ANSWER_OUT_OF_RANGE` |
| 3 | `mcq.choices` has 2-6 string items | `ERR_YAML_PARSE` |
| 4 | `ox.answer` is boolean (not `"true"`) | `ERR_YAML_PARSE` |
| 5 | `meta.title` is 1-80 chars, non-empty | `ERR_META_TITLE_MISSING` |
| 6 | `questions` is non-empty | `ERR_NO_QUESTIONS` |
| 7 | Field names are English lowercase | parse fail |
| 8 | No YAML custom tags (`!!js/function`) | blocked by safe schema |
| 9 | File size < 5 MB | `ERR_FILE_TOO_LARGE` |
| 10 | Extension `.yml` or `.yaml` | `ERR_UNSUPPORTED_EXT` |

---

## 4. ★ Quoting rule for Korean content (critical — external LLMs frequently miss this)

**Wrap every `q:` value and every `choices:` item in single quotes `'...'`.** Idempotent and safe.

```yaml
- type: mcq
  q: 'MergeTree part `202601_5_12_3_8`의 다섯 숫자는?'   # backticks fine
  choices:
    - 'Enum8: 256 / Enum16: 65536'      # colon would break plain scalar
    - '**항상** batch processor 사용'    # asterisk would be YAML alias
    - '1000 / -1'                        # pure digits parse as int
    - "''too many parts'' 에러"          # internal ' escaped as ''
  answer: 0
```

Cases that break if emitted as plain scalar (all rejected):
- Choice with colon: `- Enum8: 256` → mapping → indentation error
- Choice starting with `*`: `- *항상` → YAML alias
- Pure number/bool choice: `- 1000`, `- true` → wrong type
- `q:` with embedded double-quotes: `q: "foo" 라는 에러` → mismatch

`explanation:` may use literal block `|` (correct indentation required).

---

## 5. Quality gates

- **Source citation**: every correct answer must be directly supportable from the source text. If you cannot quote it, do not write the question.
- **Unambiguously correct**: prefer "is correct" over "is more correct".
- **Plausible distractors**: wrong choices must be wrong but learn-worthy. No absurd fillers.
- **Answer-index spread**: don't let one index dominate across the pack.
- **Explanation required**: ≥ 1 line, source quote recommended.
- **Tags**: lowercase English, 3-5 per pack, 1-3 per question.

---

## 6. Self-validation (run before emitting each question)

- [ ] type is `mcq` or `ox`
- [ ] mcq → answer is int and in `[0, len(choices))`
- [ ] mcq → 2-6 choices
- [ ] ox → answer is boolean
- [ ] q is non-empty, one clear sentence
- [ ] Answer is quotable from source
- [ ] explanation has ≥ 1 line
- [ ] mcq distractors are plausible
- [ ] Across the pack: answer indices are spread

If a question fails the check, **skip it** — quality over count.

---

## 7. Response format

1. One-line summary: e.g. `"clickhouse-mergetree.yml — mcq 5 + ox 3 = 8 questions (from clickhouse-docs.pdf, pages 12-28)"`
2. Suggested filename: kebab-case + `.yml`
3. **YAML body inside a single ```yaml fenced block.** Nothing else.

The user copies the YAML body verbatim into the suggested `.yml` filename and drops it onto StudyAndGame.

---

## 8. Time heuristics

| Question kind | Recommended `time` |
|---------------|-------------------|
| Simple OX | 15 s |
| Short mcq (1-2 lines) | 20-30 s |
| Long mcq (with code/SQL) | 45-60 s |
| Reasoning/calculation mcq | 60-90 s |

`meta.set_time` ≈ `default_time × len(questions) × 0.8` (mild time pressure).

---

## 9. Korean content policy

- `q`, `choices`, `explanation` may be Korean.
- Code, SQL, commands, proper nouns, abbreviations stay in the original language (do not translate).
- Tags are English lowercase only.

---

## 10. Worked example (end-to-end)

User: `/workbook ./docs/clickhouse-mergetree.pdf "ClickHouse MergeTree" mcq 3 + ox 2`

You:
1. `Read` the PDF.
2. Extract key facts: parts + primary key sort, background merge, sparse index, granule = 8192.
3. Draft 3 mcq + 2 ox, run §6 checklist on each.
4. Emit response:

`clickhouse-mergetree.yml — mcq 3 + ox 2 = 5 questions (from clickhouse-mergetree.pdf)`

```yaml
meta:
  title: ClickHouse MergeTree
  version: 0.1.0
  default_time: 25
  set_time: 100
  tags: [clickhouse, mergetree, oss-db]

questions:
  - type: mcq
    q: 'MergeTree 엔진이 part 내부 정렬에 사용하는 기준은?'
    choices:
      - 'Hash sort'
      - 'ORDER BY로 지정된 primary key'
      - 'B+Tree 인덱스'
      - '정렬 없음'
    answer: 1
    explanation: |
      docs — "store data in parts and sort it by primary key".
      MergeTree는 part 내부를 primary key 기준으로 정렬해 저장한다.
    tags: [mergetree, indexing]

  - type: mcq
    q: 'ClickHouse primary index 자료구조의 핵심 특성은?'
    choices:
      - 'B+Tree'
      - 'sparse index — granule 단위로 mark'
      - 'Bloom filter only'
      - 'LSM tree'
    answer: 1
    explanation: |
      docs — "sparse indexing with granules of size index_granularity ... one mark per granule".
    tags: [mergetree, index]

  - type: mcq
    q: 'index_granularity의 기본값은?'
    choices:
      - '1024'
      - '4096'
      - '8192'
      - '16384'
    answer: 2
    explanation: 'docs — "granules of size index_granularity (default 8192)"'
    time: 15
    tags: [mergetree, granule]

  - type: ox
    q: 'MergeTree의 part는 background에서 자동으로 merge된다.'
    answer: true
    explanation: 'docs — "Parts are merged in background"'
    time: 15
    tags: [mergetree, merge]

  - type: ox
    q: 'MergeTree는 데이터를 정렬하지 않고 그대로 저장한다.'
    answer: false
    explanation: |
      반대다. primary key 기준으로 정렬해 저장한다.
      이 sparse 정렬이 인덱스 기반 빠른 range scan을 가능하게 한다.
    time: 15
    tags: [mergetree]
```

---

## 11. Common mistakes to avoid

1. ❌ `answer: "1"` (string) — must be int
2. ❌ `type: typing` — unsupported
3. ❌ Plain scalar Korean with special chars — see §4 Quoting rule
4. ❌ All answers at the same index
5. ❌ Korean field names (`질문:`, `정답:`)
6. ❌ `meta.title` over 80 chars (Korean chars count)
7. ❌ Trying to install OCR tools — use Claude's built-in multimodal Read first
8. ❌ Processing > 5 files in a folder without user confirmation
