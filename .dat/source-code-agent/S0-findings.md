---
name: S0 Spike Findings — Datadog Code Knowledge Agent
description: "S0 de-risking spike results (plan.md §9) on dd-trace-py + dd-trace-go. Code-RAG retrieval recall, citation integrity, IndexText/Text split, version-interval storage cost vs fixed go/no-go bar."
id: project:tooling:source-code-agent-s0-findings
tags: [type:report, topic:tooling, topic:apm, status:complete, privacy:public]
links: []
updated: 2026-06-22T00:00:00Z
---

# S0 Spike Findings

**Date:** 2026-06-22 · **Scope:** dd-trace-py @ `c627814` + dd-trace-go @ `86ab9ad` · **Throwaway code:** `scratchpad/s0-code-spike/`

## Go/no-go bar (fixed BEFORE running, plan §9) — **PASS**
| Metric | Bar | Result (corpus A, the citation-safe config) | Verdict |
|---|---|---|---|
| golden-file-in-top-N recall | ≥ 85% | **100%** (13/13 at N=8, 16, 24) | ✅ PASS |
| citation-integrity (real source file) | = 100% | **100%** (13/13 real-file-verified) | ✅ PASS |
| refusals (no fabrication) | clean | 2/2 | ✅ PASS |
| version-case delta substrate | computable + accurate | intervals computable, 1.31× storage, real changed-symbol anchors | ✅ PASS (see caveat) |

> **Caveat on version-delta:** S0 proves the version-interval *substrate* (churn computable, intervals accurate, changed-symbol anchors real). It does **not** test `-version`-filtered *generation* — that filter is D0+Pv and is not buildable on the unmodified `ka` binary. Honest scope: substrate validated; filtered-answer behavior deferred to Pv where it's built.

## What was built (all throwaway, Python — D0 builds the real Go module)
- `chunker.py` — tree-sitter (py+go) AST chunker. One chunk ≈ one named symbol (func/method/class/type/const), leading doc-comment/decorator attached. Emits the docs-agent `Chunk` JSON schema so the **unmodified `ka` binary** ingests it (engine reuse confirmed).
- Corpus scope: py `ddtrace/propagation` + `ddtrace/_trace`; go `ddtrace/tracer` + `ddtrace/ext`. **1664 chunks** (583 py / 1081 go). Tests excluded.
- Two corpora: **A** = `Text` is body-only (citation-safe); **B** = `Text` is synthetic-header + body (tests retrieval lift + leak hazard).

## Engine-reuse findings (verified by reading ddocs source)
- **~80% reuse holds.** Index → embed (ollama nomic) → hybrid retrieve → rerank → Citations-API generation → verify ran on code chunks with **zero engine changes**, via a scratch `-corpus`.
- **Whitespace hazard (§2) already solved:** `agent/answer.go verifyCitation` already does `normWS` on both sides. No new work.
- **`ResolveText` is a safe pass-through on code:** it only rewrites the unicode sentinels `⟦rp:⟧`/`⟦site:⟧`, which source code never contains. Site-resolution is inert on code (confirms the §1 "DROP site-resolution" plan is clean).
- **Citation verifies against the chunk `Text`, not the source file** — so corpus B's header, if cited, would *falsely* verify in-engine. The harness adds the real-source-file substring re-check (the true code guarantee). This is precisely why the §2 `IndexText`/`Text` split is mandatory.
- Chunk **fidelity 40/40**: corpus-A `Text` is an exact byte-substring of the real source file at the recorded lines.

## Chunker gap found + fixed in-spike
- Initial chunker emitted only func/method/class/type → **dropped all const/var blocks** (e.g. `ddtrace/ext/` is pure constants: 0 chunks). Added const/var handling (py ALL_CAPS module assignments; go `const_declaration`/`var_declaration`). Const blocks are exactly where defaults/env-var names live — the §6 version-delta surface. **P0 must include const/var as first-class.**

## Recall + citation gates (15 goldens: 13 answerable + 2 refusals)
| Metric | Corpus A (Text = body) | Corpus B (Text = header+body) |
|---|---|---|
| recall@8 / @16 / @24 | 100% / 100% / 100% | 100% / 100% / 100% |
| citation engine-verified | 13/13 | 13/13 |
| **citation REAL-FILE-verified** | **13/13** ✅ | **0/13** ❌ |
| must_contain in answer | 13/13 | 12/13 |
| refusals clean | 2/2 | 2/2 |

