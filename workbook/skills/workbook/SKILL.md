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
    figure: <one of the supported names>            # optional — app renders a rotatable 3D unit cell (§14)
    # ── Vocab gloss cards (optional, language packs — see §13) ───────
    glossary:                                       # 'word｜reading｜meaning' scalars
      - '<word>｜<reading>｜<meaning>'              # NOT a nested map (parser limit)
    # ── Prerequisite Knowledge Graph fields (optional, see §12) ──────
    concept: '<kebab-case-concept-id>'              # what this question teaches (1 atom)
    prereq: ['<concept-id>', ...]                   # concepts the learner must know first
    bloom: <recall | understand | apply | analyze>  # cognitive level (Bloom's taxonomy)

  - type: ox
    q: '<proposition>'
    answer: <true | false>                          # boolean, NOT string
    explanation: <explanation>
    time: <int seconds, optional>
    tags: [<tag>, ...]
    concept: '<kebab-case-concept-id>'              # see §12
    prereq: ['<concept-id>', ...]
    bloom: <recall | understand | apply | analyze>
```

**Only `mcq` and `ox` are supported.** `typing` / `fill` / `match` are rejected with `ERR_UNKNOWN_TYPE`.

`concept` / `prereq` / `bloom` are **optional** — packs without them still parse,
but a pack that opts in must keep them consistent across its questions (§12).

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
- [ ] `figure` is empty, or one of the §14 supported names — and attached only to a question that is actually *about* that structure

If a question fails the check, **skip it** — quality over count.

---

## 7. Response format

1. One-line summary: e.g. `"clickhouse-mergetree.yml — mcq 5 + ox 3 = 8 questions (from clickhouse-docs.pdf, pages 12-28)"`
2. Suggested filename: kebab-case + `.yml`
3. **YAML body inside a single ```yaml fenced block.** Nothing else.

The user copies the YAML body verbatim into the suggested `.yml` filename and drops it onto StudyAndGame.

**Figure trigger (user choice — see §14).** Default = attach nothing. If the
content has **no** renderable structure, never mention figures. If you detect
N figure-eligible questions and the user did *not* already pass a `figures` /
`그림` flag, **ask once before emitting** — e.g. `"구조 그림(3D)을 N개 문항에
붙일까요? [예/아니오]"` — and add `figure:` lines only if the user opts in.

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
9. ❌ Naming `concept` after the question's *tag* instead of the atom it teaches
    (e.g. `concept: 'mergetree'` is too broad — use `'mergetree-primary-key'`)
10. ❌ Listing `prereq` items that no question in any related pack actually teaches
    — the chain dead-ends. Either teach that prereq first or drop the citation.

---

## 12. ★ Prerequisite Knowledge Graph (PKG) — terminology-first authoring

**Why it exists.** A learner who cannot answer "What is a *MergeTree*?" cannot
benefit from a question about *primary key uniqueness in MergeTree*. The
question is unanswerable not because it is wrong but because every load-bearing
noun in it is an unknown. Conventional flat quiz packs ignore this and burn
the learner's attention on the wrong rung.

