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
    reward: <int >= 1, optional, default 1>         # 강화권 배수 — 시간이 오래 걸리는 유형(독해)에 2~3 (§14)
    tags: [<tag>, ...]
    # ── Reading passage (optional — 독해 문제; see §14) ──────────────
    passage: '<지문 본문. 정답은 반드시 이 지문에서 도출>'  # 있으면 문제 위에 지문 패널로 표시
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
- [ ] **Single defensible answer** — the marked answer is uniquely correct, every other choice refutable (§6.1)
- [ ] Across the pack: answer indices are spread

If a question fails the check, **skip it** — quality over count.

### 6.1 Single defensible answer (정답 유일성)

A distractor must be **plausible but refutable** — plausible enough to tempt,
yet provably wrong from the source/passage. The failure mode to kill: a choice
that the given text supports **just as well as** the marked answer. Then there
are really two right answers and the learner is punished for a coin-flip.

**The trap (real example).** Passage/stem:

> その国の主要（　）は半導体と自動車だ。
> choices: ['輸出品', '輸入品', '生産品', '消費財'] — answer: 0 (輸出品)

半導体と自動車が主要な **輸出品** なのか **輸入品** なのかは、この一文だけでは
決められない。輸出品も輸入品も文脈上ともに成立する → **曖昧問題、廃棄**。

**The fix — two options:**
1. **Disambiguate the text.** Add a sentence that excludes the rival choice:
   「その国は半導体と自動車を世界中に**輸出して**外貨を稼いでいる。」→ 輸入品が排除され、輸出品が唯一解になる。
2. **Drop the rival distractor.** Replace 輸入品 with a clearly-refutable option.

**Authoring test — for EACH non-answer choice ask:** "Is there a specific
sentence in the passage/stem that rules this out?" If you can't point to one for
some distractor, that distractor is either the answer's twin (ambiguous → fix or
drop) or pure noise (too easy → strengthen). This applies to ALL mcq, but is
most dangerous in 内容理解/穴埋め where two choices share a category
(輸出↔輸入, 増加↔減少, 原因↔結果, 賛成↔反対).

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

## 14. Reading-passage questions (독해 — JLPT 内容理解 / 主張理解 / 情報検索)

A reading question is still **`type: mcq`** — the answer mechanic is unchanged
(pick one of the choices). What makes it a reading question is two optional
fields:

```yaml
- type: mcq
  passage: '<지문 본문 — 정답이 반드시 이 지문에서 도출되어야 함>'
  q: '<지문에 대한 질문>'
  choices:
    - '<선택지 1>'
    - '<선택지 2>'
    - '<선택지 3>'
    - '<선택지 4>'
  answer: <0-based int>
  explanation: '<정답 근거 — 지문의 어느 문장에서 나왔는지 인용>'
  time: 90            # 독해는 한 문제 푸는 데 오래 걸림 — §14.1
  reward: 3           # 시간 대비 보상 — §14.2
```

`passage` shows as a scrollable panel **above** the question. Multiple
consecutive questions may repeat the **same** `passage` string to model "one
passage, several questions" (内容理解 中文/長文, 統合理解).

### 14.1 Time per reading question (`time`)

JLPT 독해 한 문제당 평균 소요시간 기준. 지문 길이로 정한다:

| 지문 유형 | 분량 | `time` |
|-----------|------|--------|
| 短文 (内容理解·短) | ~200자 | `60` |
| 中文 (内容理解·中) | ~500자 | `90` |
| 長文 / 主張理解 | ~900자 | `120` |
| 情報検索 (광고·표) | 스캔형 | `90` |

### 14.2 Reward multiplier (`reward`)

기본 어휘·문법 문제는 `reward` 생략(=1). 독해는 시간이 ~3배 들므로
보상도 비례시킨다 — 短文 `2`, 中文·長文 `3`. 過하게 주지 말 것(밸런스 붕괴).

### 14.3 Rules

- **정답은 100% 지문에서 도출** — 외부 지식 없이 지문만으로 답이 나와야 한다.
  지문에 근거가 없으면 그 문제는 버린다 (No hallucination 원칙 유지).
- `passage` 도 한국어/CJK는 작은따옴표로 감쌀 것 (§4 동일 규칙).
- 지문 속 N3+ 어휘는 `glossary` 로 카드화 가능하되, **정답 단서가 되는 단어는 제외**.

### 14.4 Self-check

- [ ] `passage` 의 어느 문장이 정답 근거인지 `explanation` 에 인용했다.
- [ ] `time` 이 지문 분량과 맞는다 (§14.1).
- [ ] `reward` 가 1~3 범위, 어휘/문법 문제엔 reward 안 줬다.
