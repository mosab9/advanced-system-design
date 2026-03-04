# Design a Search Engine

Full-text search with relevance ranking.

---

## 1. Requirements

### Functional
- Index documents/web pages
- Search by keywords
- Rank results by relevance
- Autocomplete/suggestions
- Spell correction

### Non-Functional
- Low latency (< 200ms)
- High availability
- Scale to billions of documents
- Fresh results

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────────┐                       ┌─────────────┐            │
│   │  Crawler    │──────────────────────▶│  Indexer    │            │
│   └─────────────┘                       └──────┬──────┘            │
│                                                │                   │
│                                                ▼                   │
│                                         ┌─────────────┐            │
│                                         │  Inverted   │            │
│                                         │   Index     │            │
│                                         └─────────────┘            │
│                                                ▲                   │
│                                                │                   │
│   ┌─────────────┐     ┌─────────────┐   ┌─────┴─────┐            │
│   │   User      │────▶│   Query     │──▶│  Ranker   │            │
│   │   Query     │     │  Processor  │   └───────────┘            │
│   └─────────────┘     └─────────────┘                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Inverted Index

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Inverted Index                                  │
│                                                                     │
│   Forward Index (document → words):                                │
│   Doc1: "the quick brown fox"                                      │
│   Doc2: "the lazy dog"                                             │
│   Doc3: "quick brown dog"                                          │
│                                                                     │
│   Inverted Index (word → documents):                               │
│   ┌──────────┬─────────────────────────────────────────────────┐   │
│   │ Term     │ Posting List (doc_id, position, frequency)      │   │
│   ├──────────┼─────────────────────────────────────────────────┤   │
│   │ the      │ [(1, [0]), (2, [0])]                            │   │
│   │ quick    │ [(1, [1]), (3, [0])]                            │   │
│   │ brown    │ [(1, [2]), (3, [1])]                            │   │
│   │ fox      │ [(1, [3])]                                      │   │
│   │ lazy     │ [(2, [1])]                                      │   │
│   │ dog      │ [(2, [2]), (3, [2])]                            │   │
│   └──────────┴─────────────────────────────────────────────────┘   │
│                                                                     │
│   Query: "quick brown"                                             │
│   → Get posting lists for "quick" and "brown"                     │
│   → Intersect: Doc1, Doc3 contain both terms                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Ranking (TF-IDF + PageRank)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Ranking Factors                                 │
│                                                                     │
│   TF-IDF (Term Frequency - Inverse Document Frequency):            │
│                                                                     │
│   TF = (count of term in doc) / (total terms in doc)              │
│   IDF = log(total docs / docs containing term)                    │
│   TF-IDF = TF × IDF                                               │
│                                                                     │
│   Example: Query "apple"                                           │
│   - Doc1 (tech blog): "apple" appears 10 times in 1000 words      │
│     TF = 10/1000 = 0.01                                           │
│   - Doc2 (recipe): "apple" appears 5 times in 200 words           │
│     TF = 5/200 = 0.025                                            │
│   - IDF = log(1M docs / 100K docs with "apple") = 1               │
│                                                                     │
│   PageRank (link analysis):                                        │
│   - Pages with more inbound links rank higher                     │
│   - Quality of linking pages matters                              │
│                                                                     │
│   Final Score = f(TF-IDF, PageRank, freshness, user signals)      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Query Processing

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Query Processing Pipeline                       │
│                                                                     │
│   User Query: "How to bake an apple pie?"                          │
│                                                                     │
│   1. Tokenization                                                  │
│      → ["how", "to", "bake", "an", "apple", "pie"]                │
│                                                                     │
│   2. Normalization                                                 │
│      → lowercase, remove punctuation                               │
│                                                                     │
│   3. Stop word removal                                             │
│      → ["bake", "apple", "pie"]                                   │
│                                                                     │
│   4. Stemming/Lemmatization                                        │
│      → ["bake", "appl", "pie"] (or lemmas)                        │
│                                                                     │
│   5. Spell correction                                              │
│      → "aple" → "apple"                                            │
│                                                                     │
│   6. Query expansion (synonyms)                                    │
│      → ["bake", "cook", "apple", "pie", "tart"]                   │
│                                                                     │
│   7. Index lookup + ranking                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Index Sharding

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Sharding Strategies                             │
│                                                                     │
│   Document-based sharding:                                         │
│   - Docs 1-1M → Shard 1                                           │
│   - Docs 1M-2M → Shard 2                                          │
│   - Query hits ALL shards, merge results                          │
│                                                                     │
│   Term-based sharding:                                             │
│   - Terms A-H → Shard 1                                           │
│   - Terms I-P → Shard 2                                           │
│   - Query hits specific shards based on terms                     │
│                                                                     │
│   Google approach: Document-based with replication                 │
│                                                                     │
│   Query: "apple pie"                                               │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐                             │
│   │ Shard 1 │ │ Shard 2 │ │ Shard 3 │                             │
│   │ Top 10  │ │ Top 10  │ │ Top 10  │                             │
│   └────┬────┘ └────┬────┘ └────┬────┘                             │
│        │           │           │                                   │
│        └───────────┼───────────┘                                   │
│                    ▼                                               │
│              Merge & Re-rank                                       │
│              Return Top 10                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Autocomplete (Typeahead)

```
Trie structure for prefix matching:

        root
       /    \
      a      b
     / \      \
    p   n     a
   / \   \     \
  p   r   d    k
 /     \        \
l       o        e
e       i
        d

Query: "ap"
→ Traverse to "ap" node
→ Return top completions: "apple", "app", "april"

Optimizations:
- Store popular queries in memory
- Pre-compute top suggestions per prefix
- Use Redis sorted sets for ranking
```

---

## 8. Interview Tips

**Key Points:**
- Inverted index structure
- TF-IDF scoring
- PageRank for web search
- Query processing pipeline
- Index sharding strategies
- Autocomplete with tries

**Questions to Ask:**
- Web search or document search?
- Real-time indexing needed?
- Query volume?
- Languages supported?