## IndexText/Text split (§2) — **the headline result, now empirically mandatory**
Corpus B puts the synthetic symbol header into the citable `Text`. Result: the generator **preferentially cites the header** (it's the most symbol-identifying text), so **0/13** citations are real substrings of the source file — yet the engine reports **13/13 verified**, because it checks against the chunk `Text` (which contains the header), not the file. Every leak sample begins with `dd-trace-go · ddtrace/tracer · <symbol> · <kind>\n…` — text that does not exist in the repo.

**Conclusions:**
1. **The `IndexText`/`Text` split is mandatory, not optional.** Header → `IndexText` (BM25/embeddings only); `Text` → verbatim body only (the sole citable field). Confirmed by B = 0/13 real-file integrity behind a *false* 13/13 engine-verified — a silent forwardability failure if shipped.
2. **Body-only retrieval already clears the recall bar (A = 100%).** On this set the header gave **no recall lift** (A = B = 100%) — BM25 on identifiers is strong without it. So the header is a *possible* P2 retrieval tweak, not a requirement; **if ever added, the split is non-negotiable.** Simplest safe default: body-only.
3. **D0 must add real-source-file citation verification** (verify `cited_text` against the file at the SHA, not just the chunk `Text`) — the docs engine's chunk-Text check is necessary but insufficient for code. Cheap: the file + SHA are already in the clone.

## Version-interval cost (step 6 — churn/dedup across 8 successive minors)
| Repo | Window | latest-only | unique (deduped) | **storage ×** | adj. churn/minor |
|---|---|---|---|---|---|
| dd-trace-py | v3.12.9→v3.19.7 (8) | 511 | 668 | **1.31×** | 5.8% |
| dd-trace-go | v1.67.1→v1.73.2 (7, stable layout) | 676 | 888 | **1.31×** | 9.8% |
| dd-trace-go | v1.67.1→v1.74.8 (8, incl. relocation) | 191* | 978 | 5.12×* | 16.1%* |

\* **Artifact, not real churn.** dd-trace-go moved ~24 tracer files out of `ddtrace/tracer` into internal packages between v1.73 and v1.74 (pre-v2 refactor). A path-scoped, qualified-name-keyed measure conflates *moved* with *churned* → inflated multiplier and the 191 outlier. The stable-window row is the honest number.

**Findings:**
- **Storage multiplier ≈ 1.31× over ~8 minors for BOTH repos** — well under the plan's conservative "3–5×". Content-hash dedup collapses the unchanged majority as predicted. Confidence upgraded medium-low → **medium-high**. (More minors push it up sub-linearly; ~30-line cap likely lands ≤2–3×, still < 5×.)
- **The relocation result is a real S0 lesson for Pv:** §6's "symbol identity across refactors" is the binding hard problem. Symbol intervals must key on **symbol identity repo-wide** (qualified name + body-similarity re-link), NOT a frozen path — else file moves register as mass remove+introduce. The conservative "treat move as removed+introduced" rule (§6) is safe for *correctness* but will under-claim interval span across big refactors; a body-similarity re-link is needed to avoid pessimism. **Validates building Pv with care; does not block S0.**
- Per-symbol intervals are computable: py 462 symbols stable across all 8 tags, 89 changed ≥once (a real "behavior changed in vX" set); go 132 stable, 70 changed — concrete version-delta anchors exist.

## Verdict — **GO. Proceed to D0.**
The single de-risking unknown (§0: "does code RAG retrieve well enough at chunk quality + scale") is answered: **yes** — 100% recall and 100% real-file citation integrity on the citation-safe config, with ~80% engine reuse confirmed and the version-interval storage model validated at ~1.3× (well under budget).

**Design mandates carried into D0/P0/Pv (hardened by S0 evidence):**
1. **P0:** `IndexText`/`Text` split is mandatory (B = 0/13 proves it). `Text` = verbatim body only.
2. **D0:** add real-source-file citation verification (verify against file@SHA, not just chunk Text).
3. **P0:** const/var blocks are first-class chunks (gap found + fixed in-spike).
4. **Pv:** symbol-identity must be repo-wide (qualified-name + body-similarity re-link), not path-scoped — the dd-trace-go v1.74 relocation proves path-anchored intervals mis-count moves as churn.
5. **P2:** body-only retrieval already clears recall; treat the synthetic header as an optional tweak to *measure*, not a default to assume.

**Storage go-ahead:** ~1.3× over 8 minors → the ~30-line/repo cap (version-sets.json) projects comfortably under the 3–5× / LFS budget. Safe to commit to the historical backfill at Pv.

## Reproduce
`scratchpad/s0-code-spike/`: `chunker.py` (chunker), `goldens.json` (15 cases), `harness.py` (recall+citation gates), `churn.py` (version churn), `results.json` + `churn.json` (raw), this file (findings). Throwaway — D0 starts the real Go module by copying `ddocs`.

