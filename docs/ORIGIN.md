# Why We Built This

**ai-evidence-format-spec** started from a recurring problem in ai retrieval reliability: teams had more signal than operational clarity. That difference between visibility and usability kept showing up under pressure.

The recurring pressure in this space showed up around retrieval drift, citation breakdowns, and rising hallucination risk as corpora and prompts evolve. In practice, that meant teams could collect logs, metrics, workflow state, documents, or events and still not have a good answer to the hardest questions: what is drifting, what matters first, who owns the next move, and what evidence supports that move? Once a system reaches that point, the problem is no longer only technical. It becomes operational.

That is why **ai-evidence-format-spec** was built the way it was. The repo is a deliberate attempt to model a real operating layer for AI platform, search, and knowledge-system teams. It is not just trying to present data attractively or prove that a stack can be wired together. It is trying to show what happens when evidence, prioritization, and next-best action are treated as first-class product concerns.

Existing tools helped with adjacent workflows. vector tooling, LLM observability stacks, and evaluation suites covered storage, reporting, scanning, or execution in pieces. What they still missed was a durable operator workflow for evidence quality, source freshness, and trust decisions. That left operators reconstructing the story manually at exactly the moment they needed clarity.

That shaped the design philosophy:

- **operator-first** so the riskiest or most time-sensitive signal is surfaced early
- **decision-legible** so the logic behind a recommendation can be understood by humans under pressure
- **review-friendly** so the repo supports discussion, governance, and iteration instead of hiding the reasoning
- **CI-native** so checks and narratives can live close to the build and change process

This repo also avoids trying to be a vague platform for everything. Its value comes from being opinionated about a real problem: AI Evidence Format v0.1 draft. JSON document format for structured citations that travel with LLM-generated claims: source identity, span selector, retrieval confidence, freshness, content hash, declared synthesis role. Part of the Kinetic Gain Protocol Suite.

What comes next is practical. The roadmap is about deeper release diffs, stronger collection history, and clearer evidence export for AI governance reviews. The long-term value of **ai-evidence-format-spec** is that it makes that operating layer concrete enough to review, improve, and trust.