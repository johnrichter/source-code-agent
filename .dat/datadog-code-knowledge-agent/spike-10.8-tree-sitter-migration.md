---
name: "Spike 10.8 — tree-sitter binding migration (smacker → official)"
description: "Tracking plan for migrating the code chunker off the unmaintained smacker/go-tree-sitter (ABI 14) to the official tree-sitter/go-tree-sitter (ABI 15). Verified dep matrix, full-rewrite transform spec, single-pass validation goldens, risk register, live progress log. Read before working spike 10.8."
id: project:datadog-code-knowledge-agent:tree-sitter-migration
tags:
  - type:project
  - topic:tooling
  - topic:apm
  - status:complete
  - privacy:public
links:
  - agent:datadog-code-knowledge-agent:build-plan
updated: 2026-06-23T22:00:00Z
---

# Spike 10.8 — migrate `smacker/go-tree-sitter` → official `tree-sitter/go-tree-sitter`

**Goal:** replace the unmaintained, ABI-14-frozen, single-bundle `smacker/go-tree-sitter` with the actively-maintained official `tree-sitter/go-tree-sitter` (ABI 15) + per-language first-party grammar modules. Unblocks the Dart ABI-15 pin and future ABI moves. Convention (`agent/identity.md`): prefer official/first-party deps, decide deliberately, surface trust trade-offs.

**Operator decisions (locked 2026-06-23):** (1) **full rewrite** of all extractors against the official API — no facade; (2) **swift** pinned to a `-with-generated-files` tag; (3) **all 14 languages in one pass**, validate every golden together at the end.

## Verified facts (inspected on cloned upstream, not from memory)

Clones for reference: `<local-clones>`.

- **Runtime** `github.com/tree-sitter/go-tree-sitter` latest tag **v0.24.0**; bundles the core C (`src/lib.c`). Core **`TREE_SITTER_LANGUAGE_VERSION` 15, `MIN_COMPATIBLE` 13** → loads ABI 13/14/15 grammars. cgo + C compiler still required (unchanged).
- **ABI policy:** runtime is **ABI 15 — the latest** (v0.24.0 is the newest official Go-binding release; there is no "newer" to target, 15 is the ceiling not a floor). We do **not** force grammars to 15: each grammar uses its latest-stable ABI and the runtime loads anything ≥13. Current spread — ABI 15: python/go/js/c#/rust/c/cpp/php/dart; ABI 14: java/ruby/typescript/kotlin/swift. An all-ABI-15 grammar set is impossible (java/ruby/ts/kotlin/swift have no ABI-15 release) and unnecessary.
- **Grammar bindings** are self-contained cgo (`bindings/go/binding.go` does `#include "../../src/parser.c"`), each exporting `Language() unsafe.Pointer`; depend on the runtime only for types. Adding a language = add one module.
- **API delta vs smacker** (drives the rewrite):
  - `sitter.ParseCtx(ctx, src, lang)` → `p:=ts.NewParser(); p.SetLanguage(l); t:=p.ParseCtx(ctx,src,nil); root:=t.RootNode()` (+ `Close()` on parser & tree).
  - `node.Content(src)` → `node.Utf8Text(src)`.
  - `node.StartPoint()/EndPoint()` → `node.StartPosition()/EndPosition()` (`.Row` is `uint`).
  - `node.Child(uint)`, `node.ChildCount() uint`, `node.StartByte()/EndByte() uint` (smacker were int/uint32). Existing `int(node.StartByte())` casts stay valid.
  - `grammar.GetLanguage()` → `ts.NewLanguage(grammar.Language())`.
  - typescript binding exports `LanguageTypescript()` / `LanguageTSX()`; php exports `LanguagePHP()` / `LanguagePHPOnly()`.

## Dependency matrix (the deps we rely on) — FINAL, RESOLVED & SMOKE-TESTED

Runtime: **`github.com/tree-sitter/go-tree-sitter v0.25.0`** (latest; ABI 15, `MIN_COMPATIBLE` 13). Confirmed latest via `go list -m -versions` (full tag set: v0.23.0/v0.23.1/v0.24.0/v0.25.0).

