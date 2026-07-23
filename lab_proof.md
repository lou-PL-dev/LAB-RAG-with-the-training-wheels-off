# Lab Proof — RAG with the Training Wheels Off

## Pipeline overview

- Loaded 2 source PDFs from `input/` (EU HLEG Ethics Guidelines, UNESCO AI Ethics Recommendation)
- Split into 398 overlapping chunks (chunk_size=500, overlap=50 characters)
- Generated embeddings using OpenAI's `text-embedding-3-small` (1536 dimensions)
- Retrieval via cosine similarity, top-k=3
- Answer generation via `gpt-4o-mini`, grounded strictly in retrieved context

## Case 1: Successful grounded retrieval

**Query:** What are the requirements for human oversight of AI systems?

**Retrieved evidence:**

1. Score: 0.7094 | Source: `ai_hleg_ethics_guidelines_for_trustworthy_ai-en_87F84A41-A6E8-F38C-BFF661481B40077B_60419.pdf`
   > to be subject to a decision based solely on automated processing when this 
produces legal effects on users or similarly significantly affects them.36 
Human oversight. Human oversight helps ensuring ...

2. Score: 0.7027 | Source: `ai_hleg_ethics_guidelines_for_trustworthy_ai-en_87F84A41-A6E8-F38C-BFF661481B40077B_60419.pdf`
   > gations, both as regards horizontally applicable rules as well as domain -specific 
regulation. 
In the following paragraphs, each requirement is explained in more detail.  
 
1.1 Human agency and ove...

3. Score: 0.6715 | Source: `ai_hleg_ethics_guidelines_for_trustworthy_ai-en_87F84A41-A6E8-F38C-BFF661481B40077B_60419.pdf`
   > es? 
 Did you take safeguards to prevent overconfidence in or overreliance on the AI system for work 
processes? 
Human oversight: 
 Did you consider the appropriate level of human control for the p...

**Final answer:** The requirements for human oversight of AI systems include:

1. Ensuring that AI systems support human autonomy and decision-making.
2. Implementing governance mechanisms such as human-in-the-loop (HITL), human-on-the-loop (HOTL), or human-in-command (HIC) approaches to allow for human intervention.
3. Considering the appropriate level of human control for the particular AI system and use case.
4. Describing the level of human control or involvement, including identifying who is the “human in control” and the moments or tools for human intervention.
5. Establishing mechanisms and measures to ensure human control or oversight.


## Case 2: Failure case (out-of-scope query)

**Query:** What is the best recipe for lasagna?

**Retrieved evidence:**

1. Score: 0.1327 | Source: `ai_hleg_ethics_guidelines_for_trustworthy_ai-en_87F84A41-A6E8-F38C-BFF661481B40077B_60419.pdf`
   > lues and (3) it should be r obust, 
both from a technical and social perspective since to ensure that, even with good intentions, AI systems do not 
cause any unintentional harm. Each component is nec...

2. Score: 0.1256 | Source: `ai_hleg_ethics_guidelines_for_trustworthy_ai-en_87F84A41-A6E8-F38C-BFF661481B40077B_60419.pdf`
   > and New Technologies (“EGE”) 
proposed a set of 9 basic principles, based on the fundamental values laid down in the EU Treaties and Charter. 24 
We build  further on this work, recognising most of th...

3. Score: 0.1225 | Source: `ai_hleg_ethics_guidelines_for_trustworthy_ai-en_87F84A41-A6E8-F38C-BFF661481B40077B_60419.pdf`
   > group-artificial-intelligence).  
The reuse policy of European Commission documents is regulated by Decision 2011/8 33/EU (OJ L 330, 14. 12.2011, p.39). 
For any use or reproduction of photos or other...

**Final answer:** I don't know.


## Grounding risk / limitation

Initial hypothesis: corpus imbalance (the EU HLEG document accounts for
~87% of all chunks — 156k of 178k total characters) would cause the
smaller UNESCO document to be structurally under-retrieved.

This hypothesis was tested directly and **did not hold up**. Queries using
UNESCO-specific terminology (e.g. "Readiness Assessment Methodology",
"AI Ethics Experts Without Borders network", "social scoring or mass
surveillance") consistently retrieved UNESCO chunks in the top position,
despite being outnumbered ~7:1 by HLEG chunks. Cosine similarity correctly
prioritized relevance over source volume.

A harder test surfaced the **actual** grounding risk: short factual details
embedded inside thematically broader sentences are hard to retrieve, regardless
of source document. For the query "How many Member States adopted the
Recommendation?", the chunk containing the correct answer ("adopted by all
193 Member States in November 2021") ranked **12th out of 398** chunks
(score 0.336), well below the ~0.65-0.75 scores seen for well-matched
conceptual queries. The reason: the embedding represents the *whole
sentence's* meaning (UNESCO publishing a standard amid urgency and
marginalized groups), not the isolated fact the query is asking for, so a
query targeting one specific detail competes against, and loses to, chunks
whose overall topic is a closer thematic match.

A test increasing the surrounding context window (250 → 1000
characters) only raised the score modestly (0.336 → 0.365), confirming
chunk size is a contributing factor but not the dominant cause. The
underlying issue: cosine similarity matches on overall sentence meaning,
so a query targeting one precise fact competes against and loses to
chunks whose broader topic is a closer thematic match. Two low-effort
fixes would likely help: hybrid search (adding keyword/BM25 matching
alongside semantic search, so an exact term like "Member States" is
caught even when embedding similarity is weak), and reranking (retrieving
a wider top-k and letting a cross-encoder rerank it — the correct chunk
was already at rank 12, just outside a narrow top-3 window).
