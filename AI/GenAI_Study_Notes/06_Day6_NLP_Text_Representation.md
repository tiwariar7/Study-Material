# Day 6: NLP Text Representation — Bag of Words & TF-IDF; Word2Vec (CBOW & Skip-Gram)

---

## Topics Covered
1. **Bag of Words (BoW)**
2. **TF-IDF** (Term Frequency-Inverse Document Frequency)
3. **Word2Vec** (CBOW & Skip-Gram)
4. **Comparison of Text Representation Methods**

---

## 1. BAG OF WORDS (BoW)

### 1.1 Definition & Fundamentals

**Definition:** Bag of Words represents text as a vector of word counts (or presence), ignoring word order and grammar. Each document becomes a fixed-length vector where each dimension corresponds to a unique word in the vocabulary.

**Plain English:** Count how many times each word appears in a document. The position/order of words doesn't matter — it's literally a "bag" of words.

### 1.2 How It Works

```
Documents:
D1: "I love NLP"
D2: "I love machine learning"
D3: "NLP and machine learning are great"

Vocabulary: ["I", "love", "NLP", "machine", "learning", "and", "are", "great"]

BoW Matrix (count-based):
     I  love  NLP  machine  learning  and  are  great
D1: [1,  1,    1,    0,       0,       0,   0,    0  ]
D2: [1,  1,    0,    1,       1,       0,   0,    0  ]
D3: [0,  0,    1,    1,       1,       1,   1,    1  ]
```

### 1.3 Binary BoW vs Count BoW

- **Binary BoW:** 1 if word present, 0 if absent. Ignores frequency.
- **Count BoW:** Integer count of each word. Default.

### 1.4 Limitations

1. **Ignores word order:** "dog bites man" = "man bites dog" — same BoW vector!
2. **High dimensionality:** Vocabulary can be 50K+ words → very sparse vectors.
3. **No semantic meaning:** "good" and "excellent" are equally different as "good" and "bad".
4. **Dominated by frequent words:** Common words like "the" have high counts but low information.
5. **Sparse representation:** Most entries are 0.

### 1.5 Implementation

```python
from sklearn.feature_extraction.text import CountVectorizer

corpus = [
    "I love NLP and machine learning",
    "NLP is a subfield of machine learning",
    "I love deep learning"
]

# Count Vectorizer
cv = CountVectorizer(
    max_features=1000,      # Keep top 1000 words
    stop_words='english',   # Remove English stopwords
    ngram_range=(1, 2),     # Unigrams + bigrams
    min_df=1,               # Min document frequency
    max_df=0.95             # Max document frequency (remove too-common words)
)

X = cv.fit_transform(corpus)  # Returns sparse matrix
print(f"Shape: {X.shape}")  # (3, n_features)
print(f"Vocabulary: {cv.get_feature_names_out()}")
print(f"Dense matrix:\n{X.toarray()}")

# For new documents:
X_new = cv.transform(["NLP and deep learning"])
```

---

## 2. TF-IDF (TERM FREQUENCY — INVERSE DOCUMENT FREQUENCY)

### 2.1 Definition & Fundamentals

**Definition:** TF-IDF is a statistical measure that evaluates how important a word is to a document within a corpus. It increases with the word's frequency in the document (TF) but decreases with the word's frequency across all documents (IDF).

**Plain English:** Words that appear frequently in one document but rarely across all documents are likely important/distinctive for that document. TF-IDF captures this intuition.

### 2.2 Mathematical Foundation

**Term Frequency (TF):**
```
TF(t, d) = (Number of times term t appears in document d) / (Total terms in document d)
```
Alternative: raw count, log normalization, boolean.

**Inverse Document Frequency (IDF):**
```
IDF(t) = log(N / df(t))

where:
- N = total number of documents
- df(t) = number of documents containing term t
```
- Words in every document → IDF ≈ 0 (not distinctive).
- Words in few documents → IDF is high (distinctive).
- Sklearn uses smoothed version: IDF(t) = log((1 + N) / (1 + df(t))) + 1

**TF-IDF:**
```
TF-IDF(t, d) = TF(t, d) × IDF(t)
```

### 2.3 Worked Example

