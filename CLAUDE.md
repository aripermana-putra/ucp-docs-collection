# CLAUDE.md — UCP Documentation Repository

This file provides guidance to Claude Code when working with documentation in this repository.

## Repository Purpose

This repository contains project documentation for UCP (Universal Control Platform).
It is separate from the `ucp-platform` source repository.

**Remote:** `git@ghe.rakuten-it.com:aripermana-putra/ucp-docs-collection.git`
**Main branch:** `main`
**Docs location:** `docs/`

---

## Documentation Writing Style

Documentation describes **what is**, not **how we got here**. Write every document as
if the current implementation is the only implementation that ever existed.

### Rules

**No history or journey language**
- Do not write "originally we planned X but switched to Y"
- Do not write "the old code had a bug where..."
- Do not write "what was originally planned vs what was built"
- Do not write "Finding N:" as a section format — just state the fact directly
- Do not use strikethroughs (`~~text~~`) to show superseded decisions

**No status-report framing in design documents**
- Design docs describe the design, not a checklist of what is done vs not done
- Implementation status belongs in `implementation.md` files, not in design docs
- Avoid "Current State:" sections that enumerate "what exists" and "what does not exist"
- Cross-reference `implementation.md` rather than duplicating status inline

**No editorial footnotes about correctness**
- Do not add "(note the 's')" or similar annotations implying a gotcha was discovered
- Document the correct API/field/behavior directly without explaining why it might be wrong

**Tense and voice**
- Use present tense: "the implementation uses Cloud Monitoring" not "we switched to Cloud Monitoring"
- Use declarative statements: "Cloud SQL instance count has no metric ID" not "we discovered that..."
- Limitations and gaps are stated as facts, not as failures of a plan

### Example

**Wrong:**
> The original plan used the Cloud Quotas API. During implementation, a JSON tag bug
> (`dimensionInfos` vs `dimensionsInfos`) caused all limits to return 0. The final
> implementation switched to Cloud Monitoring only.

**Right:**
> Both quota limits and usage are read from Cloud Monitoring
> (`serviceruntime.googleapis.com`). No separate quota API is used.

---

## File Structure Conventions

- Each PoC or feature area gets its own subdirectory under `docs/projects/<project>/pocs/<area>/`
- Every directory should have at most one file per concern — avoid splitting one topic across many small files
- Prefer merging related files over creating new ones
- File naming: lowercase with hyphens, descriptive noun phrases (`implementation.md`, `gcp-api-reference.md`)

### Standard files per PoC area

| File | Purpose |
|------|---------|
| `implementation.md` | What is built: scope, sequence diagram, limitations, API details |
| `<provider>-api-reference.md` | Provider-specific API reference (confirmed field structures) |
| `concepts.md` | Terminology, mental models, background knowledge |
| `ucp-<area>-design.md` | UCP-level design spec for future implementation phases |

---

## Commit Conventions

- Commit message format: `docs: <what changed>` (e.g. `docs: add quota API sequence diagram`)
- Always commit to `main` in this repo — no feature branches unless changes are large and draft
- Do not commit draft or incomplete content — each commit should be self-contained and accurate

---

## Worklog

Worklogs live at `docs/worklog/{YYYY-MM-DD}.md`. They are personal daily notes and do not
follow the same writing style rules above — they may include exploratory thoughts, open
questions, and session reflections.
