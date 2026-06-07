# AI Evidence Format v0.1 — Specification

**Status:** Draft
**Version:** 0.1.1
**Editor:** Miz Causevic
**License:** MIT — the specification text, schema, and examples may be freely implemented, extended, and redistributed with attribution. Reference implementations (such as [mcp-kinetic-gain](https://github.com/mizcausevic-dev/mcp-kinetic-gain)) are licensed separately under AGPL-3.0.

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

Every evidence object **MUST** include a `verification.content_hash` over the canonical text of the cited span (§5.1). Consumers **MAY** fetch the source independently, re-apply the canonicalization, and recompute the hash to confirm what the model actually used.

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
| `content_hash` | string | yes | `sha256:<hex>` of the canonical span text. See §5.1. |
| `signature` | string | no | Detached JWS over the canonical JSON form of the rest of the document. |
| `signing_key_uri` | URI | no | Location of the JWK used to sign. |
| `signed_by` | string | no | Identifier of the signer (retrieval pipeline, source publisher, etc.). |

### 4.8 `synthesis_role` (required)

One of `supporting` / `contradicting` / `partial` / `background`. See §3.3.

### 4.9 `notes` (optional)

Free-form notes from the answer engine — e.g. "Source contradicts X but is the only authoritative source on Y." Strictly informational.

## 5. Canonicalization rules

### 5.1 `content_hash`

`verification.content_hash` is the string `sha256:` followed by the lowercase hexadecimal SHA-256 digest of the **canonical text of the cited span**.

The canonical text is produced by applying the following steps, in order, to the span content:

1. **Decode as UTF-8.** Treat the span content as a UTF-8 string.
2. **Resolve character references.** When the span was extracted from a markup source (HTML/XML), decode character references so the hash is over readable text rather than escapes — at minimum the five predefined XML entities (`&amp;` `&lt;` `&gt;` `&quot;` `&apos;`) and their numeric equivalents (`&#38;` and so on). For plain-text or JSON sources there is nothing to decode and this step is a no-op. Implementations **MUST NOT** expand the broader HTML named-entity set at this step (e.g. `&nbsp;` is left intact), so the result is deterministic across runtimes.
3. **Collapse whitespace.** Replace every maximal run of Unicode `White_Space` characters with a single U+0020 SPACE.
4. **Trim.** Remove leading and trailing U+0020 (equivalently, trim the surrounding whitespace produced by step 3).
5. **Normalize residual line endings (legacy).** Convert any remaining CRLF or CR to LF and strip one trailing LF. After steps 3–4 this is inert; it is retained so that implementations which hash lightly-processed text still converge on the same digest.
6. **Hash.** Compute SHA-256 over the UTF-8 bytes of the canonical text, encode as lowercase hex, and prefix `sha256:`.

Call the steps 1–5 transform `C`. A verifier **MUST** be able to reproduce the digest as `sha256:` + hex( SHA-256( UTF-8( C(`span.exact_text`) ) ) ). `C` is **idempotent** — `C(C(x)) == C(x)` — so a producer **MAY** instead store the already-canonical string in `span.exact_text`, in which case the verifier's `C` is a no-op and the digest is simply `sha256:` + hex( SHA-256( UTF-8(`span.exact_text`) ) ). The hash is taken over the **text**, never over surrounding markup, the JSON document, or any selector value.

> **Self-quoting evidence.** When a claim is its own span — a publisher disclosing the provenance of text it authored, rather than citing an external source — `claim_text` and `span.exact_text` hold the same canonical string and either **MAY** be used as the hash input. The WordPress reference implementation ([ai-evidence-block](https://github.com/mizcausevic-dev/ai-evidence-block)) operates in this mode.

> **The `White_Space` set (step 3).** "Unicode `White_Space`" means the code points carrying the Unicode `White_Space` property: U+0009–U+000D, U+0020, U+0085, U+00A0, U+1680, U+2000–U+200A, U+2028, U+2029, U+202F, U+205F, U+3000. This **includes** the no-break space (U+00A0), the em space (U+2003), and the line/paragraph separators, and **excludes** U+200B ZERO WIDTH SPACE (Unicode category `Cf`, not whitespace), which is preserved. A regex `\s` in Unicode mode (e.g. PCRE2 `/\s/u`) matches exactly this set; engines whose default `\s` is ASCII-only **MUST** widen it to the set above.

#### 5.1.1 Unicode normalization (NFC) — RECOMMENDED

Producers **SHOULD** apply Unicode Normalization Form C (NFC) to the canonical text before step 6. Without it, visually identical text in different normalization forms hashes differently — for example `café` written with a precomposed U+00E9 versus `e` + combining acute U+0301 (worked example in §5.3).

NFC is **RECOMMENDED but not REQUIRED** in v0.1. Some runtimes do not ship a Unicode normalizer in their default install — PHP's `Normalizer` requires the `intl` extension, which is absent on many shared hosts — so mandating NFC would make conformance host-dependent. A future spec version is expected to promote NFC to a **MUST** once a portable fallback is specified. Until then:

- A producer that can normalize **SHOULD** emit NFC text; the `span.exact_text` / `claim_text` it publishes is then also NFC.
- A verifier that sees a hash mismatch **SHOULD** apply NFC to both sides before concluding the content differs, since the difference may be normalization form alone.

### 5.2 Signed objects

For signed evidence objects, the JWS signs the *canonical JSON form* of the document with `verification.signature` and `verification.signing_key_uri` omitted. Canonical JSON is defined as RFC 8785 (JSON Canonicalization Scheme, JCS).

### 5.3 Test vectors

Each vector lists the raw span input, the canonical text after §5.1 steps 1–5, and the resulting `content_hash`. All three use pure ASCII / basic-entity input, so they reproduce with any SHA-256 tool — e.g. `printf '%s' '<canonical text>' | sha256sum` — independent of language or the reference implementation.

| # | Raw span input | Canonical text | `content_hash` |
|---|---|---|---|
| 1 | `·· The·· sky⇥is⏎blue. ··` — leading/trailing spaces, doubled spaces, a tab (`⇥`) and a newline (`⏎`), shown here as `·` `⇥` `⏎` | `The sky is blue.` | `sha256:52ae4fd504d855f1dd094ab54e2da8402c96250ef75ef775a3cf20ff39d0cb2b` |
| 2 | `Tom &amp; Jerry said &quot;hi&quot;` | `Tom & Jerry said "hi"` | `sha256:7b1677fdf0e83117a316f9958946440011ec6cf9920c9ef4dad97562b6781103` |
| 3 | `The Eiffel Tower is 330 metres tall.` | (unchanged) | `sha256:22d55b4f4a196374a8444337fc5005edc190b7724912c00e0ecf7b5c22201e67` |

> **NFC example (informative).** The string `café` yields a different digest depending on normalization form: precomposed (`caf` + U+00E9) → `sha256:850f7dc43910ff890f8879c0ed26fe697c93a067ad93a7d50f466a7028a9bf4e`; decomposed (`cafe` + U+0301) → `sha256:81ef060bcd98adc7824eb5c1ada83c32491b16018e11e79f00ab9d09e04b015a`. Applying NFC (§5.1.1) makes both produce the precomposed digest. Illustrative only; not a v0.1 conformance requirement.

The reference implementation's `includes/class-claim.php::normalize_text()` and `includes/helpers.php::content_hash()` implement §5.1, and every `content_hash` in [`examples/`](examples/) is computed by these rules.

## 6. Conformance levels

| Level | Requirements |
|---|---|
| **Level 1 — Attach** | Schema-valid document with all required fields. |
| **Level 2 — Verify** | Level 1, plus `verification.content_hash` is reproducible — a consumer fetching the source and applying §5.1 canonicalization can recompute the hash and match. |
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
