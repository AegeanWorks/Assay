# Decisions

Why this project is built the way it is, oldest first. One entry per decision that changes a
contract, a dependency, a data format or the architecture; IDs are consecutive and never reused.

### ADR-001: AGPL-3.0-or-later core with a commercial licensing path
- **Date**: 2026-10-23
- **Status**: Accepted
- **Context**: Companies will self-host evidence software and may embed it in proprietary
  products. The project must stay open enough to be trusted by auditors and funded by the
  companies that capture the most value.
- **Decision**: License the whole project under AGPL-3.0-or-later. Sell commercial licenses for
  embedding, enterprise support, policy packs, retention connectors and signed evidence
  operations. Never paywall the verifier or the evidence format.
- **Consequences**: Closed-source SaaS use requires a commercial license. The open core remains
  auditable and offline-verifiable, which is the trust argument the product depends on.

### ADR-002: JSONL hash chain with optional HMAC signing
- **Date**: 2026-10-23
- **Status**: Accepted
- **Context**: Evidence must be portable, append-only and verifiable with the standard library
  only. A database would hide the integrity rules behind a vendor-specific engine.
- **Decision**: Store one canonical-JSON record per line. Each record contains `seq`, `prev`,
  `event` and `hash`, where `hash = SHA-256(canonical({"seq","prev","event"}))`. An optional
  `mac = HMAC-SHA256(key, same payload)` lets a key holder detect a full-chain rewrite.
- **Consequences**: Humans and auditors can reimplement verification in any language. A hash
  chain alone cannot detect truncation or a complete rewrite; deployments should anchor the head
  and protect the HMAC key. See `docs/SECURITY.md`.

### ADR-003: First-match policies bound to a policy digest
- **Date**: 2026-10-23
- **Status**: Accepted
- **Context**: A review finding is only defensible if the reviewer knows which policy was in
  force when the action happened. Rule engines that silently combine rules are hard to explain.
- **Decision**: Evaluate rules in file order and stop at the first match. Support exact action
  globs, exact metadata matches, and `amount_over` / `amount_under` thresholds. Any amount rule
  that cannot see a finite numeric `amount` becomes a deny finding. Hash the normalized policy
  and store its digest on events recorded with `--policy`.
- **Consequences**: Policy order is part of the contract and must be reviewed. Missing amounts
  fail closed. A later policy change is visible as `policy_drift` rather than silently
  reinterpreted history.

### ADR-004: Self-contained evidence packs and a strict exit-code contract
- **Date**: 2026-10-23
- **Status**: Accepted
- **Context**: An evidence pack is often copied to a buyer, auditor or insurer who cannot access
  the original repository. Findings without records are not evidence.
- **Decision**: The JSON pack embeds every ledger envelope, the verification result, policy
  metadata, findings and the chain head. Markdown is a human review surface, capped at 200 event
  rows, with a pointer to the JSON for the complete set. Commands return 0 for success, 1 for a
  failed verification or policy finding, and 2 for usage, configuration or input errors.
- **Consequences**: Packs can be large and may contain sensitive metadata. Producers must
  minimize PII. `report` still writes a FAIL pack when the chain is broken, then exits 1.

### ADR-005: Zero runtime dependencies and Python 3.10+
- **Date**: 2026-10-23
- **Status**: Accepted
- **Context**: Security and compliance teams must be able to vendor, inspect and run the verifier
  in locked-down environments. A dependency tree expands the supply-chain review.
- **Decision**: Use only the Python standard library at runtime. Require Python 3.10 or newer.
  Keep packaging limited to `setuptools` at build time. Do not call external services from the
  recording, verification or reporting path.
- **Consequences**: Some conveniences (Ed25519, SQLite indexing, cloud storage adapters) move to
  optional enterprise components or later versions. The core remains auditable in one sitting.

### ADR-006: Head anchoring journal with key identification and rotation format
- **Date**: 2026-10-24
- **Status**: Accepted
- **Context**: Hash chains alone cannot detect tail truncation or a full rewrite. Operators need to
  anchor the head seq and hash to an independent append-only journal, with a key ID for HMAC
  rotation without exposing secrets.
- **Decision**: Introduce `AnchorJournal` and the `assay anchor` command. Store canonical JSON records
  with `schema`, `ledger`, `seq`, `head`, `created_at`, `key_id`, `prev`, and `hash`. Require
  `--key-id` whenever `--hmac-key-env` is used to bind anchors to key rotation identifiers.
  `assay verify --anchors FILE` validates that every anchored head matches the ledger and refuses
  truncated tails or rewritten chains.
- **Consequences**: Truncation and rewrites are detectable offline with the standard library. Anchor
  files are append-only and independently verifiable. Key rotations do not break verification history.

### ADR-007: Zero-dependency agent framework adapters
- **Date**: 2026-10-24
- **Status**: Accepted
- **Context**: Autonomous agents built on MCP, OpenAI Agents SDK, LangGraph, or CrewAI emit
  heterogeneous event structures. Requiring developer teams to install heavy SDK dependencies
  inside Assay or write manual transformation glue slows adoption and introduces supply-chain risk.