```
Corpus: D1="cat sat mat", D2="dog sat mat", D3="cat dog bird"
N = 3

TF for D1: cat=1/3, sat=1/3, mat=1/3
IDF: cat → log(3/2)=0.405, sat → log(3/2)=0.405, mat → log(3/2)=0.405
     dog → log(3/2)=0.405, bird → log(3/1)=1.099

TF-IDF for "cat" in D1 = (1/3) × 0.405 = 0.135
TF-IDF for "bird" in D3 = (1/3) × 1.099 = 0.366  ← Higher! "bird" is more distinctive
```

### 2.4 Why TF-IDF Over BoW?

| BoW | TF-IDF |
|-----|--------|
| Raw counts | Weighted by importance |
| Common words dominate | Common words downweighted |
| No distinction between distinctive and generic words | Highlights document-specific words |
| Simpler | More informative |

### 2.5 Implementation

```python
from sklearn.feature_extraction.text import TfidfVectorizer

tfidf = TfidfVectorizer(
    max_features=5000,
    stop_words='english',
    ngram_range=(1, 2),
    min_df=2,               # Ignore terms in < 2 docs
    max_df=0.9,             # Ignore terms in > 90% docs
    sublinear_tf=True,      # Apply log normalization to TF: 1 + log(tf)
    norm='l2'               # L2 normalize each document vector
)

X = tfidf.fit_transform(corpus)
print(f"Shape: {X.shape}")

# Feature names with their IDF scores
feature_names = tfidf.get_feature_names_out()
idf_scores = tfidf.idf_
top_features = sorted(zip(feature_names, idf_scores), key=lambda x: x[1], reverse=True)[:10]
print("Top distinctive features:", top_features)

# Use with classifier
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline

text_clf = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=5000, stop_words='english')),
    ('clf', MultinomialNB())
])
text_clf.fit(train_texts, train_labels)
predictions = text_clf.predict(test_texts)
```

### 2.6 Practical Considerations

- `sublinear_tf=True`: Use 1 + log(tf) instead of raw tf. Reduces impact of very frequent terms.
- `norm='l2'`: Normalize vectors so document length doesn't affect similarity.
- `min_df` / `max_df`: Filter rare and too-common words to reduce dimensionality and noise.
- **TF-IDF + ML classifiers** (SVM, Naive Bayes, Logistic Regression) is a strong baseline for text classification.

---

## 3. WORD2VEC

### 3.1 Definition & Fundamentals

**Definition:** Word2Vec is a neural network-based method that learns **dense, low-dimensional vector representations (embeddings)** of words, where semantically similar words have similar vectors. Developed by Mikolov et al. at Google (2013).

**Why Word2Vec is revolutionary:**
- BoW/TF-IDF produce **sparse, high-dimensional** vectors with no semantic meaning.
- Word2Vec produces **dense, low-dimensional** vectors (typically 100-300 dims) that capture semantic relationships.
- Famous example: **king - man + woman ≈ queen** (vector arithmetic captures analogy).

### 3.2 Key Insight — Distributional Hypothesis

> **"You shall know a word by the company it keeps."** — J.R. Firth

Words appearing in similar contexts have similar meanings. Word2Vec learns word vectors by predicting context.

### 3.3 Two Architectures

#### A. CBOW (Continuous Bag of Words)

**Task:** Predict the **target word** from its surrounding context words.

```
Context window = 2:
"The cat [sat] on the mat"

Input: ["The", "cat", "on", "the"]  →  Predict: "sat"
```

**Architecture:**
```
Input (context words) → Projection (average embeddings) → Output (predict target)

context_word_1 ─┐
context_word_2 ─┼→ Average → Hidden Layer → Softmax → target_word
context_word_3 ─┤                                    (probability over vocab)
context_word_4 ─┘
```

**Properties:**
- Faster to train (one prediction per position).
- Better for frequent words.
- Smooths over context (averages multiple words).

#### B. Skip-Gram

**Task:** Predict the **context words** from the target word.

```
Context window = 2:
"The cat [sat] on the mat"

Input: "sat"  →  Predict: ["The", "cat", "on", "the"]
```

