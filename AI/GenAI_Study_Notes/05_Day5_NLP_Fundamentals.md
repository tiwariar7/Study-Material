# Day 5: NLP Fundamentals — Tokenization, Stopwords, Stemming & Lemmatization

---

## Topics Covered
1. **NLP Overview & Text Preprocessing Pipeline**
2. **Tokenization** (Word, Sentence, Subword)
3. **Stopwords** (Removal, Custom Lists)
4. **Stemming** (Porter, Snowball, Lancaster)
5. **Lemmatization** (WordNet, spaCy)
6. **Other Text Preprocessing** (Lowercasing, Punctuation, Regex, N-grams)

---

## 1. NLP OVERVIEW & TEXT PREPROCESSING

### 1.1 What is NLP?

**Definition:** Natural Language Processing (NLP) is a branch of AI that enables computers to understand, interpret, and generate human language.

**Why preprocessing?** Raw text is noisy, ambiguous, and variable. Machines need standardized numerical representations. Preprocessing converts raw text → clean, structured data → features for ML models.

### 1.2 Standard NLP Preprocessing Pipeline

```
Raw Text
  → Lowercasing
  → Removing punctuation, special characters, HTML tags
  → Tokenization (splitting into tokens)
  → Removing stopwords
  → Stemming or Lemmatization
  → Feature extraction (BoW, TF-IDF, embeddings) [Day 6]
```

**Important:** The pipeline order matters. Tokenize before removing stopwords (need tokens to identify stopwords). The choice of stemming vs lemmatization depends on the task.

---

## 2. TOKENIZATION

### 2.1 Definition

**Definition:** Tokenization is the process of splitting text into smaller meaningful units called **tokens** — typically words, subwords, or sentences.

**Why it matters:** It's the fundamental first step in NLP. Every downstream operation (counting words, building vocabulary, creating embeddings) requires tokenized text.

### 2.2 Types of Tokenization

**Word Tokenization:**
Split text into words.
```python
text = "I can't believe it's not butter!"

# Simple split (naive)
text.split()  # ["I", "can't", "believe", "it's", "not", "butter!"]
# Problem: "can't" → one token. "butter!" includes punctuation.

# NLTK
import nltk
nltk.download('punkt_tab')
from nltk.tokenize import word_tokenize
word_tokenize(text)  # ["I", "ca", "n't", "believe", "it", "'s", "not", "butter", "!"]
# Handles contractions: "can't" → "ca" + "n't"

# spaCy
import spacy
nlp = spacy.load("en_core_web_sm")
doc = nlp(text)
[token.text for token in doc]  # ["I", "ca", "n't", "believe", "it", "'s", "not", "butter", "!"]
```

**Sentence Tokenization:**
Split text into sentences.
```python
from nltk.tokenize import sent_tokenize
text = "Dr. Smith went to Washington. He arrived at 3 p.m. It was cold."
sent_tokenize(text)
# ["Dr. Smith went to Washington.", "He arrived at 3 p.m.", "It was cold."]
# Handles abbreviations like "Dr." and "p.m." correctly
```

**Subword Tokenization (Critical for Modern NLP):**

Modern LLMs use subword tokenization instead of word tokenization:

| Method | Used By | Approach |
|--------|---------|----------|
| **BPE (Byte Pair Encoding)** | GPT-2, GPT-3, RoBERTa | Iteratively merges most frequent character pairs |
| **WordPiece** | BERT, DistilBERT | Similar to BPE, but uses likelihood-based merging |
| **SentencePiece** | T5, ALBERT, XLNet | Language-independent, treats text as raw bytes |
| **Unigram** | Part of SentencePiece | Starts with large vocab, prunes low-probability pieces |

**Why subword tokenization?**
- Handles **out-of-vocabulary (OOV)** words: "unforgettable" → ["un", "forget", "table"] or similar.
- Balances vocabulary size vs sequence length.
- Language-agnostic.
- All modern transformers use subword tokenization.

**BPE Example:**
```
Corpus: "low", "lower", "newest", "widest"

1. Start with character-level: l o w </w>, l o w e r </w>, ...
2. Most frequent pair: (e, s) → merge to "es"
3. Most frequent pair: (es, t) → merge to "est"
4. Continue until desired vocab size...

Result: "newest" → ["new", "est"]
```

### 2.3 Tokenization Edge Cases