Recent educational research models this exactly: **Prerequisite Knowledge
Graphs** ([ACE, JEDM 2025](https://jedm.educationaldatamining.org/index.php/JEDM/article/view/737),
[RPKT, arXiv 2508.11892](https://arxiv.org/pdf/2508.11892)) represent each
atomic concept as a node and each "you must know Y before X" edge explicitly.
A question that probes concept X *implicitly assumes* its prereq closure —
spell that out so a downstream tool can detect gaps.

We pair PKG with **spaced retrieval** (the +25% meta-analysis effect from
[IJASSR 2025](https://journals.zeuspress.org/index.php/IJASSR/article/view/425)):
PKG decides *what* to teach next; SRS decides *when* to revisit it. The
StudyAndGame app already runs SRS via 오답노트 — PKG is the missing half.

### 12.1 Filling `concept`, `prereq`, `bloom`

For each question, before writing the YAML body, answer three questions:

1. **`concept`** — *What single atom does a correct answer prove the learner
   has internalised?* One concept per question. kebab-case ID, scoped to the
   domain (e.g. `mergetree-sparse-index`, not `sparse-index` which is ambiguous
   across DBs). If you cannot name one, the question is testing more than one
   thing — split it or rewrite.

2. **`prereq`** — *What concepts must the learner already understand for the
   question to even make sense?* List them as concept IDs. Two grades of
   inclusion:
   - **Required**: without it, the question is gibberish.
     `mergetree-primary-key-uniqueness` needs `rdb-primary-key`, `mergetree`,
     `sparse-index`.
   - **Excluded**: ambient prior knowledge (basic SQL, integers, English) —
     don't list these. The list stops at the boundary of *what this pack
     domain teaches*.

3. **`bloom`** — *What kind of cognitive work does the question demand?*
   - `recall` — definition lookup. ("MergeTree는 어떤 종류의 엔진?")
   - `understand` — explain in own words / paraphrase. ("primary key가 unique
     를 보장하지 않는 이유?")
   - `apply` — use the concept on a new instance. ("이 ORDER BY 표현식으로
     생성된 테이블에서 다음 쿼리는 인덱스를 쓸 수 있는가?")
   - `analyze` — compare/decompose/debug. ("두 partition 설계 중 어느 쪽이
     SELECT FD 폭증을 막는가?")

### 12.2 Graph completeness — the "no orphan prereq" rule

If a question lists `prereq: ['column-store']`, then **some other question in
the same pack OR a referenced sibling pack must have `concept: 'column-store'`
at a lower bloom level (typically `recall` or `understand`).** Otherwise the
chain dead-ends and a learner who lacks `column-store` has no path forward.

Sibling packs are tracked via `meta.references`:

```yaml
meta:
  title: ClickHouse MergeTree 심화
  references:
    - clickhouse-fundamentals          # this pack's prereq pool extends to here
```

When emitting a multi-pack series (fundamentals → core → advanced), build
**bottom-up**: write the fundamentals concepts first, then the next layer
gets to reference them in `prereq`. Never list a `prereq` that does not exist
yet — the gap is itself a bug.

### 12.3 The bloom progression rule

Within one pack: order questions roughly `recall → understand → apply →
analyze`. Within one **concept across packs**: introduce it at `recall` in
the fundamentals pack and only escalate to `apply` / `analyze` once that
exists. A pack that opens with an `analyze` on a concept never taught at
`recall` is a difficulty cliff — same failure mode the PKG was built to
prevent.

### 12.4 Self-check for PKG packs

Before emitting:

- [ ] Every `concept` is unique within the pack (or split the question)
- [ ] Every `prereq` ID is either (a) the `concept` of an earlier question
      in this pack at lower bloom, or (b) a `concept` taught by a pack in
      `meta.references`
- [ ] First question of the pack has empty or near-empty `prereq` — it is
      the root
- [ ] Pack ends at bloom `apply` or `analyze` — recall-only packs are
      flashcards, not workbooks
- [ ] `bloom` progression is monotonic-non-decreasing across the pack
      (small backsteps OK to introduce a new concept)
- [ ] **Explanation-body audit** (§12.5)

### 12.5 The explanation-body audit ("tail-of-tail" rule)

The PKG `prereq` field nails *what must I have learnt before this question*.
But the human-readable `explanation:` is free Korean + English text and can
silently introduce another domain term the learner has not seen yet — at
which point the chain breaks and the learner googles instead of clicking
the next prereq chip in our UI.

**Rule.** If the `explanation:` of question Q mentions a multi-word domain
term that *is* a `concept` of some other question in this pack (or any
referenced sibling pack), then either:

1. **List it in `Q.prereq`** so the learner can jump to the teacher
   question. This is the default.
2. **Forward reference (acceptable)**: if the cited concept is taught
   *later* in the same pack (so a prereq edge would create a cycle), drop
   the specific term from the explanation and replace it with a generic
   pointer — e.g. *"sparse index의 기반이다"* → *"그 정렬된 데이터 위에
   어떤 인덱스가 얹히는지는 뒤에서 별도 학습"*. The learner is told a
   later topic exists without being asked to recognise a term they have
   not yet been taught.

**What this catches that §12.2 misses.** §12.2 only validates `prereq`
field values. A question can have a perfectly closed `prereq` list and
still have an `explanation:` paragraph that drops six unknown nouns on the
learner. That paragraph IS prerequisite content, even if it is not in the
field.

**How to run the audit.** Studyandgame ships
`tests/debug_explanation_audit.gd` which loads every pack under
`res://data/quizzes/`, builds a `multi-word phrase → concept_id` map from
every `concept` value with ≥ 2 hyphen-separated parts, scans each
`explanation:` for those phrases, and flags hits not present in that
question's `prereq`. The exit code is zero only when the report is empty.

Single-word concept IDs are deliberately excluded — they overmatch their
own teacher's explanation. If a multi-word phrase you cite intentionally
*isn't* a concept (e.g. a SQL keyword everyone is assumed to know), the
audit ignores it, which is correct.

---

## 13. Vocab gloss cards (language packs — JLPT / 漢字 / 어휘)

For language-learning packs, a foreign-language word in the **question stem
or passage** is itself an obstacle: the learner can't answer a question they
can't even read. The optional `glossary` field attaches reading + meaning
cards that StudyAndGame renders **below the question, before the choices**.

### 13.1 Format — delimited scalars, NOT a nested map

The in-tree YAML parser (`yaml_pack_parser.gd`) is a schema-scoped subset: it
parses block lists of **scalars** but not block lists of **maps**. So a
glossary entry is one single-quoted scalar `'word｜reading｜meaning'`, split
by the app on the fullwidth bar `｜` (U+FF5C):

```yaml
  - type: mcq
    q: '「悪天候（　）、試合は予定どおり行われた。」빈칸에 알맞은 것은?'
    choices: [ ... ]
    answer: 3
    glossary:
      - '悪天候｜あくてんこう｜악천후'
      - '試合｜しあい｜시합'
      - '予定どおり｜よていどおり｜예정대로'
    explanation: ' ... '
```

- Exactly three `｜`-separated fields: **word · reading · meaning**.
- Reading may be empty (`'ケチ｜｜인색함, 구두쇠'`) for words already in kana.
- Meaning is in the learner's language (Korean here); commas inside it are fine.

### 13.2 Which words to gloss (the rule)

- Gloss words at roughly **N3 level and above** (N3 / N2 / N1). Skip N4/N5
  basics (私, 食べる, 学校) — glossing them is noise.
- **Never gloss the tested word or the answer.** On a reading/meaning question
  the target word IS the question; a card would hand over the answer. Those
  questions get no glossary. Gloss only the *surrounding* words.
- Order cards as the words appear in the stem.

### 13.3 Self-check

- [ ] Every glossary entry has exactly three `｜` fields (word non-empty).
- [ ] The answer / tested word is absent from the glossary.
- [ ] Only N3+ words are carded; trivial words omitted.

---

## 14. 구조 시각화 (figure) — 3D 단위격자 자동 첨부

For crystallography / solid-state / 재료공학 packs, a question *about a crystal
structure* is much clearer with the structure in front of the learner. The
optional per-question `figure` field tells StudyAndGame to render a **rotatable
3D unit cell** below the question. It is purely a visual aid — the YAML still
parses and grades identically without it.

**This field is OPTIONAL and off by default.** Attach it only when (a) the
question is genuinely *about* a structure the app can render, AND (b) the user
has opted in (see §14.3 trigger). When the source has no crystal structure,
attach nothing and say nothing.

### 14.1 Supported figure names (the ONLY valid values)

The app renderer and this list are kept in **exact sync**. A `figure:` value
outside this list is **silently ignored** by the app (no error, no picture) —
so never invent one.

| `figure` | 구조 / structure | Keyword triggers (in `q` or `explanation`) |
|----------|------------------|---------------------------------------------|
| `sc`         | 단순입방 / simple cubic | 단순입방, simple cubic, SC |
| `bcc`        | 체심입방 / body-centered cubic | 체심입방, body-centered cubic, BCC |
| `fcc`        | 면심입방 / face-centered cubic | 면심입방, face-centered cubic, FCC |
| `diamond`    | 다이아몬드 구조 / diamond | 다이아몬드 구조, diamond, Si, Ge, silicon, germanium |
| `hcp`        | 육방조밀 / hexagonal close-packed | 육방조밀, hexagonal close-packed, HCP |
| `zincblende` | 섬아연광 / zinc blende | 섬아연광, zinc blende, sphalerite, GaAs, ZnS |
| `rocksalt`   | 암염 / rock salt | 암염, rock salt, NaCl, halite, sodium chloride |
| `cscl`       | 염화세슘 / cesium chloride | 염화세슘, cesium chloride, CsCl |
| `fluorite`   | 형석 / fluorite | 형석, fluorite, CaF2, calcium fluoride |

### 14.2 When to attach (keyword auto-detection)

For each question, scan the `q` and `explanation` text for the keywords above:

- A hit → map to the matching `figure` name (e.g. `면심입방`/`FCC`/`단위격자`
  context + `유효 원자` → `figure: fcc`; `섬아연광`/`GaAs` → `figure: zincblende`;
  `암염`/`NaCl` → `figure: rocksalt`; `다이아몬드 구조`/`Si` → `figure: diamond`;
  `체심입방` → `figure: bcc`; `염화세슘`/`CsCl` → `figure: cscl`; `형석`/`CaF2`
  → `figure: fluorite`).
- **On-topic only.** Attach the figure only if the structure *is the subject*
  of the question — e.g. counting effective atoms, coordination number, or
  packing of that cell. A question that merely name-drops "NaCl 같은 이온결정"
  while testing something unrelated (e.g. 격자에너지 공식) gets **no** figure.
- **Never invent a name** outside §14.1. If the structure isn't in the list
  (e.g. perovskite, wurtzite), attach nothing — the app would ignore it anyway.
- One figure per question at most. If two structures are compared, pick the one
  the *answer* hinges on, or omit.

### 14.3 User-choice trigger (ask before attaching)

Figures are **opt-in**:

- **Default (no structure content):** attach nothing, don't mention figures.
- **Figure-eligible questions detected, no flag:** ask **once** before emitting
  — `"구조 그림(3D)을 N개 문항에 붙일까요? [예/아니오]"`. Add `figure:` lines
  only if the user answers yes.
- **User passed a `figures` / `그림` flag** (e.g. `/workbook ... figures`):
  skip the question and attach figures to all on-topic structure questions.
- **User answers no / omits the flag:** emit the pack with no `figure:` lines.

### 14.4 Worked example (FCC 유효 원자 + zinc blende)

```yaml
  - type: mcq
    q: '면심입방(FCC) 단위격자 안의 유효 원자 개수는?'
    choices:
      - '1'
      - '2'
      - '4'
      - '6'
    answer: 2
    explanation: |
      꼭짓점 8개 × 1/8 + 면 6개 × 1/2 = 1 + 3 = 4.
      FCC 단위격자의 유효 원자 수는 4개다.
    figure: fcc        # 질문이 FCC 단위격자 그 자체를 다루므로 on-topic
    tags: [crystal, fcc]

  - type: mcq
    q: '섬아연광(zinc blende) 구조의 GaAs에서 각 Ga 원자의 배위수는?'
    choices:
      - '4'
      - '6'
      - '8'
      - '12'
    answer: 0
    explanation: |
      zinc blende는 FCC 부격자 두 개가 (1/4,1/4,1/4)만큼 어긋나 겹친 구조로,
      각 Ga는 4개의 As와 정사면체 배위를 이룬다. 배위수 = 4.
    figure: zincblende
    tags: [crystal, zincblende, gaas]
```

---

## 15. 語彙「用法」(word-usage) questions — JLPT 語彙 섹션

JLPT 語彙 파트는 한자읽기·표기·문맥(빈칸)뿐 아니라 **用法(word-usage)** 문항을
포함한다. 단어 하나를 주고 "이 단어를 **올바르게 쓴** 문장을 고르라"는 형식이다.
플러그인이 지금까지 빈칸채우기만 만들어 이 표준 섹션을 빠뜨려 왔다 — 언어/어휘
팩(JLPT 등)을 만들 때는 用法 문항도 **반드시 섞어** 출제한다.

**새 question type가 필요 없다.** 用法은 그냥 `type: mcq`다 — 다만 보기 4개가
**문장 4개**라는 점만 다르다(정답 1개는 올바른 용법, 나머지 3개는 그럴듯한 오용).

### 15.1 Shape — 일반 mcq, 보기만 "문장 4개"

- `type: mcq`
- `q:` (stem) — `'「<word>」の使い方として最もよいものを選びなさい。'`
- `choices:` — 그 단어를 넣은 **완전한 문장 4개**. 정답 문장 1개 + 오용 문장 3개.
- `answer:` — 올바른 용법 문장의 0-based 인덱스.
- `target:` (optional) — 시험 대상 단어/표현을 담는다. 앱은 이 필드를 무시하지만
  채점·검수·태깅 시 "이 문항이 어떤 단어의 用法을 묻는가"를 기계가 알 수 있게 한다.

**오용 문장(distractor)은 "그럴듯해야" 한다, 황당하면 안 된다.** 각 오답 문장은
*문법은 자연스럽되* 그 자리에 그 단어가 **연어(collocation)·문맥상 어울리지
않게** 쓰여 있어야 한다 — 보통 그 자리엔 다른 단어가 와야 맞는 경우다. §5의
"plausible distractors" 규칙이 그대로 적용된다.

**explanation 규칙.** 정답 문장이 왜 맞는지 + **오답 문장 각각이 그 단어를 어떻게
오용했는지(원래 어떤 단어/표현을 썼어야 하는지)** 를 한 줄씩 적는다. 이것이 用法
문항의 학습 가치 전부다 — "왜 틀렸나"가 없으면 학습자는 배우지 못한다.

**보기를 가리킬 때는 번호(1번·정답 2 등) 대신 그 문장/단어를 「」로 인용**한다.
이는 모든 mcq explanation에 적용되는 규칙이다 — 앱은 보기를 1-based("1. 2. 3. 4.")로
표시하므로 0-based 번호나 어긋난 번호는 학습자를 헷갈리게 한다. 인용은 표시 번호와
무관하게 안전하다.

**글로싱(§13)과의 관계.** target 단어 자체는 시험 대상이므로 절대 glossary에
넣지 않는다(§13.2). 문장 속 *주변* N3+ 단어만 필요 시 카드로 단다.

### 15.2 Worked example (用法 mcq)

```yaml
  - type: mcq
    q: '「あいにく」の使い方として最もよいものを選びなさい。'
    target: 'あいにく'
    choices:
      - 'お訪ねしましたが、あいにく社長は外出しておりました。'
      - '宝くじに当たって、あいにくとてもうれしい気持ちになった。'
      - '彼はあいにく毎日まじめに勉強しているので成績がよい。'
      - 'この料理はあいにくおいしくて、何度も食べたくなる。'
    answer: 0
    explanation: |
      「あいにく」=「都合の悪いことに(공교롭게도)」. 기대가 어긋난 안 좋은
      상황에 쓴다. 정답 「あいにく社長は外出しておりました」: 방문했으나
      공교롭게도 부재였다. ／ 「あいにくとてもうれしい」는 좋은 결과(당첨)에
      붙었으니 오용 — 「思いがけず／運よく」가 맞다. ／ 「あいにく毎日まじめに
      勉強している」는 긍정적·일상적 사실에 부적합 — 「いつも」면 충분. ／
      「あいにくおいしくて」는 호평에 붙을 수 없다 — 「意外に」가 와야 한다.
    tags: [jlpt, n3, word-usage]
```

### 15.3 Self-check (用法 문항 전용)

- [ ] stem이 「<word>」の使い方として最もよいものを選びなさい。 형식이다.
- [ ] choices가 그 단어를 포함한 **완전한 문장 4개**다(단어·구가 아님).
- [ ] 정답 1개만 올바른 용법, 나머지 3개는 **그럴듯한** 오용(황당 문장 금지).
- [ ] explanation이 오답 문장마다 *원래 와야 할 단어/표현*을 명시한다.
- [ ] explanation이 보기를 번호 대신 「」 인용으로 가리킨다(0-based·어긋난 번호 금지).
- [ ] target 단어는 glossary에 넣지 않았다(§13.2).
- [ ] 언어 팩이면 用法 문항을 빈칸채우기와 **섞어** 출제했다.