| Lang | Module | Class | Pin (go.mod) | Loaded ABI | License |
|---|---|---|---|---|---|
| python | `tree-sitter/tree-sitter-python` | official | v0.25.0 | 15 | MIT |
| go | `tree-sitter/tree-sitter-go` | official | v0.25.0 | 15 | MIT |
| java | `tree-sitter/tree-sitter-java` | official | v0.23.5 | 14 | MIT |
| ruby | `tree-sitter/tree-sitter-ruby` | official | v0.23.1 | 14 | MIT |
| javascript | `tree-sitter/tree-sitter-javascript` | official | v0.25.0 | 15 | MIT |
| typescript/tsx | `tree-sitter/tree-sitter-typescript` | official | v0.23.2 | 14 | MIT |
| c# | `tree-sitter/tree-sitter-c-sharp` | official | v0.23.5 | 15 | MIT |
| rust | `tree-sitter/tree-sitter-rust` | official | v0.24.2 | 15 | MIT |
| c | `tree-sitter/tree-sitter-c` | official | **v0.24.0** | 15 | MIT |
| cpp | `tree-sitter/tree-sitter-cpp` | official | v0.23.4 | 14 | MIT |
| php | `tree-sitter/tree-sitter-php` | official | v0.24.2 | 15 | MIT |
| kotlin | `fwcd/tree-sitter-kotlin` | community | **commit `c8ac3d262724`** (2026-06-02 tip) | 14 | MIT |
| swift | `alex-pinkus/tree-sitter-swift` | community | **commit `31d17fe7e818`** (`0.7.3-with-generated-files`) | 14 | MIT |
| dart | `UserNobody14/tree-sitter-dart` | community | **commit `a9bdfa3db2fb`** (2026-05-20 tip, ABI 15) | 15 | MIT |

1 runtime + 14 grammar modules, all MIT, 11 first-party. kotlin/swift/dart stay community (no official grammar exists). Grammars span ABI 14–15; all load under the ABI-15 runtime (floor 13).

### Dependency-resolution findings (S1)

- **Runtime latest is v0.25.0, not v0.24.0** — the depth-1 clone's `ls-remote` truncated tag refs; `go list -m -versions` is authoritative. v0.25.0 ABI is still 15 / min-compat 13, so ABI-14 grammars still load.
- **`tree-sitter-c` v0.24.1 & v0.24.2 are broken** — their go.mod requires `go-tree-sitter v0.24.1`, a runtime version that was never published (0.24.0→0.25.0). Pinned **c v0.24.0** (highest c release requiring an existing runtime).
- **kotlin tag `0.3.8` is not a usable Go module version** (no `v` prefix → not proxy-fetchable; `go get @<that-commit>` silently fell back to an unrelated 2024 pseudo-version). Only clean semver is the ancient `v0.3.2`. Pinned the **proxy `@latest` tip commit `c8ac3d262724`**, which equals our validation clone HEAD.
- **API rename to carry into the rewrite:** official binding uses `node.Kind()` (smacker `node.Type()`); also `Utf8Text`, `StartPosition`/`EndPosition`, `uint` child indices, parser-object model with `Close()`.
- **Swift:** `-with-generated-files` commit pin compiles cleanly (only a harmless `TOKEN_COUNT` macro-redefine warning); no `tree-sitter generate` build step needed.

### S1 smoke result (throwaway probe, since removed)

All 14 grammars compiled via cgo and loaded under v0.25.0; every parse returned `err=false`. Root kinds sane (python→`module`, go→`source_file`, php→`program`, dart→`program`, …). **S1 green.**

## Execution checklist (update status as completed)