- **Contractions:** "don't" → "do" + "n't" or "don't" (depends on tokenizer)
- **Hyphenated words:** "state-of-the-art" → one or multiple tokens
- **Numbers:** "3.14" → "3.14" or "3" + "." + "14"
- **URLs, emails:** Special handling needed
- **Emojis, Unicode:** Modern tokenizers handle; older ones may not
- **Multilingual text:** SentencePiece handles; NLTK/spaCy need language-specific models

---

## 3. STOPWORDS

### 3.1 Definition

**Definition:** Stopwords are common words that carry little semantic meaning and are often removed during preprocessing to reduce noise and dimensionality.

**Examples:** "the", "is", "at", "which", "on", "a", "an", "and", "or", "but", "in", "of"

### 3.2 Why Remove Stopwords?

- **Reduce noise:** These words appear in every document, adding little discriminative power.
- **Reduce dimensionality:** Fewer tokens = smaller feature vectors.
- **Improve model focus:** Model attends to meaningful content words.
- **Faster processing:** Fewer tokens to process.

### 3.3 When NOT to Remove Stopwords

- **Sentiment analysis:** "not good" → removing "not" changes meaning completely.
- **Machine translation:** Every word matters for grammatical correctness.
- **Question answering:** "What is the capital?" — "the" and "is" provide structure.
- **Language models / Deep learning:** Modern models (BERT, GPT) learn to handle stopwords; removing them can hurt performance.
- **Phrase matching:** "to be or not to be" — all stopwords but very meaningful.

**Rule of thumb:** Remove stopwords for traditional ML (BoW, TF-IDF). Keep them for deep learning models.

### 3.4 Implementation

```python
# NLTK stopwords
import nltk
nltk.download('stopwords')
from nltk.corpus import stopwords

stop_words = set(stopwords.words('english'))
print(f"NLTK has {len(stop_words)} English stopwords")
# 179 stopwords including: i, me, my, myself, we, our, ours, ...

tokens = ["this", "is", "a", "sample", "sentence", "showing", "stopword", "removal"]
filtered = [w for w in tokens if w.lower() not in stop_words]
# ["sample", "sentence", "showing", "stopword", "removal"]

# spaCy stopwords
nlp = spacy.load("en_core_web_sm")
print(f"spaCy has {len(nlp.Defaults.stop_words)} stopwords")
# ~326 stopwords

doc = nlp("This is a sample sentence showing stopword removal")
filtered_spacy = [token.text for token in doc if not token.is_stop]

# Custom stopwords
custom_stops = stop_words | {"sample", "showing"}  # Add domain-specific
custom_stops -= {"not", "no", "nor"}  # Remove negations (for sentiment)
```

---

## 4. STEMMING

### 4.1 Definition

**Definition:** Stemming reduces words to their **stem** (root form) by removing suffixes using crude heuristic rules. The result may not be a valid dictionary word.

**Plain English:** Chop off word endings: "running" → "run", "studies" → "studi", "fairly" → "fairli".

**Note:** The stem is NOT always a real word. Stemming is fast but approximate.

### 4.2 Common Stemming Algorithms

| Algorithm | Aggressiveness | Quality | Speed |
|-----------|---------------|---------|-------|
| **Porter Stemmer** | Moderate | Good | Fast |
| **Snowball Stemmer** | Moderate (improved Porter) | Better | Fast |
| **Lancaster Stemmer** | Aggressive | Lower (over-stems) | Fastest |

### 4.3 Porter Stemmer Rules (Key Examples)

```
Step 1: Remove plurals and -ed/-ing
  caresses → caress → caress
  ponies → poni
  cats → cat
  agreed → agree
  walking → walk

Step 2: Map suffixes
  relational → relate
  conditional → condition
  effectiveness → effective

Step 3: Map suffixes
  electrici → electric
  hopeful → hope

Step 4: Remove suffixes
  revival → reviv
  allowance → allow

Step 5: Clean up
  probate → probat
  rate → rate
```

### 4.4 Implementation

```python
from nltk.stem import PorterStemmer, SnowballStemmer, LancasterStemmer

porter = PorterStemmer()
snowball = SnowballStemmer("english")
lancaster = LancasterStemmer()

words = ["running", "runs", "ran", "runner", "fairly", "studies", "studying"]

print("Word       | Porter    | Snowball  | Lancaster")
print("-" * 55)
for w in words:
    print(f"{w:11}| {porter.stem(w):10}| {snowball.stem(w):10}| {lancaster.stem(w)}")

# Output:
# running    | run       | run       | run
# runs       | run       | run       | run
# ran        | ran       | ran       | ran        ← Irregular verb not handled!
# runner     | runner    | runner    | run
# fairly     | fairli    | fair      | fair
# studies    | studi     | studi     | study
# studying   | studi     | studi     | study
```