- **Decision**: Ship pure standard-library adapters in `assay.adapters` that transform JSON-RPC
  and dictionary telemetry from MCP, OpenAI, LangGraph, and CrewAI into canonical `Event` records.
  Support stream ingestion via `assay ingest --format <name>` with session actor fallbacks.
  Duck-type dictionary payloads without requiring runtime SDK packages.
- **Consequences**: Assay integrates out-of-the-box with the top 4 agent ecosystems without
  violating ADR-005 (zero runtime dependencies). Developers can pipe telemetry directly to
  the CLI or call `adapt_event` in code.

### ADR-008: Curated policy pack library and pack resolution
- **Date**: 2026-10-24
- **Status**: Accepted
- **Context**: Compliance and risk teams evaluating agent systems need ready-to-use guardrail
  policies for common regulated workflows (payments, infrastructure changes, customer PII data,
  and support operations). Writing JSON policy rules from scratch creates friction and leads to
  inconsistent rule definitions across organizations.
- **Decision**: Ship curated, version-controlled policy packs inside the package (`assay/packs/`)
  covering `payments`, `production_change`, `customer_data`, and `support_agent`. Enable direct
  resolution via `pack:<name>` or bare name across all policy-aware CLI commands and Python APIs.
  Introduce `assay policy list` and `assay policy show` for offline pack inspection.
- **Consequences**: Evaluators can evaluate live agent traces against enterprise guardrails in
  seconds. Packs are self-contained, canonical JSON documents with deterministic SHA-256 digests.
## Licensing migration (v0.1.0-agpl)

- New versions use Aegean License 1.1 (AL-1.1); versions at or before `v0.1.0-agpl` remain AGPL-3.0.
- Rights holder: AegeanWorks; internal AI commit identity `assistant@local`.
- Counsel review of the 1.1 texts remains pending.

## License notice cleanup (2026-10-26)

- The license-history notes were removed from README.md and COMMERCIAL.md: the project is offered only under the Aegean License 1.1 (AL-1.1), with commercial use under ACL-1.0.

### ADR-009: Assay swarm Stage 3 verified bug fixes (no contract break)
- **Date**: 2026-11-04
- **Status**: Accepted
- **Context**: Parallel analysis (8 modules) returned reproduced defects with file:line and
  failing-first probes. All sampled findings were reproduced by the main session before fixing.
