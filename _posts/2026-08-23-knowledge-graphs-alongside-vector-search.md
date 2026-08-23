---
layout: post
title: "Why I keep reaching for knowledge graphs alongside vector search"
date: 2026-08-23 09:00:00 +0000
categories: [retrieval, knowledge-graphs]
---

Vector search is the default retrieval story for a reason. It's fast, it's language-agnostic, it degrades gracefully, and it composes cleanly with everything else in a modern stack. If you can only have one retrieval mechanism, embed things and search over them.

But every non-trivial system I've worked on eventually grows a knowledge graph next to the vector index. Not instead of. Alongside. Here's why that keeps happening.

## Vectors are good at similarity, not structure

An embedding tells you two chunks of text are semantically close. It does not tell you that one is the parent of the other, that one supersedes the other, that one is only valid in a specific region, or that two documents talk about the same underlying entity under different names.

Domain vocabulary is full of that kind of structure. Regulations reference other regulations. Products have variants. Equipment has parents and children. Clients have contracts that override defaults. If you flatten all of that into text and embed it, you throw away the exact information the retrieval was supposed to preserve.

## Hybrid retrieval isn't "vector plus keyword"

The usual framing of hybrid retrieval is dense-plus-sparse — embed the query, also run BM25, blend the scores. That's fine. It's not the interesting version.

The interesting version is: use the graph to *constrain* the vector search. When someone asks a question about a specific piece of equipment operating in a specific jurisdiction, you don't want the top-k chunks from the entire corpus. You want the top-k chunks from documents that the graph says are relevant to this equipment, in this jurisdiction, currently in force. The graph narrows the search space; the vectors rank inside it.

That composition is what actually reduces hallucinations in practice, more than any prompt engineering.

## Ontologies force you to make the domain explicit

Nobody enjoys writing an ontology. It's the least glamorous work in AI. But the act of writing one is where you find out what the business actually means by its own vocabulary — where terms overlap, where they contradict each other, where the same word means two different things to two different teams.

You can't do this with prompts. Prompts encode assumptions; they don't surface disagreements. An ontology forces the disagreements into the open, which is exactly where you want them if you're building something that has to be right.

## The trade-off is honest

Graphs are more work to build and more work to maintain. They need governance. They need someone who cares whether the entity for "Generator Model X" gets merged with "Model X Generator" when the two arrive from different pipelines.

If your use case is genuinely open-domain — search the whole web, help me find any document — a graph is probably overkill. If your use case is bounded and the correctness bar is high, a graph pays for itself surprisingly quickly. Enterprise retrieval almost always falls in the second bucket.

---

*Next: what a four-layer context stack actually looks like when you have to build one.*
