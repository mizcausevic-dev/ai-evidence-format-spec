# AI Evidence Format

A draft specification for **machine-readable evidence objects** that travel with every claim an answer engine produces.

When an LLM says *"Cambridge is in Massachusetts, source: en.wikipedia.org/wiki/Cambridge,_Massachusetts"*, the answer is two things: the **claim** and the **evidence**. Today the evidence is unstructured — a URL, maybe a quoted span, maybe nothing. The AI Evidence Format makes it structured: source identity, span selector, retrieval confidence, freshness, content hash, and a declared synthesis role.

## The three pillars

| Pillar | What it does |
|---|---|
| **Attach** | Every cited claim carries one or more evidence objects in a defined format |
| **Verify** | Each evidence object carries a content hash (and optional signature) so consumers can detect tampering or staleness |
| **Synthesize** | Each object declares its role in the answer — `supporting`, `contradicting`, `partial`, `background` — making "the model cited two sources that disagree" a first-class fact |

## Why not just a URL?

A URL in a footnote tells you where to look. It does not tell you:
- **Which span** of the source was used (the page is 12,000 words; which sentence?)
- **How confident** the retrieval system was that this was relevant
- **How fresh** the content was when retrieved
- **Whether the content hash matches** what the model actually consumed (the page may have changed)
- **What role** the evidence played — did the model use it to support, contradict, or only as background?

The AI Evidence Format makes each of those answerable in a single JSON object.

## Why not Schema.org `Citation` or W3C Annotations?

Those vocabularies describe citations as documents. The AI Evidence Format describes citations as *retrieval artifacts in a generative pipeline*. The differences:

- We need retrieval method and confidence (vector / keyword / graph / hybrid)
- We need freshness at the moment of retrieval (not document publication date)
- We need synthesis role (the model's *intended use* of the evidence)
- We need a content hash that lets a consumer verify the model read what it claims to have read

The format **reuses** Schema.org wherever it fits, but it is not a subset of any existing vocabulary.

## Quickstart

1. For each claim your answer engine produces, build one or more evidence objects conforming to [`evidence.schema.json`](evidence.schema.json).
2. Compute a `content_hash` over the canonicalized bytes of the cited span. (See §5 of [`SPEC.md`](SPEC.md) for the canonicalization rules.)
3. Either embed the evidence inline in your answer payload, or publish it at a URI and reference it.
4. Pair with an [AEO Protocol](https://github.com/mizcausevic-dev/aeo-protocol-spec) declaration on the source origin to give consumers an authoritative anchor.

## Files in this repo

- [`SPEC.md`](SPEC.md) — full v0.1 specification
- [`evidence.schema.json`](evidence.schema.json) — JSON Schema (draft 2020-12)
- [`examples/`](examples/) — reference documents for a supporting citation, a contradicting citation, and a background-only citation

## Status

**v0.1 draft.** Issues and pull requests welcome.

## License

AGPL-3.0. Specification text is freely implementable; schema and examples are AGPL-3.0.

## Related work

Part of the **Kinetic Gain Protocol Suite** — open specifications for entity declaration, agent capability disclosure, prompt lineage, MCP tool disclosure, and citation evidence in the answer-engine era. See [github.com/mizcausevic-dev](https://github.com/mizcausevic-dev).

---

**Connect:** [LinkedIn](https://www.linkedin.com/in/mirzacausevic/) · [Kinetic Gain](https://kineticgain.com) · [Medium](https://medium.com/@mizcausevic/) · [Skills](https://mizcausevic.com/skills/)
