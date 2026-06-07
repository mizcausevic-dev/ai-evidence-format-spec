# Changelog

All notable changes to this project are documented here.

## [1.0.1] - 2026-06-07

### Specified
- Rewrote **SPEC.md §5** into a normative, reproducible `content_hash` canonicalization: decode character references on markup sources → collapse runs of Unicode `White_Space` to a single space → trim → SHA-256 → lowercase hex with a `sha256:` prefix. The previous §5 described only line-ending normalization, which diverged from how conforming implementations actually hash text.
- Defined the exact whitespace set (Unicode `White_Space`; U+200B ZERO WIDTH SPACE preserved) and stated the verifier rule `content_hash == sha256(C(span.exact_text))`, noting `C` is idempotent so producers may store already-canonical `exact_text`.
- Added **§5.1.1** declaring Unicode NFC normalization **RECOMMENDED** (not required) for v0.1, because some runtimes lack a normalizer by default (e.g. PHP's `intl` extension on shared hosts); flagged NFC as a candidate `MUST` for a future version.
- Added **§5.3 test vectors** — three SHA-256 vectors reproducible with any tool (`printf … | sha256sum`) and cross-checked against the WordPress reference implementation, plus an informative NFC example.

### Made verifiable
- Replaced the placeholder `content_hash` values in all three `examples/` documents with real digests computed by the §5.1 rules, so each example is now an independently verifiable Level 2 ("Verify") object.

### Why this mattered
- The content hash is the spec's trust primitive. A canonicalization a third party cannot reproduce — or examples carrying invented digests — quietly breaks the "Verify" pillar. This revision makes "recompute the hash and match" hold end to end, against both `sha256sum` and the reference implementation.

## [1.0.0] - 2026-05-12

### Released
- Released **ai-evidence-format-spec** publicly as a reviewable operating system for ai retrieval reliability.
- Packaged the current implementation, documentation, validation flow, and proof surfaces into a repo that can be reviewed by technical and operating stakeholders.
- Clarified the core problem the project is addressing: retrieval drift, citation breakdowns, and rising hallucination risk as corpora and prompts evolve.

### Why this mattered
- Existing approaches in vector tooling, LLM observability stacks, and evaluation suites were useful for parts of the workflow.
- They still left out a durable operator workflow for evidence quality, source freshness, and trust decisions.
- This release made the repo read like an operational capability rather than a narrow technical demo.

## [0.1.0] - 2026-02-22

### Shipped
- Cut the first coherent internal version of **ai-evidence-format-spec** with stable domain objects, review surfaces, and decision outputs.
- Established the first reviewable version of the architecture described as: AI Evidence Format v0.1 draft. JSON document format for structured citations that travel with LLM-generated claims: source identity, span selector, retrieval confidence, freshness, content hash, declared synthesis role. Part of the Kinetic Gain Protocol Suite.
- Focused the repo around actionability instead of passive reporting.

## [Prototype] - 2025-02-16

### Built
- Built the first runnable prototype for the repo's main workflow and decision model.
- Validated the concept against pressure points such as RAG hallucination rates, stale retrieval context, and citation-quality breakdowns.
- Used the prototype phase to test whether the project could drive action, not just present information.

## [Design Phase] - 2022-12-14

### Designed
- Defined the system around operator-first and decision-legible outputs.
- Chose interfaces and examples that made sense for AI platform, search, and knowledge-system teams.
- Avoided reducing the project to a generic dashboard, CRUD app, or fashionable wrapper around the stack.

## [Idea Origin] - 2022-02-14

### Observed
- The original idea surfaced while looking at how teams were handling retrieval drift, citation breakdowns, and rising hallucination risk as corpora and prompts evolve.
- The recurring pattern was that teams had data and tools, but still lacked a usable operating layer for the hardest decisions.

## [Background Signals] - 2022-08-09

### Context
- Earlier platform, governance, and operator-tooling work made one pattern hard to ignore: the systems that create the most drag are often the ones with partial controls and weak operational coherence, not the ones with no controls at all.
- That pattern shaped the thinking behind this repo well before the public version existed.