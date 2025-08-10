# Translation Initiative

This repository is currently undertaking a full translation of the docs into Japanese. The effort starts with the BOTR (Book of the Runtime) section and then expands to the rest of `docs/`.

## Goal
- Translate all documentation under `docs/` to Japanese while preserving technical accuracy and structure.

## Current Focus
- Active: `docs/design/coreclr/botr` (BOTR)
- Japanese output root: `docs_ja/design/coreclr/botr` (mirrors original paths and filenames)

## Working Rules
- See `Trannslation_Rules.md` at repo root for style, metadata template, terminology, and workflow.
- Keep directory layout identical between `docs/` and `docs_ja/`.
- Use WIP/Review/Published status headers in each translated file’s metadata block.

## Progress Tracking
- Overall: `Docs_Translation_Progress.md`
- Perfolder e.g.BOTR folder: `docs_ja/design/coreclr/botr/TRANSLATION_PROGRESS.md`
- Mark WIP if it is work in progress.

## Next Steps
1. Scaffold WIP files in `docs_ja/design/coreclr/botr` mirroring all BOTR Markdown files.
2. Translate in batches; prioritize foundational docs (`intro-to-clr.md`, `exceptions.md`, `type-system.md`).
3. Periodically sync with upstream changes; record `source_commit` in each file.

If you’re picking this up: start by reading `Trannslation_Rules.md`, then open `docs_ja/design/coreclr/botr/TRANSLATION_PROGRESS.md` and claim a file by adding yourself to the metadata block in its JA counterpart.
