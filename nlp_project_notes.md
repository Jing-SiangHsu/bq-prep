# NLP Journal Finder — Project Notes

Team of 4 | University of Twente | September–November 2023
Goal: given a paper abstract + subject areas, recommend the top 5 journals to publish in.
Dataset: 40,001 Elsevier open-access articles (JSON) → 35,370 after cleaning → 378 journals across 26 subject areas.

---

## What was built

27 models total:
- 26 SciBERT-based abstract classifiers (one per subject area: COMP, MATH, BIOC, MEDI, ...)
- 1 subject area classifier (bag-of-words, not SciBERT)

Combined with a dynamic accuracy-weighted ensemble.
Deployed as a Flask web application.

**Results (test size: 3,000):**
- Top-1 accuracy: 76.8%
- Top-3 accuracy: 93.4%
- Top-5 accuracy: 96.9%
- Random baseline: 0.26% (1 out of 378 journals)

---

## How the system works, step by step

1. User submits a paper abstract (text) and selects subject areas (e.g., COMP, MATH)
2. The subject area classifier runs on the selected subject area names
3. For each selected subject area, the corresponding SciBERT abstract model runs on the abstract
4. All active models' predictions are combined using accuracy-based weights
5. Top 5 journals from the combined prediction are shown

**If user selects COMP + MATH, three models run:**
- Subject area model (always runs)
- COMP abstract model
- MATH abstract model

**Weight formula:**
```
weight of each model = its accuracy / sum of all active models' accuracies
final prediction = sum(weight × prediction) for each active model
```

Dynamic because: if you select only MATH (2 journals, ~100% accuracy), MATH model dominates. If you select AGRI (73 journals, ~60% accuracy) + COMP (34 journals), weights shift accordingly.

---

## What SciBERT is

SciBERT is a **pretrained language model** — not just a tokenizer. It has two parts:
- **Tokenizer:** splits text into tokens ("myocardial" → ["myo", "card", "ial"])
- **Transformer model:** converts tokens into contextual vectors (embeddings)

Trained by Allen AI on 1.14 million scientific papers (vs. regular BERT which trained on Wikipedia). Better at understanding scientific vocabulary and phrasing.

In the project: SciBERT weights were **frozen** (never changed). It was used only to convert abstracts into 768-number vectors. The 26 classifiers on top were what the team actually trained.

---

## What a transformer is

A transformer is a **type of neural network architecture** — the internal structure of the model.

Key invention: **attention mechanism.** Instead of reading text word by word (left to right), transformers read ALL words simultaneously and each word decides how much to "attend to" every other word.

Example — the word "bank":
- "I went to the bank to deposit money" → attends heavily to "deposit" and "money" → financial vector
- "I sat on the bank of the river" → attends heavily to "river" → geographic vector

Same word, different vector depending on context. This is called a **contextual embedding.**

The transformer processes all words at once (parallelizable), handles long-range dependencies well.

---

## What the [CLS] token is

When you feed text into BERT/SciBERT, it prepends a special fake word: `[CLS]` (classification token).

```
Input:  [CLS]  The  abstract  text  here  ...
Output:  v0    v1    v2       v3    v4   ...
```

Every token gets a 768-number vector out. **v0 ([CLS])** is special — because it has no real word meaning of its own, the model uses it to accumulate a summary of the **entire sentence/abstract.**

In the project: feed the abstract in → take only the [CLS] vector (768 numbers) → that's the abstract's "meaning summary" → feed into the small classifier.

---

## Transfer learning vs. fine-tuning

**Transfer learning / feature extraction (what the project did):**
SciBERT weights are frozen. Use SciBERT as a fixed "translation machine": text → 768-number vector. Train your own small classifier on those vectors.

```
Abstract → [SciBERT, frozen] → 768-dim vector → [your classifier, trained] → journal probabilities
```

Fast, cheap, works well with smaller datasets.

**Fine-tuning (not done here):**
Take SciBERT and continue training ALL its weights on your specific task. SciBERT updates its internal language understanding based on your journal data. More powerful but needs much more data and compute. With 110M parameters and only 35,370 samples, fine-tuning would likely overfit.

The project's choice was correct for the dataset size.

---

## Why 26 separate classifiers instead of one

