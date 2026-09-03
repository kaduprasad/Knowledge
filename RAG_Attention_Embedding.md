# NLP Endsem — Quick Revision Notes: RAG, Attention, Contextual Embeddings

> Grounded in: NLP_all PDF (pages cited), Session 11 slides, and past papers — **Feb 2026 Endsem (Q4, Q6)**, **Mar 2026 Makeup (Q4, Q5, Q6)**, **Oct 2024 Endsem (Q6)**. All three papers tested these topics — expect ~12–18 marks combined.

---

## 1. Contextual Embeddings (PDF p20, 158–178)

### Definition (asked verbatim — Feb 2026 Q4a, 1.5 marks)
- A contextual embedding is a **vector representation of a word that depends on the sentence it appears in** — the *same word gets different vectors in different contexts*.
- Computed by running the whole sequence through **attention layers** (e.g., BERT encoder); each token's output vector is a function of *all* tokens around it.

### Static vs Contextual (asked verbatim — Feb 2026 Q4b, 1.5 marks)
| | Static (word2vec, GloVe) | Contextual (ELMo, BERT) |
|---|---|---|
| Vectors per word | **One fixed vector** for all contexts | **Different vector per occurrence** |
| Polysemy | Cannot distinguish "bank" (river/finance) | Disambiguates by context |
| Computation | Lookup table after training | Full model forward pass at inference |
| Training | Skip-gram/CBOW/co-occurrence counts | Masked LM + NSP (BERT), bidirectional |
| Cost | Cheap, fast | Expensive (GPU, large models) |

### BERT essentials (p175–177)
- **Encoder-only** transformer; bidirectional self-attention (sees left + right context).
- Pre-training tasks: **MLM** (mask 15% of tokens, predict them) + **NSP** (are two sentences adjacent?).
- BERT-base: 12 layers, 768-dim, 12 heads, 110M params; BERT-large: 24/1024/16/340M.
- Fine-tune for classification, NER, QA; use `[CLS]` for sentence-level tasks.

### Limitations
- Expensive inference (no simple lookup); fixed max sequence length (512 for BERT).
- Encoder-only models **cannot generate text** (use decoder-only GPT for generation).
- Domain shift: pre-trained on general text — needs fine-tuning for medical/legal.

### When to use what
| Scenario | Use |
|---|---|
| Fast similarity lookup, small compute, analogies | Static (word2vec/GloVe) |
| WSD, sentence classification, NER, QA — meaning depends on context | Contextual (BERT) |
| Text generation / completion | Decoder-only (GPT-style) |
| Translation / summarization (seq→seq) | Encoder-decoder (T5-style) |

### 🔥 Exam pattern (Feb 2026 Q4c — 3 marks)
Contextual embedding of a word = **weighted sum of value vectors**: `Output = Σ αᵢ·vᵢ`
Given α = [0.2, 0.5, 0.3], v_the=[1,0], v_cat=[1,1], v_sat=[0,1]:
`0.2[1,0] + 0.5[1,1] + 0.3[0,1] = [0.7, 0.8]` ✅ Just multiply-and-add — show the setup line for the method mark.

---

## 2. Attention & Self-Attention (PDF p162–173)

### Why attention exists (theory marks)
- RNN encoder-decoder squeezes everything into **one fixed context vector → bottleneck** (p162).
- Attention lets the decoder **look at ALL encoder hidden states**, weighted by relevance, at every step.
- Also mitigates **vanishing gradients** (direct connections) and enables **parallelism** (transformers drop recurrence entirely).

### The 3-step recipe (memorize — p163)
**Attention = Score → Softmax → Weighted Sum**
1. **Score**: similarity between query and each key — dot product `score(q, kᵢ) = q·kᵢ`
2. **Softmax**: normalize scores into weights αᵢ (sum to 1)
3. **Weighted sum**: `output = Σ αᵢ·vᵢ`

