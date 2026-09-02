---
name: Code Knowledge Agent — effort index
description: "Routing index for the code knowledge agent effort in this repository: the build plan, the S0 de-risking spike results, and the tree-sitter binding migration, with a note on what this scrubbed copy omits."
id: index:datadog-code-knowledge-agent:effort
tags: [type:index, topic:tooling, status:complete, privacy:public]
links: [project:tooling:datadog-code-knowledge-agent]
updated: 2026-09-02T00:00:00Z
---

# Code Knowledge Agent — effort index

Three planning documents for a citation-backed question-answering agent over public source code.

- `plan.md` — the build plan and resume point. All planned phases are done. The remaining work is per-repo build-out.
- `S0-findings.md` — the de-risking spike results, which `plan.md` section 9 cites. It holds the A/B proof that split the indexed text from the citable text, and the version-churn measurement.
- `spike-10.8-tree-sitter-migration.md` — the migration off an unmaintained parser binding, with its own A/B table. That table, not the plan's older counts, is the chunk-count baseline.

## What this copy omits

This is a scrubbed copy, prepared for public release. Every reference to a vendor-internal source, identifier, or measurement is removed, and the technical design is unchanged.

One consequence is worth stating plainly. The plan bounds its version set by usage data, and that input is vendor-internal. A public consumer substitutes a public rule: take each repository's released tags, and index backward from the latest.

## Open items

Findings swept out of these documents are filed as LED entries in this repository's own `.dat/feedback-register.json`. The documents stay as the historical record.
