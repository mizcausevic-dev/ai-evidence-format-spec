# AI Evidence Format v0.1 — Specification

**Status:** Draft
**Version:** 0.1.0
**Editor:** Miz Causevic
**License:** AGPL-3.0 (this document, schema, and examples). Implementations are unrestricted.

RFC 2119 keywords apply throughout.

---

## 1. Scope

This specification defines a JSON document format for **evidence objects** — structured citations that travel with claims produced by LLM-based answer engines.

The format is designed for:
- Retrieval-augmented generation (RAG) pipelines
- Agentic systems that browse and synthesize
- LLM-as-judge evaluation harnesses
- Audit and incident-response tooling

The spec does **not** standardize:
- The retrieval method itself (vector search, keyword, graph, hybrid)
- The chunking strategy for source documents
- The format of the answer the evidence accompanies

## 2. Terminology

- **Claim** — an assertion in a synthesized answer.
- **Evidence** — a structured citation supporting, contradicting, or contextualizing a claim.
- **Source** — the document or system that produced the cited content.
- **Span** — the specific portion of the source that was used.
- **Synthesis role** — the declared function of the evidence in the synthesized answer.

## 3. The three pillars

### 3.1 Attach

Every claim in a synthesized answer **SHOULD** be accompanied by zero or more evidence objects. Zero is acceptable for claims the answer engine declares as common-knowledge (and which conformance Level 2+ must explicitly mark).

Evidence objects **MAY** be embedded inline in the answer payload or referenced by URI. The spec does not mandate one or the other.

### 3.2 Verify

Every evidence object **MUST** include a `verification.content_hash` over the canonicalized bytes of the cited span. Consumers **MAY** fetch the source independently and recompute the hash to confirm what the model actually used.

Evidence objects **MAY** additionally include `verification.signature` — a JWS signed by the retrieval pipeline or the source publisher. Signatures are advisory; their absence does not invalidate the evidence.

### 3.3 Synthesize

Every evidence object **MUST** declare a `synthesis_role`. The role is the answer engine's stated intent for the evidence in this specific answer:

| Role | Meaning |
|---|---|
| `supporting` | Evidence directly supports the claim. |
| `contradicting` | Evidence contradicts the claim. The engine surfaced it anyway — for transparency or to acknowledge disagreement in sources. |
| `partial` | Evidence supports part of the claim. The model synthesized across multiple sources. |
| `background` | Evidence is contextual but not load-bearing. Cited for orientation. |

Multiple evidence objects per claim are common; their roles are independent.

## 4. Document structure

### 4.1 `evidence_version` (required)

A semver string. **MUST** be `"0.1"` for documents conforming to this draft.

### 4.2 `evidence_id` (required)

A stable identifier for this evidence object. Used by audit tooling to track citation usage across answers.

### 4.3 `claim_text` (required)

The exact text of the claim the evidence accompanies. Quoted verbatim from the synthesized answer; **MUST NOT** be paraphrased.

### 4.4 `source` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `uri` | URI | yes | Canonical URI of the source document or system. |
| `type` | enum | yes | `document` / `webpage` / `api_response` / `knowledge_graph` / `aeo_declaration` / `dataset` / `database_row`. |
| `title` | string | no | Human-readable title of the source. |
| `publisher` | string | no | Publishing entity. |
| `published_at` | datetime | no | Original publication time, if known. |
| `fetched_at` | datetime | yes | When the retrieval pipeline obtained the content. |

### 4.5 `span` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `selector_type` | enum | yes | `byte_range` / `line_range` / `xpath` / `css` / `text_quote` / `paragraph_index` / `none`. |
| `selector_value` | string | yes when `selector_type` ≠ `none` | The selector value (e.g. `1024-2048` for byte_range). |
| `exact_text` | string | no | The exact text content of the span at fetch time. RECOMMENDED for human-verifiable audit. |
| `surrounding_context` | string | no | Optional surrounding text to disambiguate the span if the source changes. |