### Self-attention formula (p165)
$$\text{SelfAttention}(Q,K,V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$
- Q = queries, K = keys, V = values — all three are **linear projections of the same input sequence** (Q=XW_Q etc.).
- `√d_k` scaling prevents softmax saturation for large dimensions.
- **Self-attention**: Q,K,V from same sentence. **Cross-attention**: decoder query attends to encoder keys/values (translation, p169).
- **Masked self-attention** (decoder): each position can only attend to previous positions — needed for autoregressive generation (p173).

### Multi-head attention (p168) — asked Mar 2026 Q4 (6 marks)
- **Why multiple heads (1.5 marks)**: different heads capture **different relation types simultaneously** (syntax, coreference, position); a single head averages everything into one pattern.
- Mechanics: run h attention heads in parallel → **concatenate outputs** → multiply by **output projection W_O**.

### 🔥 Exam pattern (Mar 2026 Q4b–c)
head₁=[0.7,0.8], head₂=[0.6,0.9] → concat = [0.7, 0.8, 0.6, 0.9] → multiply by W_O (4×2) → final [0.84, 1.07].
Head specialization (1 mark): look at which position gets highest weight per head (e.g., head 1 → verb, head 2 → determiner).

### Limitations of (self-)attention
- **O(n²) cost** in sequence length — long documents are expensive.
- No inherent word-order awareness → needs **positional encoding** (p170–171).
- Attention weights ≠ full explanation (interpretability is partial).

### When to use what
| Scenario | Use |
|---|---|
| Short sequences, low compute, streaming | RNN/LSTM |
| Long-range dependencies, parallel training | Transformer self-attention |
| Generation (next-token) | Masked/causal self-attention (decoder) |
| Understanding/classification | Bidirectional self-attention (encoder/BERT) |
| Seq2seq (MT, summarization) | Cross-attention (encoder-decoder) |

---

## 3. RAG — Retrieval Augmented Generation (PDF p218–228)

### Why RAG (limitations of plain LLMs — p218)
- **Knowledge cutoff**: LLM knowledge frozen at training time.
- **Hallucination**: fluent but wrong facts.
- No access to **private/domain data** (HR policies, medical records).
- RAG fixes these by retrieving external chunks and stuffing them into the prompt → answers are **grounded + citable + updatable without retraining**.

### RAG flow (p219)
1. **User query** → embed the query.
2. **Semantic search**: embed document chunks, compute **similarity scores** (cosine) query↔chunk.
3. **Select chunks**: by **threshold** (keep score ≥ t — variable count, quality guaranteed) or **top-k ranking** (fixed count, may include weak chunks). *These are not the same — classic 2-mark differentiation.*
4. **Augment prompt** (system prompt + query + chunks) → LLM generates grounded answer.

### 🔥 Exam pattern 1 — token budget math (Feb 2026 Q6, 6 marks)
> Query=160 tok, system prompt=240 tok, context limit=4096, chunk=480 tok.
- (a) Fixed = 160+240 = 400 → available = 4096−400 = 3696 → **max chunks = floor(3696/480) = 7**
- (b) 75% of 7 = 5.25 → **floor → 5 chunks** → tokens = 400 + 5×480 = **2800**
- (c) Fewer chunks → less noise/distraction but risk of missing relevant info: **quality improves if the top chunks already contain the answer; degrades if recall is lost**. Argue both sides, pick one based on precision-vs-recall.

### 🔥 Exam pattern 2 — thresholding (Mar 2026 Q6, 6 marks)
Scores 0.92, 0.86, 0.79, 0.71, 0.64, 0.58 with threshold 0.75 → **3 chunks pass** (0.92, 0.86, 0.79). Know the trade-off: high threshold ↑precision ↓recall; low threshold ↑recall ↓precision.

### Precision & Recall in RAG (p221)
- **Precision** = relevant retrieved / total retrieved → controls **noise**.
- **Recall** = relevant retrieved / total relevant in corpus → controls **coverage**.
- Retrieval quality directly bounds generation quality ("garbage in, garbage out").

### Vector DB vs Knowledge Graph (p222) — scenario question favorite
| Scenario | Use |
|---|---|
| Fuzzy semantic lookup: "Who is the CEO?" — similarity search | **Vector DB** |
| Multi-hop/structured query: "Which board meetings had ≥2 abstentions in 12 months?" | **Knowledge Graph** |
| Complete, consistent, explainable answers; causal/relational facts | **KG** |
| Unstructured text at scale, quick setup | **Vector DB** |
| Best of both (support agent stack) | **Hybrid: Vector + KG RAG** |

### RAG variants (1-line each — p226–228)
- **Multimodal RAG**: chunks include text + images + tables via multimodal embeddings.
- **Agentic RAG**: LLM acts as an agent — sets subgoals, plans retrieval steps, calls tools/APIs (not a passive generator).
- **KG-RAG / DRAGON**: integrate knowledge-graph retrieval with RAG for factual QA.

### RAG limitations
- Retrieval failures propagate (wrong chunks → wrong answer).
- Context window still finite — chunking strategy matters.
- Latency (retrieval + long prompt) and chunk-boundary information loss.
- Doesn't fix reasoning errors — only knowledge gaps.

---

## 4. Bonus quick-theory (also asked in these papers)

- **WordNet fails on domain text** (Mar 2026 Q5A): general-purpose senses miss medical/legal senses ("discharge") → fix: domain ontology/UMLS or domain fine-tuned embeddings.
- **RDF vs OWL** (Mar 2026 Q5B): RDF = ⟨subject, predicate, object⟩ triples for *representing facts*; OWL adds class hierarchies, property restrictions, cardinality → enables *reasoning & consistency checking*.
- **RDF modelling pattern** (Mar 2026 Q5C): entities as resources, facts as triples — (Paper1, hasAuthor, Author1); OWL defines classes (Paper, Author) + constraints (every Paper ≥1 Author).
- **Summarization types** (Mar 2026 Q7): single-doc generic vs multi-doc (dedupe across sources) vs query-focused (answers an info need) — one approach can't serve all because intent/redundancy handling differ.
- **MMR** (Feb 2026): `MMR(s) = λ·sim(s,q) − (1−λ)·max sim(s, selected)` — picks relevant but **non-redundant** sentences; higher-relevance sentence can lose if it's redundant.

---

### 30-second cheat: which formula for which question
| Question mentions... | Reach for |
|---|---|
| "contextual embedding for word X" + α weights | Output = Σ αᵢ·vᵢ |
| "multiple heads", W_O matrix | concat(heads) × W_O |
| "attention weights from scores" | softmax, then weighted sum |
| "max chunks / tokens passed to LLM" | (limit − fixed tokens) ÷ chunk size, floor |
| "similarity threshold" | count scores ≥ threshold |
| "relevant retrieved chunks" | precision / recall definitions |