**Key Limitation:** Stemming cannot handle irregular verbs (ran→ran, not "run"), and often produces non-words ("studies" → "studi").

---

## 5. LEMMATIZATION

### 5.1 Definition

**Definition:** Lemmatization reduces words to their **lemma** (dictionary base form) using vocabulary lookup and morphological analysis. The result is always a valid dictionary word.

**Plain English:** Find the actual dictionary word: "running" → "run", "better" → "good", "studies" → "study".

### 5.2 How Lemmatization Differs from Stemming

| Aspect | Stemming | Lemmatization |
|--------|----------|---------------|
| Approach | Rule-based suffix stripping | Dictionary lookup + morphological analysis |
| Output | May not be a real word | Always a valid word |
| Speed | Very fast | Slower |
| Quality | Approximate | Accurate |
| Context | Ignores context | Can use POS tags |
| "better" → | "better" | "good" |
| "studies" → | "studi" | "study" |
| "ran" → | "ran" | "run" |

### 5.3 Part-of-Speech (POS) Matters for Lemmatization

The lemma depends on whether a word is a noun, verb, adjective, etc.:
```
"meeting" (noun) → "meeting"
"meeting" (verb) → "meet"

"better" (adjective) → "good"
"better" (adverb) → "well"
```

### 5.4 Implementation

```python
# NLTK WordNet Lemmatizer
import nltk
nltk.download('wordnet')
from nltk.stem import WordNetLemmatizer
from nltk.corpus import wordnet

lem = WordNetLemmatizer()

# Without POS tag (defaults to noun)
print(lem.lemmatize("running"))    # "running" (treated as noun!)
print(lem.lemmatize("running", pos='v'))  # "run" ← specify verb

# POS mapping helper (NLTK POS tags → WordNet POS)
def get_wordnet_pos(treebank_tag):
    if treebank_tag.startswith('J'):    return wordnet.ADJ
    elif treebank_tag.startswith('V'):  return wordnet.VERB
    elif treebank_tag.startswith('N'):  return wordnet.NOUN
    elif treebank_tag.startswith('R'):  return wordnet.ADV
    else:                               return wordnet.NOUN  # default

# Auto-lemmatize with POS tagging
from nltk import pos_tag
tokens = word_tokenize("The striped bats are hanging on their feet")
tagged = pos_tag(tokens)  # [('The', 'DT'), ('striped', 'JJ'), ...]
lemmatized = [lem.lemmatize(word, get_wordnet_pos(tag)) for word, tag in tagged]
# ['The', 'striped', 'bat', 'be', 'hang', 'on', 'their', 'foot']

# spaCy Lemmatizer (automatically uses POS context)
nlp = spacy.load("en_core_web_sm")
doc = nlp("The striped bats are hanging on their feet")
[token.lemma_ for token in doc]
# ['the', 'stripe', 'bat', 'be', 'hang', 'on', 'their', 'foot']
```

### 5.5 When to Use Stemming vs Lemmatization

| Scenario | Use |
|----------|-----|
| Speed-critical, large corpus | Stemming |
| Information retrieval, search engines | Stemming (good enough, fast) |
| Text classification with BoW/TF-IDF | Either (lemmatization slightly better) |
| Sentiment analysis | Lemmatization (preserve meaning) |
| Chatbots, question answering | Lemmatization |
| Precision-critical applications | Lemmatization |
| Modern deep learning (BERT, etc.) | **Neither** — subword tokenization handles this |

---

## 6. OTHER TEXT PREPROCESSING

### 6.1 Lowercasing

```python
text = "NLP is AMAZING!"
text.lower()  # "nlp is amazing!"
```
- **Usually do:** Standardizes text. "Apple" = "apple".
- **Don't do when:** Case carries meaning (NER: "Apple" company vs "apple" fruit, abbreviations: "US" vs "us").

### 6.2 Removing Punctuation & Special Characters

```python
import re

text = "Hello, World! How are you? Email: test@email.com #NLP"

# Remove all punctuation
clean = re.sub(r'[^\w\s]', '', text)  # "Hello World How are you Email testemailcom NLP"

# Keep some punctuation (e.g., for sentence tokenization)
clean = re.sub(r'[^a-zA-Z0-9\s.!?]', '', text)

# Remove HTML tags
html = "<p>Hello <b>world</b></p>"
clean_html = re.sub(r'<[^>]+>', '', html)  # "Hello world"

# Remove URLs
text_url = "Visit https://example.com for more"
clean_url = re.sub(r'http\S+|www\.\S+', '', text_url)
```