### 4.6 `retrieval` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `method` | enum | yes | `vector` / `keyword` / `graph` / `hybrid` / `direct_fetch` / `model_recall`. |
| `confidence` | number | no | 0.0–1.0. Spec does not mandate calibration. |
| `rank` | integer | no | The rank of this evidence in the retrieval result set. |
| `freshness_age_seconds` | integer | no | Age of the content at the moment of retrieval, in seconds. |
| `retriever_id` | string | no | Identifier of the retrieval pipeline / component. |

### 4.7 `verification` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `content_hash` | string | yes | `sha256:<hex>` of the canonicalized span bytes. See §5. |
| `signature` | string | no | Detached JWS over the canonical JSON form of the rest of the document. |
| `signing_key_uri` | URI | no | Location of the JWK used to sign. |
| `signed_by` | string | no | Identifier of the signer (retrieval pipeline, source publisher, etc.). |

### 4.8 `synthesis_role` (required)

One of `supporting` / `contradicting` / `partial` / `background`. See §3.3.

### 4.9 `notes` (optional)

Free-form notes from the answer engine — e.g. "Source contradicts X but is the only authoritative source on Y." Strictly informational.

## 5. Canonicalization rules

For `content_hash`:

1. Read the span content as UTF-8.
2. Normalize line endings to LF (`\n`).
3. Strip a single trailing newline.
4. Compute SHA-256.
5. Encode as lowercase hex, prefixed `sha256:`.

For signed evidence objects, the JWS signs the *canonical JSON form* of the document with `verification.signature` and `verification.signing_key_uri` omitted. Canonical JSON is defined as RFC 8785 (JSON Canonicalization Scheme, JCS).

## 6. Conformance levels

| Level | Requirements |
|---|---|
| **Level 1 — Attach** | Schema-valid document with all required fields. |
| **Level 2 — Verify** | Level 1, plus `verification.content_hash` is reproducible — a consumer fetching the source can recompute the hash and match. |
| **Level 3 — Sign** | Level 2, plus `verification.signature` is a valid JWS over the canonical form. |

## 7. Security and privacy considerations

- **Hash and source drift.** A `content_hash` mismatch on re-fetch indicates the source has changed — not necessarily that the evidence is invalid. Consumers **SHOULD** treat mismatch as a signal to re-evaluate, not as proof of bad faith.
- **`exact_text` and PII.** Including `exact_text` is helpful for audit but may copy personal data into the evidence object. Pipelines **SHOULD** apply PII redaction before populating `exact_text`.
- **Signature spoofing.** A signed evidence object proves *who issued the citation*, not *that the citation is accurate*. A malicious retriever can sign garbage. Consumers requiring high assurance **SHOULD** pair signed evidence with reproducible content hashes against trusted sources.
- **Synthesis role honesty.** The format permits an answer engine to declare a role that does not match the actual use of the evidence. The format makes such dishonesty *detectable* by audit tooling, not impossible.

## 8. Relationship to existing work

| Standard | Relationship |
|---|---|
| **W3C Web Annotations** | Borrows the selector / target distinction; differs by adding retrieval and synthesis semantics. |
| **Schema.org `Citation`** | Companion. AI Evidence Format is finer-grained and includes runtime retrieval metadata. |
| **JSON-LD** | Compatible. An AI Evidence object can be JSON-LD by adding `@context`; the spec does not require it. |
| **AEO Protocol** ([aeo-protocol-spec](https://github.com/mizcausevic-dev/aeo-protocol-spec)) | `source.type: aeo_declaration` lets evidence cite an AEO document directly, enabling end-to-end entity-grounded citation. |
| **Prompt Provenance** ([prompt-provenance-spec](https://github.com/mizcausevic-dev/prompt-provenance-spec)) | An evidence object may reference the prompt-provenance record of the prompt that produced the citation. |

## 9. Open questions

- **Embedded vs referenced evidence.** Should the spec recommend one over the other for performance reasons?
- **Multi-claim evidence.** Should one evidence object be permitted to support multiple claims? Current draft is one-to-one; a single source span supporting two claims requires two evidence objects.
- **Negative evidence.** The format supports `contradicting`, but should it also support `absence_evidence` ("the source does not mention X")?
- **Streaming.** How does evidence attach to streamed token output?
