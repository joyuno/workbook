---
description: Verify a StudyAndGame .yml pack against the schema and quoting rules without uploading. Args — path to the .yml file.
---

You have been invoked through the `/workbook-verify` slash command. The user wants to dry-run-validate a StudyAndGame YAML pack before dropping it into the desktop app.

## What to do

1. Read the file at the path in `$ARGUMENTS` (use the `Read` tool).
2. Cross-check against the absolute rules from the `workbook` skill — invoke that skill to load the spec.
3. Report findings as a checklist:
   - [ ] Schema header (`meta.title`, `meta.version`, `questions[]`)
   - [ ] All `type` values are `mcq` or `ox`
   - [ ] All `mcq.answer` are 0-based ints in range
   - [ ] All `mcq.choices` have 2-6 items
   - [ ] All `ox.answer` are boolean (not strings)
   - [ ] All `q:` and `choices:` items with Korean / colons / asterisks / pure-digits are single-quoted
   - [ ] `explanation` present (warn if missing)
   - [ ] Answer-index spread (flag if one index dominates)
4. For each issue, output `line N: <problem> — <fix suggestion>`.
5. End with a single summary line: `OK — N mcq + M ox, no issues` or `N issues found — see above`.

## What NOT to do

- Do not modify the file. This is a verify pass only.
- Do not auto-fix unless the user explicitly asks in a separate turn.

## User input

$ARGUMENTS