### 6.3 N-grams

**Definition:** N-grams are contiguous sequences of n tokens from text.

```
Text: "I love NLP"

Unigrams (n=1): ["I", "love", "NLP"]
Bigrams (n=2):  ["I love", "love NLP"]
Trigrams (n=3): ["I love NLP"]
```

**Why N-grams?**
- Capture local context: "not good" (bigram) vs "not" + "good" (unigrams).
- Used in BoW/TF-IDF to capture phrases.
- Language models: predict next word given previous n-1 words.

```python
from nltk import ngrams

tokens = ["I", "love", "natural", "language", "processing"]
bigrams = list(ngrams(tokens, 2))
# [('I', 'love'), ('love', 'natural'), ('natural', 'language'), ('language', 'processing')]

# Sklearn can handle n-grams directly in TF-IDF
from sklearn.feature_extraction.text import TfidfVectorizer
tfidf = TfidfVectorizer(ngram_range=(1, 2))  # unigrams + bigrams
```

### 6.4 Complete Preprocessing Pipeline

```python
import re
import nltk
from nltk.tokenize import word_tokenize
from nltk.corpus import stopwords
from nltk.stem import WordNetLemmatizer

nltk.download(['punkt_tab', 'stopwords', 'wordnet', 'averaged_perceptron_tagger_eng'])

def preprocess_text(text, use_lemma=True, remove_stopwords=True):
    """Complete NLP preprocessing pipeline."""
    # 1. Lowercase
    text = text.lower()
    
    # 2. Remove HTML tags
    text = re.sub(r'<[^>]+>', '', text)
    
    # 3. Remove URLs
    text = re.sub(r'http\S+|www\.\S+', '', text)
    
    # 4. Remove special characters and numbers (optional)
    text = re.sub(r'[^a-zA-Z\s]', '', text)
    
    # 5. Tokenize
    tokens = word_tokenize(text)
    
    # 6. Remove stopwords
    if remove_stopwords:
        stop_words = set(stopwords.words('english'))
        tokens = [t for t in tokens if t not in stop_words]
    
    # 7. Stemming or Lemmatization
    if use_lemma:
        lemmatizer = WordNetLemmatizer()
        tokens = [lemmatizer.lemmatize(t) for t in tokens]
    else:
        from nltk.stem import PorterStemmer
        stemmer = PorterStemmer()
        tokens = [stemmer.stem(t) for t in tokens]
    
    # 8. Remove very short tokens
    tokens = [t for t in tokens if len(t) > 1]
    
    return ' '.join(tokens)

# Example
raw = "<p>The cats are running quickly! Visit https://example.com for MORE info.</p>"
print(preprocess_text(raw))
# "cat running quickly info"
```

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Removing stopwords before tokenization** — you need tokens to identify stopwords.
2. **Using stemming when meaning matters** — "better" stemmed is "better" not "good".
3. **Not specifying POS for NLTK lemmatizer** — defaults to noun, gives wrong results for verbs.
4. **Applying traditional preprocessing to transformer inputs** — BERT/GPT have their own tokenizers. Don't stem/lemmatize/remove stopwords.
5. **Removing all numbers blindly** — "COVID-19", "Python 3.10" — numbers may be meaningful.
6. **Not handling contractions** — "I'll", "don't" need special handling.

---

## Interview Preparation

### Direct Interview Answers

**Q: What is tokenization and why is it important?**
> "Tokenization splits text into smaller units (tokens) — words, subwords, or sentences. It's the first step in NLP because all downstream processing (counting, embedding, model input) requires discrete tokens. Modern LLMs use subword tokenization (BPE, WordPiece) to handle any word, even unseen ones, by breaking them into known subword pieces."

**Q: Stemming vs Lemmatization — when to use which?**
> "Stemming is fast, rule-based suffix stripping that may produce non-words ('studies'→'studi'). Lemmatization uses dictionary lookup and morphology to return valid words ('studies'→'study', 'better'→'good'). Use stemming for speed in search/IR; use lemmatization when accuracy matters (sentiment, QA). For modern deep learning, use neither — subword tokenization handles morphology."

### Interview Q&A Bank

**Basic:**
1. **Q:** What are stopwords? **A:** Common words (the, is, at, which) with little semantic meaning. Removed in traditional NLP to reduce noise and dimensionality. Keep for deep learning and tasks where context matters.

