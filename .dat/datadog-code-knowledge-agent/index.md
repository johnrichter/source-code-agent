---
name: Code Knowledge Agent — effort index
description: "Routing index for the code knowledge agent effort: the build plan and resume point, the de-risking spike results, and the parser-binding migration, with the version-selection rule a consumer applies."
id: index:datadog-code-knowledge-agent:effort
tags: [type:index, topic:tooling, status:complete, privacy:public]
links: [project:tooling:datadog-code-knowledge-agent]
updated: 2026-09-02T00:00:00Z
---

# Code Knowledge Agent — effort index

Three planning documents for a citation-backed question-answering agent over public source code.

- `plan.md` — the build plan and resume point. All planned phases are done, and the remaining work is per-repository build-out.
- `S0-findings.md` — the de-risking spike results, which `plan.md` section 9 cites. It holds the A/B proof that split the indexed text from the citable text, and the version-churn measurement.
- `spike-10.8-tree-sitter-migration.md` — the migration off an unmaintained parser binding, with its own A/B table. That table, not the plan's older counts, is the chunk-count baseline.

## Choosing which versions to index

The plan bounds its version set by how widely each version runs, and no public source reports that. Apply the public rule instead: take each repository's released tags, and index backward from the latest until the list is exhausted or the per-repository cap is reached.

One case stays exact. A shared core library is vendored rather than installed, so its version set is derived from the tracer releases that pin it, and every one of those pins is public.

## Open items

Findings from these documents are filed as entries in this repository's own `.dat/feedback-register.json`. The documents stay as the historical record.