**Architecture:**
```
Input (target word) → Projection (embedding) → Output (predict each context word)

                                              → context_word_1
target_word → Embedding → Hidden Layer ──────→ context_word_2
                                              → context_word_3
                                              → context_word_4
```

**Properties:**
- Slower to train (multiple predictions per position).
- Better for rare words (each occurrence creates multiple training examples).
- Works better with small datasets.
- **More commonly used in practice.**

### 3.4 Mathematical Foundation

**Skip-Gram Objective:**
```
Maximize: (1/T) Σᵢ₌₁ᵀ Σ_{-c ≤ j ≤ c, j≠0} log P(wᵢ₊ⱼ | wᵢ)
```
Where:
- T = number of words in corpus
- c = context window size
- P(wₒ | wᵢ) = softmax:
```
P(wₒ | wᵢ) = exp(v'ₒᵀ vᵢ) / Σₖ₌₁ᵛ exp(v'ₖᵀ vᵢ)
```

**Problem:** Softmax over entire vocabulary V (50K-1M words) is **extremely expensive** to compute at every step.

### 3.5 Training Optimization Techniques

**Negative Sampling:**
Instead of updating all V words in softmax, update the target word + K random "negative" (non-context) words.

```
Objective (per word pair):
log σ(v'ₒᵀ vᵢ) + Σₖ₌₁ᴷ 𝔼[log σ(-v'ₖᵀ vᵢ)]
```
Where σ = sigmoid, K = number of negative samples (typically 5-20).

**Intuition:** Maximize similarity with actual context words, minimize similarity with random words. Much faster than full softmax.

**Hierarchical Softmax:**
Use a binary tree (Huffman tree) to compute softmax in O(log V) instead of O(V).

**Subsampling of frequent words:**
Very common words ("the", "a") provide less information. Randomly discard them with probability:
```
P(discard) = 1 - √(t / f(w))
```
Where f(w) = word frequency, t = threshold (typically 10⁻⁵).

### 3.6 Properties of Word2Vec Embeddings

**Semantic relationships captured:**
```
king - man + woman ≈ queen       (gender analogy)
Paris - France + Italy ≈ Rome     (capital-country)
bigger - big + small ≈ smaller    (comparative)
```

**Cosine similarity:** Used to measure word similarity:
```
cos(w₁, w₂) = (w₁ · w₂) / (||w₁|| × ||w₂||)
```
Range: [-1, 1]. 1 = identical direction, 0 = orthogonal, -1 = opposite.

### 3.7 Hyperparameters

| Parameter | Description | Typical Values |
|-----------|-------------|----------------|
| `vector_size` | Dimensionality of word vectors | 100-300 |
| `window` | Context window size | 5-10 |
| `min_count` | Ignore words with frequency < this | 5 |
| `sg` | 0=CBOW, 1=Skip-Gram | 1 (Skip-Gram usually better) |
| `negative` | Number of negative samples | 5-20 |
| `epochs` | Training epochs | 5-10 |
| `workers` | CPU threads | 4 |

### 3.8 Implementation

```python
from gensim.models import Word2Vec
import gensim.downloader as api

# --- Train custom Word2Vec ---
# Corpus: list of tokenized sentences
sentences = [
    ["I", "love", "natural", "language", "processing"],
    ["word", "embeddings", "capture", "semantic", "meaning"],
    ["deep", "learning", "revolutionized", "NLP"],
    # ... (needs large corpus for good embeddings)
]

model = Word2Vec(
    sentences=sentences,
    vector_size=100,    # Embedding dimension
    window=5,           # Context window
    min_count=1,        # Min word frequency
    sg=1,               # 1=Skip-Gram, 0=CBOW
    negative=5,         # Negative sampling
    epochs=10,
    workers=4
)

# Get word vector
vector = model.wv['love']  # 100-dimensional vector
print(f"Shape: {vector.shape}")

# Find similar words
similar = model.wv.most_similar('love', topn=5)
print("Similar to 'love':", similar)

# Analogy: king - man + woman = ?
result = model.wv.most_similar(positive=['king', 'woman'], negative=['man'], topn=3)

# Similarity between two words
sim = model.wv.similarity('love', 'like')
print(f"Similarity: {sim:.4f}")

# --- Use pre-trained Word2Vec (Google News, 3M words, 300 dims) ---
# Download (1.7 GB)
wv = api.load('word2vec-google-news-300')
print(wv.most_similar('king'))
print(wv.most_similar(positive=['king', 'woman'], negative=['man']))

# Save / Load
model.save("word2vec.model")
model = Word2Vec.load("word2vec.model")
```