2. **Q:** What is the difference between word and subword tokenization? **A:** Word tokenization splits on whitespace/punctuation — struggles with OOV words. Subword (BPE, WordPiece) splits into smaller pieces from a learned vocabulary — handles any word by decomposing into known subwords. All modern transformers use subword tokenization.

**Intermediate:**
3. **Q:** How does BPE tokenization work? **A:** Start with character-level vocabulary. Iteratively merge the most frequent adjacent pair of tokens into a new token. Repeat until desired vocabulary size. New words are decomposed into the longest matching subwords from the vocabulary.

4. **Q:** Why does NLTK's WordNetLemmatizer need POS tags? **A:** The lemma depends on part of speech. "meeting" as noun stays "meeting"; as verb becomes "meet". Without POS info, it defaults to noun, giving incorrect results for verbs and adjectives.

**Advanced:**
5. **Q:** If you're building a text classification system with TF-IDF features, describe your preprocessing pipeline. **A:** Lowercase → remove HTML/URLs → remove special characters → tokenize → remove stopwords (domain-specific list) → lemmatize → create TF-IDF features with n-grams (1,2). I'd skip stemming for quality. For deep learning, I'd use the model's built-in tokenizer and skip manual preprocessing.

6. **Q:** Why do modern transformers (BERT, GPT) not need traditional preprocessing? **A:** They use subword tokenization that handles morphology internally. Their attention mechanism learns which tokens to focus on (effective stopword handling). Removing stopwords or lemmatizing would destroy important contextual information that the model leverages for understanding.

---

## Rapid Revision Summary

- **Tokenization:** Split text into tokens. Word (NLTK, spaCy), Sentence (sent_tokenize), Subword (BPE/WordPiece for transformers).
- **Stopwords:** Common low-meaning words. Remove for BoW/TF-IDF. Keep for deep learning and sentiment.
- **Stemming:** Heuristic suffix removal. Fast, approximate, may produce non-words. Porter (moderate), Lancaster (aggressive).
- **Lemmatization:** Dictionary-based. Always produces valid words. Needs POS tags for accuracy. Slower but better quality.
- **Pipeline:** lowercase → clean → tokenize → stopwords → stem/lemmatize → vectorize.
- **Modern NLP:** Use model's tokenizer directly. Skip traditional preprocessing.

---

## Self-Assessment

1. Why is "can't" → ["ca", "n't"] and not ["can't"] in NLTK?
2. When should you NOT remove stopwords? Give 3 examples.
3. Stem "running", "runs", "ran" using Porter Stemmer. What's the limitation?
4. Lemmatize "running" without and with POS='v'. Explain the difference.
5. What is BPE and why is it used in modern LLMs?
6. Describe a complete preprocessing pipeline for a TF-IDF based text classifier.

**Answers:**
1. NLTK follows Penn Treebank convention — contractions are split into constituent parts for grammatical analysis: "can" + "not" → "ca" + "n't".
2. Sentiment analysis ("not good"), question answering (structure words needed), machine translation (every word matters). Also: phrase extraction, language modeling.
3. "running"→"run", "runs"→"run", "ran"→"ran". Limitation: cannot handle irregular verbs — "ran" should become "run" but stemming doesn't know that.
4. Without POS: lemmatize("running") = "running" (treated as noun). With pos='v': lemmatize("running", 'v') = "run" (treated as verb). POS disambiguation is essential.
5. Byte Pair Encoding: iteratively merges most frequent character pairs to build a vocabulary of subwords. Used in GPT-2/3 because it handles OOV words (decomposes into known subwords), is language-agnostic, and balances vocabulary size vs sequence length.
6. lowercase → re.sub HTML/URLs → re.sub special chars → word_tokenize → remove stopwords (custom list) → lemmatize with POS tags → TfidfVectorizer(ngram_range=(1,2)) → train classifier.

---

## Recall Checklist

- [ ] Explain 3 types of tokenization with examples
- [ ] Compare BPE, WordPiece, and SentencePiece
- [ ] Define stopwords and list when to remove/keep them
- [ ] Implement stopword removal in NLTK and spaCy
- [ ] Compare Porter, Snowball, and Lancaster stemmers
- [ ] Explain why lemmatization needs POS tags
- [ ] Implement lemmatization with POS in NLTK and spaCy
- [ ] Write a complete preprocessing pipeline function
- [ ] Explain why transformers don't need traditional preprocessing
- [ ] Define N-grams and their purpose