- [x] **S1 — deps:** runtime + 14 grammar modules added to `ddcode/go.mod` at the final pins; all compile via cgo + load under v0.25.0 (smoke-verified). smacker removal + `go mod tidy` deferred to S5 (still imported until the rewrite lands).
- [x] **S2 — dispatch (`codechunk.go`):** parser-object model (`NewParser`/`SetLanguage`/`ParseCtx`/`Close`); new `grammarFor()` wraps the 14 official `Language()` bindings; `node.Type()`→`Kind()`, `Content`→`Utf8Text`, `StartPoint/EndPoint`→`StartPosition/EndPosition` swept across helpers. Kept the `sitter` import alias pointed at the official module (no facade — direct API rewrite).
- [x] **S3 — extractors (×11 files + py/go in dispatch):** mechanical API transform applied via perl + hand-fixed child-index `uint()` sites. No node-type-string changes needed for compile (drift surfaces at S6). `resolve` builds clean.
- [x] **S4 — dart wrapper:** `dartLanguage()` now wraps via the official `sitter.NewLanguage`; ABI-15 grammar pinned; quirk-handling code intact (validated by the dart unit test passing).
- [x] **S5 — build/tests + smacker removal:** `go build ./...` + `go test ./...` fully green; `go mod tidy -e` (worked around the swift binding's broken test-import) dropped smacker from go.mod, promoted grammars to direct; orphan smacker go.sum lines removed; `go mod verify` = all modules verified. **smacker absent everywhere.**
- [x] **S6 — regression (A/B, not stale-golden):** A/B (identical clone+flags, old smacker vs new official). **9/14 byte-identical** (py, go, java, ruby, js, ts, c, php, rust); kotlin/dart improved; cpp benign (vendored); **swift + c# = upstream-grammar parse-ERROR limitation on ~4% of files** (accepted + logged per operator). Full diagnosis above.
- [x] **S7 — wrap-up:** smacker fully removed; dart `extension type` unit test added; `plan.md §10.8` marked DONE + status block updated + goldens rebaselined as historical (A/B table is the new baseline). `refresh-code-agent` SKILL.md needs **no change** (pins no grammar — only the cgo prereq, unchanged). swift/c# ERROR-tolerance tracked as follow-up in §10.8.

### Golden chunk counts (original P0/P6a record, full scopes)

| py | go | java | ruby | js | c# | rust | c | cpp | kotlin | swift | php | dart |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 584 | 1086 | 1534 | 1855 | 3844 | 909 | 7581 | 312 | 2562 | 10338 | 21323 | 1988 | 5233 |

### S6 A/B results (old smacker vs new official, identical clone+flags per repo)

| Lang | Repo / scope | old | new | Δ | Verdict |
|---|---|---|---|---|---|
| python | dd-trace-py/ddtrace | 9769 | 9769 | 0 | identical |
| go | dd-trace-go/ddtrace | 1367 | 1367 | 0 | identical |
| java | dd-trace-java/dd-trace-api | 1597 | 1597 | 0 | identical |
| ruby | dd-trace-rb/lib | 9866 | 9866 | 0 | identical |
| javascript | dd-trace-js/packages | 5587 | 5587 | 0 | identical |
| typescript | browser-sdk/packages | 2127 | 2127 | 0 | identical |
| c | libdatadog (whole, -lang c) | 311 | 311 | 0 | identical |
| php | dd-trace-php/src | 1988 | 1988 | 0 | identical |
| rust | libdatadog/datadog-sidecar | 569 | 569 | 0 | identical |
| kotlin | dd-sdk-android/dd-sdk-android-core | 1569 | 1632 | +63 | **improvement** — new grammar exposes `interface`s + members old missed; 1 trivial removal |
| cpp | dd-trace-cpp/src | 1245 | 1293 | +48 | **benign** — all in vendored `nlohmann/json` (more template getters + 1 enum); no tracer-source loss |
| c# | dd-trace-dotnet/tracer/src | 49581 | 50143 | +562 | **mixed** — +534 methods/props captured, but `record` types (TracerSettings family) lose members → **records regressed** |
| swift | dd-sdk-ios/DatadogCore | 953 | 937 | −16 | **regression** — production `class DataUploadWorker` collapses: 19 qualified members → 3 de-qualified top-level funcs (new grammar parses that file's class body into a shape our walk misses) |
| dart | dd-sdk-flutter/packages | 4811 | 4810 | −1 | minor — 1 top-level func (`RumWebRawResourceEvent`) dropped |

**Read:** the migration is behavior-preserving for 9 languages and *improves* kotlin. The drift is the newer grammars parsing differently — so matching the old (ABI-14, unmaintained) counts is the wrong bar.

### S6 resolution (root-caused, fixed where clean)

- **dart — FIXED + improved.** The −1 was the extractor not handling Dart 3.3 `extension_type_declaration` (a new node; reuses `identifier` name + `class_body`). Added two cases (`dartScope` route + `extension type` kind). dd-sdk-flutter/packages now **4926** chunks (was 4811 old / 4810 new) — recovers the lost symbol AND captures **35 extension types + members the old grammar missed entirely**. Dart unit tests green.
- **kotlin — keep (improvement).** New grammar exposes interfaces + members; no action.
- **cpp — accept (benign).** Drift entirely in vendored `nlohmann/json`; no tracer-source loss.
- **swift + c# — upstream-grammar parse-error limitation (not node-rename, not cleanly fixable).** ~4% of files emit a tree-sitter `ERROR` root under the *latest* grammars (swift 3/72; c# 201/5073), trapping that file's members. Verified the old smacker grammars parsed several of these files (so it's a real regression on those files), BUT: (1) the new grammars are **net-better** (c# +562 overall); (2) many c# error-files are **codegen templates** (`$(…)` placeholders — correctly unparseable); (3) the latest grammar is already pinned (no newer release to adopt — older swift tags don't even compile); (4) the only "fix" is fragile per-language ERROR-tree reconstruction for ~4% recall. **Recommendation: accept + log as a known limitation, rebaseline goldens, track a follow-up to adopt upstream grammar fixes / a general ERROR-tolerant pass.** Awaiting operator confirmation.

## Risk register

- **Node-type drift (primary):** official grammars are newer than smacker's 2024 snapshots; a renamed/restructured node makes an extractor `case "x":` silently under-emit. Caught by the golden regression; fix is per-language node-name correction.
- **Swift generated-files tag** must be `go get`-able and carry `src/parser.c`. Mitigation: verified the tag exists upstream; confirm at S1.
- **Dart ABI-15 tip** may have shifted node shapes vs the old pin; the dart extractor has delicate quirk handling → S4 re-validation on dd-sdk-flutter.
- **Build time/footprint:** 14 cgo grammar compiles (cpp/php parser.c are large) → slower builds; equivalent to smacker bundling. go.sum surface grows but all MIT, 11 first-party.

## Progress log

- 2026-06-23 — Plan authored. Upstream runtime + 14 grammars cloned & inspected; dep matrix + API delta verified on disk. Operator locked: full rewrite, swift generated-files pin, single-pass. Worktree `feat/tree-sitter-official` created.
- 2026-06-23 — **S1 done.** All 15 modules resolved at final pins (runtime v0.25.0; c→v0.24.0 after the v0.24.1/.2 broken-release finding; kotlin→tip commit after the unusable-tag finding; swift/dart commit pins). Throwaway smoke probe compiled all 14 grammars via cgo and loaded each under v0.25.0 — all parsed `err=false`. Discovered `node.Type()`→`node.Kind()` rename for the rewrite.
- 2026-06-23 — **S2–S5 done.** Full API rewrite (parser-object model, `grammarFor` over official bindings, Kind/Utf8Text/StartPosition, `uint` child indices). `go build ./...` + `go test ./...` green on first compile — every per-language extractor fidelity unit test passes. smacker fully removed (go.mod/go.sum), `go mod verify` clean.
- 2026-06-23 — **S6 A/B run.** Built old+new `ka`, diffed identical builds per grammar. 9/14 byte-identical; kotlin improved; cpp benign (vendored); c# records + swift class bodies + 1 dart func regressed on the newer grammars. Root-caused all 5. Operator chose: fix regressions + keep improvements + rebaseline.
- 2026-06-23 — **S6 resolution.** dart FIXED (extension types; now net-better than old). kotlin/cpp keep. swift+c# root-caused to upstream-grammar parse ERRORs on ~4% of files on the latest grammars (not node-rename; partly codegen templates; net-better overall) — recommended accept+log+rebaseline+follow-up.
- 2026-06-23 — **S6/S7 done.** Operator confirmed accept+log+rebaseline+follow-up for swift/c#. dart extension-type unit test added (locks the fix); plan.md §10.8 marked DONE + status updated + goldens rebaselined as historical; SKILL.md unchanged (no grammar pins). Full suite green, smacker gone, `go mod verify` clean. **Migration COMPLETE — ready to commit + merge.**