### 3.9 Word2Vec vs BoW/TF-IDF

| Aspect | BoW/TF-IDF | Word2Vec |
|--------|-----------|----------|
| Vector type | Sparse, high-dimensional | Dense, low-dimensional |
| Semantic meaning | No | Yes (similar words → similar vectors) |
| Word order | Ignored | Captured via context window |
| OOV words | Handled (new word = new dimension) | NOT handled (unseen words have no embedding) |
| Training | Simple counting | Neural network training on large corpus |
| Document representation | Direct (one vector per document) | Need aggregation (average, sum of word vectors) |
| Analogies | No | Yes (king - man + woman ≈ queen) |
| Polysemy | No (one representation per word) | No (one vector per word, regardless of meaning) |

### 3.10 Limitations of Word2Vec

1. **One vector per word (static embeddings):** "bank" (financial) and "bank" (river) have the same vector. Solved by contextual embeddings (ELMo, BERT).
2. **OOV problem:** Cannot handle unseen words. Solved by FastText (uses subword information).
3. **Requires large corpus** for good embeddings.
4. **Context window is limited** — doesn't capture long-range dependencies.
5. **No sentence/document embeddings** directly — need to aggregate word vectors.

### 3.11 Related Methods

| Method | Key Improvement over Word2Vec |
|--------|------------------------------|
| **GloVe** | Uses global co-occurrence matrix + factorization. Combines local (Word2Vec) and global stats. |
| **FastText** | Uses subword n-grams. Handles OOV words by composing from character n-grams. |
| **ELMo** | Contextual embeddings from bidirectional LSTM. Different vector per context. |
| **BERT** | Contextual embeddings from transformer. "bank" has different vectors in different sentences. |

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Using BoW/TF-IDF for deep learning** — use embeddings (Word2Vec, BERT) instead.
2. **Training Word2Vec on small corpus** — needs millions of words for good embeddings.
3. **Not normalizing TF-IDF vectors** — document length affects similarity without normalization.
4. **Expecting Word2Vec to handle polysemy** — "bank" has one vector regardless of meaning.
5. **Using word embeddings without a document-level representation** — average or sum word vectors for document-level tasks, or use dedicated document models (Doc2Vec).

### Complete Text Representation Comparison

| Method | Type | Captures Semantics | Word Order | OOV Handling | Polysemy |
|--------|------|-------------------|------------|--------------|----------|
| BoW | Sparse | No | No | Yes | No |
| TF-IDF | Sparse | No (but weights importance) | No | Yes | No |
| Word2Vec | Dense | Yes | Partial (window) | No | No |
| GloVe | Dense | Yes | Partial | No | No |
| FastText | Dense | Yes | Partial | Yes (subwords) | No |
| ELMo | Dense | Yes | Yes | Partial | Yes |
| BERT | Dense | Yes | Yes | Yes (subwords) | Yes |

---

## Interview Preparation

### Direct Interview Answers

**Q: What is TF-IDF?**
> "TF-IDF measures word importance in a document relative to a corpus. TF counts word frequency in the document; IDF penalizes words common across all documents. The product gives high weight to words that are frequent in a specific document but rare overall, making them distinctive features for that document."

**Q: Explain Word2Vec.**
> "Word2Vec is a neural method that learns dense vector representations of words from their context. It has two variants: CBOW predicts a target word from context words, and Skip-Gram predicts context from a target word. Training uses negative sampling for efficiency. The resulting vectors capture semantic relationships — similar words have similar vectors, and vector arithmetic captures analogies like king - man + woman ≈ queen."

### Interview Q&A Bank

**Basic:**
1. **Q:** What is Bag of Words? **A:** Text represented as word count vectors. Each dimension = unique word, value = count. Ignores word order. Simple, effective for many tasks, but high-dimensional and sparse.