One model over all 378 journals would need to simultaneously learn:
- "Among these 73 biochemistry journals, the distinction is subtle vocabulary differences in methods sections"
- "Among these 2 math journals, the distinction is proof style"
- "Among these 34 CS journals, the distinction is application domain"

These are completely different decision boundaries — the rules for telling two biochemistry journals apart are nothing like the rules for telling two math journals apart.

Separate models: each one only ever sees abstracts from its domain. COMP model trains only on CS papers, learns CS-specific distinctions. Never confused by medical abstracts.

Tradeoff: user must tell the system which subject areas apply (the app asks for that input). Fully automatic routing wasn't feasible.

---

## The two types of models in the project

**Abstract models (26, SciBERT-based):**
1. Clean and lowercase the abstract text
2. Tokenize with SciBERT tokenizer
3. Feed through SciBERT transformer → get [CLS] embedding (768 numbers)
4. Feed into a small neural network (input layer + softmax output layer)
5. Output: probability distribution over journals in that subject area

**Subject area model (1, NOT SciBERT-based):**
1. Take the subject area names as text (e.g., "COMP MATH")
2. Convert to word count vector with CountVectorizer (bag-of-words)
3. Feed into softmax regression (a single dense layer)
4. Output: probability distribution over all 378 journals

The subject area model uses simple bag-of-words because the input is just short label strings — no need for contextual embeddings.

---

## Neural network architecture types (for context)

| Type | What it's for | Key idea |
|---|---|---|
| **Feedforward / MLP** | General purpose, tabular data | Layers connected forward, no loops. Your 26 classifiers are this type |
| **CNN** | Images, grids | Small filters slide across input looking for local patterns |
| **RNN / LSTM** | Sequences (old NLP) | Read one word at a time, maintain a memory state. Replaced by transformers |
| **Transformer** | Text, images, audio (current dominant) | Attention: all tokens read at once, each attends to all others |
| **GAN** | Image generation (2018-2022) | Generator vs. Discriminator competing against each other |
| **Diffusion model** | Image generation (current dominant) | Gradually denoise from noise to image. Used in Stable Diffusion, DALL-E 3 |
| **GNN** | Graph data (molecules, social networks) | Each node updates based on its neighbors |

**Timeline:**
- Pre-2012: hand-crafted features
- 2012-2017: CNNs for vision, RNNs/LSTMs for language
- 2017-2018: transformers invented, BERT released
- 2020-now: foundation models + fine-tuning/prompting as the default paradigm

This project (2023) used SciBERT-era transfer learning — correct approach for the time and dataset size.

---

## ML Engineer vs AI Engineer — which is this project?

**ML Engineer** — you build and train models. You work with datasets, design architectures, run training pipelines, evaluate with metrics. The model is something you *construct*.

**AI Engineer** — you build *applications* using existing AI models. You call GPT/Claude APIs, write prompts, build RAG pipelines, chain models with LangChain. No training involved. The model is something you *call*.

**This project is ML Engineering** because:
- Designed the architecture (26 domain-specific classifiers + ensemble — a deliberate decision, not default)
- Trained 27 models on actual data
- Made feature engineering decisions (SciBERT embeddings for abstracts, CountVectorizer for subject areas; experimented and explicitly rejected body text + keywords)
- Designed and implemented the accuracy-weighted ensemble
- Evaluated systematically (top-1, top-3, top-5 on 3,000 test samples)

Using SciBERT doesn't make it AI Engineering — it was used as a frozen component inside a training pipeline, not as a black-box API. AI Engineering would describe this project if you had sent each abstract to the GPT-4 API and asked it to recommend journals via prompting. What the team did was build the AI system itself.

**"Machine Learning Engineer" on the resume is correct.**

---

## Numbers to remember

- 27 models (26 abstract + 1 subject area)
- 378 journals, 26 subject areas
- 35,370 training articles (Elsevier open-access)
- 768-dimensional [CLS] embedding from SciBERT
- 96.9% top-5 accuracy (3,000 test samples)
- Random baseline: 0.26% (1/378)
- SciBERT: allenai/scibert_scivocab_uncased, 110M parameters
- Flask web app, HuggingFace Transformers, Keras, scikit-learn, PyTorch