- **Decision**: Land minimal bug fixes only, without changing hashes, digests, schemas or
  exit-code contracts: (1) ledger `signed`/`key_checked` now reflect HMAC validity and blank /
  corrupt lines no longer pollute the signing mix signal; (2) `Event` deep-copies metadata /
  policy on ingress and egress, rejects non-string metadata keys, and raises `ValidationError`
  for unknown `create()` fields; (3) `load_pack` rejects path traversal (`/`, `\`, `..`);
  policy metadata matching requires key presence with strict bool typing; (4) MCP adapter
  preserves `target`; (5) `record --ts` accepts Unix seconds per help text; (6) runtime
  `VERSION` aligned to `0.2.0`; (7) Markdown `_cell` handles `None`, avoids dangling escapes,
  and sanitizes head/digest/ledger-name sinks.
- **Consequences**: JSON pack schema, record hashes, policy digests and exit codes are unchanged.
  `wrong-key` verification now reports `signed:false,key_checked:false` (was `true/true`) and
  blank-line chains stay `signed:true` with `ok:false`. Deferred (filed, not fixed): empty-ledger
  and empty-anchor verdicts, demo `--out` re-run idempotency, seq-gap quarantine, anchor fork
  detection, glob-language documentation, drift coverage for unpinned events, and launcher/CWD
  parity — each needs a contract decision before implementation.

### ADR-010: CLI error-path hardening without contract change
- **Date**: 2026-11-04
- **Status**: Accepted
- **Context**: Second verified batch from the same 8-module swarm. All items reproduced from
  swarm reports before fixing; exit codes and schemas stay fixed.
- **Decision**: Keep exit 0/1/2 and all hashes/digests unchanged while making failures
  actionable: bare `--metadata @` names the empty `@path`; `demo --out <file>` and
  `report --out <dir>` raise `ValidationError` naming `--out`; bare policy typos route through
  `load_pack` suggestions; `Path("pack:name")` resolves like its string form; report thresholds
  render non-numeric values instead of `:g` crash; Markdown packs end with a trailing newline;
  remove dead `cli._value`, `anchor_module`, `AnchorError`, and `Tuple` surface.
- **Consequences**: No JSON, digest, or exit-code change. Operators get named `Why`/`Fix`
  triads instead of OS internals (`.`, `.tmp_` names) or silent drops.

### ADR-011: AL-1.1 is the current license; ADR-001 AGPL decision is superseded
- **Date**: 2026-11-04
- **Status**: Accepted
- **Context**: ADR-001 records AGPL-3.0-or-later as Accepted while the shipped tree offers
  only Aegean License 1.1 with a commercial ACL path (`LICENSE`, `pyproject.toml`,
  `REUSE.toml`, `CONTRIBUTING.md`). Migration notes already sit after ADR-008 but ADR-001
  was never marked superseded, so readers get opposite answers.
- **Decision**: The current license is AL-1.1 (commercial use under ACL-1.0). ADR-001 is
  retained as history and superseded by this record; versions at or before `v0.1.0-agpl`
  remain AGPL-3.0 per the migration notes. Missing in-repo pointers (`licenses/ACL-1.0.md`,
  `licenses/CONTRIBUTING_CLA.md`) stay filed, not fabricated — counsel/maintainer must supply
  them.
- **Consequences**: No code or packaging change. Follow-up stays filed: vendor or link the
  binding ACL/CLA texts so offline evaluators can read the commercial terms.

### ADR-012: Empty evidence fails verification; demo refuses to double its ledger
- **Date**: 2026-11-04
- **Status**: Accepted
- **Context**: An empty ledger verified `ok:true` and an empty anchor journal verified
  `ok:true`, so total truncation looked healthy; re-running `assay demo --out DIR` appended a
  second 5-record chain (10 records, doubled findings) with exit 0.
- **Decision**: Empty ledgers and empty anchor journals verify `ok:false` with explicit
  `empty` findings (verify/review/report exit 1; missing files still exit 2). `run_demo`
  refuses a directory whose `ledger.jsonl` already holds records (`ValidationError`, exit 2).
  Action globs are documented as `fnmatch` (`*`, `?`, `[...]`) in code and README, matching
  the implementation.
- **Consequences**: No hash, digest, or schema change. Operators get a loud empty-evidence
  signal instead of a vacuously passing pack; demo re-runs are explicit, not silent doubles.

### ADR-013: Adapter fidelity without input expansion
- **Date**: 2026-11-04
- **Status**: Accepted
- **Context**: Three reproduced data-loss/inconsistency defects: LangGraph flattened
  `data.input` over `data.output` on key collision; CrewAI stored tool results under `result`
  while LangGraph used `output`; LangGraph left JSON-array `content` as a verbatim string.
- **Decision**: Preserve both colliding values (`input.output` keeps the input side);
  CrewAI stores `output` alongside `result` (existing `result` assertions unchanged);
  array `content` parses through the shared `_as_dict` contract. No new accepted outcomes,
  no new required fields, no exit-code change.
- **Consequences**: Additive only. Multi-framework evidence is comparable without producer
  rewrites; no existing passing input is rejected.

### ADR-014: Error presence requires a truthy value; OpenAI role is not a principal
- **Date**: 2026-11-04
- **Status**: Accepted
- **Context**: Reproduced falseattribution bugs: `role: assistant` became
  `actor: assistant`, and `error: null`/`""` forced `outcome: error` with the falsy value
  stored as evidence.
- **Decision**: Drop `role` from the OpenAI actor alias table (pass `--actor` or include
  `actor`/`agent`); only truthy `error` values force `outcome: error` across OpenAI,
  CrewAI, and LangGraph, otherwise the declared `outcome`/`status` applies and no empty
  error is stored.
- **Consequences**: Rows that relied on `role`-only attribution or falsy-error forcing now
  resolve differently (missing-actor `ValidationError`, or declared-outcome verdict).
  No schema, digest, or exit-code change; no new accepted outcome.

### ADR-015: Demo matches review order; review surface states its limits
- **Date**: 2026-11-04
- **Status**: Accepted
- **Context**: `run_demo` concatenated review then drift findings while `assay review`
  sorts by `(seq, rule_id)`; the no-policy Markdown row claimed `0` matches against a policy
  that was never supplied; the band error message miscounted the outside-band semantics.
- **Decision**: Sort demo findings by `(seq, rule_id)`; render the no-policy matching row
  as `n/a`; reword the band `Why` around outside-band matching; state the full chain gate
  (envelope, seq, prev, event, hash, mac) in the Verification prose.
- **Consequences**: No JSON, digest, or exit-code change. Demo output now matches review
  order by construction.

### ADR-016: Canonical URLs replaced the filed-not-fabricated license pointers
- **Date**: 2026-11-16
- **Status**: Accepted
- **Context**: ADR-011 recorded the ACL/CLA in-repo pointers as filed, not fabricated, with counsel to supply them; commit 51452c5 resolved that follow-up by citing the canonical URLs in LICENSE, its LICENSES mirror and COMMERCIAL.md
- **Decision**: the license record cites https://github.com/AegeanWorks/Aegean-Licensing/blob/master/licenses/ACL-1.0.md and CONTRIBUTING_CLA.md; ADR-011 is retained as history and its follow-up is resolved
- **Consequences**: offline evaluators follow the citation to the hosted canonical text; the recorded-gone state stays honest; no code or packaging change