2. **Q:** What does IDF stand for and why is it needed? **A:** Inverse Document Frequency. It downweights words that appear in many documents (like "the", "is") and upweights words that appear in few documents (distinctive terms). Without IDF, common words dominate.

**Intermediate:**
3. **Q:** What is the difference between CBOW and Skip-Gram? **A:** CBOW predicts the target word from context — faster, better for frequent words. Skip-Gram predicts context from the target — slower, better for rare words and small datasets. Skip-Gram is generally preferred.

4. **Q:** What is negative sampling in Word2Vec? **A:** Instead of computing softmax over the entire vocabulary (expensive), we train on the correct word pair + K randomly sampled "negative" pairs. This makes training O(K) instead of O(V) per update. K is typically 5-20.

**Advanced:**
5. **Q:** Why can't Word2Vec handle polysemy, and how is this solved? **A:** Word2Vec assigns one static vector per word regardless of context. "Bank" (financial) and "bank" (river) share the same vector. Contextual embedding models (ELMo, BERT) solve this by producing different vectors for the same word in different contexts, using the surrounding sentence.

6. **Q:** How would you represent a document using Word2Vec? **A:** Several approaches: (1) Average all word vectors in the document (simple, often effective). (2) TF-IDF weighted average. (3) Use Doc2Vec (Paragraph Vector) which extends Word2Vec with a document vector. (4) Use a sequential model (LSTM, Transformer) over the word embeddings.

---

## Rapid Revision Summary

- **BoW:** Count word occurrences → sparse vector. Simple, no semantics, no word order.
- **TF-IDF:** TF × IDF. Weights words by importance. Better than BoW for distinguishing documents.
- **Word2Vec:** Dense embeddings via neural networks. CBOW (context→word) vs Skip-Gram (word→context). Negative sampling for efficiency. Captures semantics but not polysemy.
- **Key formulas:** TF = count/total, IDF = log(N/df), TF-IDF = TF × IDF.
- **Key tradeoff:** BoW/TF-IDF handle any word but no semantics. Word2Vec captures semantics but fails on OOV words and polysemy.

---

## Self-Assessment

1. Compute TF-IDF for "cat" in D1 given: D1="cat cat dog", D2="dog bird", D3="cat bird fish". N=3.
2. What is the distributional hypothesis and how does Word2Vec use it?
3. Draw the Skip-Gram architecture. What is the input and what is the output?
4. Why is negative sampling needed? What happens without it?
5. Can Word2Vec distinguish between "bank" (river) and "bank" (finance)? Why or why not?

**Answers:**
1. TF("cat", D1) = 2/3. df("cat") = 2 (in D1 and D3). IDF = log(3/2) = 0.405. TF-IDF = (2/3)(0.405) = 0.27.
2. "Words appearing in similar contexts have similar meanings." Word2Vec learns embeddings by predicting words from their context (Skip-Gram) or context from words (CBOW), so words in similar contexts get similar vectors.
3. Input: one-hot vector of target word → embedding layer → hidden layer → softmax output over vocabulary (predicting context words). Each (target, context) pair is a training example.
4. Full softmax requires computing probability over entire vocabulary (50K-1M words) at every training step — extremely expensive. Negative sampling only updates the correct word and K random negatives, making it O(K) instead of O(V).
5. No. Word2Vec assigns a single static vector per word type. Both senses of "bank" are averaged into one vector. Contextual embeddings (BERT) solve this by producing different vectors based on surrounding context.

---

## Recall Checklist

- [ ] Explain BoW and its limitations
- [ ] Write TF-IDF formula and compute by hand
- [ ] Explain TF and IDF separately
- [ ] Implement TfidfVectorizer with key parameters
- [ ] Explain Word2Vec CBOW and Skip-Gram architectures
- [ ] Explain negative sampling and why it's needed
- [ ] Describe the distributional hypothesis
- [ ] Use Gensim to train and query Word2Vec
- [ ] Explain vector arithmetic analogies
- [ ] Compare BoW, TF-IDF, Word2Vec, GloVe, FastText, BERT
- [ ] List Word2Vec limitations and what solves each
