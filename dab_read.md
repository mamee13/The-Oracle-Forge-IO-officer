# DAB — DataAgentBench Complete Reference

Source: arxiv.org/html/2603.20576v1 | UC Berkeley EPIC Data Lab + Hasura PromptQL | March 2026

---

## What DAB Is

The first benchmark that evaluates AI agents on realistic, multi-database enterprise workloads. Not text-to-SQL. Not table QA. End-to-end data agent evaluation across heterogeneous database systems.

Built from a formative study of real enterprise query patterns across six industries: technology, finance, food services, e-commerce, SaaS, and healthcare.

---

## Top-Line Numbers

| Metric | Value |
|---|---|
| Total queries | 54 |
| Datasets | 12 |
| Domains | 9 |
| Database systems | 4 (PostgreSQL, MongoDB, SQLite, DuckDB) |
| Queries requiring multi-database join | 54 — all of them |
| Queries with ill-formatted join keys | 26 |
| Queries requiring text extraction | 47 of 54 |
| Queries requiring domain knowledge | 30 |
| Best score — Gemini-3-Pro (ReAct) | 38% pass@1 |
| PromptQL + Claude-Opus-4.6 | 51% pass@1 |
| patents dataset score | 0% for every model tested |
| deps_dev_v1 best score | 6% |

---

## The Four Database Systems

Every dataset spans at least two of these four systems.

Placement logic used during dataset construction:
- MongoDB — unstructured and customer-facing data (documents, profiles, reviews)
- DuckDB, PostgreSQL, SQLite — structured data (sales records, stock prices, metadata)

SQL dialects differ across systems: PostgreSQL requires double quotes for case-sensitive column names; SQLite and DuckDB do not. MongoDB uses its own query language entirely.

---

## The Four Hard Requirements

Every query in DAB involves (i) and at least one of (ii) or (iii). Property (iv) appears in proportion to its prevalence in the formative study.

**(i) Multi-database integration**
A single query requires fetching from multiple databases with different query dialects and merging results. All 54 queries require this.

**(ii) Ill-formatted join keys**
The same entity has different identifier formats across databases. Examples: integer IDs in PostgreSQL become prefixed strings in SQLite; trailing spaces corrupt 25% of IDs in crmarenapro. 26 queries require this.

**(iii) Unstructured text transformation**
Structured values are embedded in free-text fields and must be extracted before they can be filtered, grouped, or joined. 47 of 54 queries require this. Two types:
- Data-independent: fixed regex pattern works uniformly (e.g., extracting star counts from GitHub descriptions)
- Data-dependent: agent must inspect each row individually (e.g., classifying news article intent)

**(iv) Domain knowledge**
Answering correctly requires expertise not in the schema — financial formulas, medical terminology, CRM definitions, fiscal conventions. 30 queries require this.

---

## The 12 Datasets

### 1. agnews — News Articles
- Domain: news/media
- Queries: 4
- Challenge: extracting article categories and geographic regions from free-text descriptions
- Text transformation type: data-dependent (classifying article content)

### 2. bookreview — Amazon Books + Reviews
- Domain: e-commerce
- Queries: 3
- Databases: PostgreSQL (`books_database`) + SQLite (`review_database`)
- Tables:
  - `books_info`: title, subtitle, author, rating_number, features, description, price, store, categories, details, book_id
  - `review`: rating, title, text, purchase_id, review_time, helpful_vote, verified_purchase
- Join key trap: `book_id` in PostgreSQL → `bid_123`; `purchase_id` in SQLite → `bref_123`. Same entity, different prefix format. Strip prefixes and join on numeric suffix.
- Text trap: publication year and language are embedded in the `details` free-text field, not in dedicated columns
- Known failure (FM4): year extraction regex matches ISBN numbers. Pattern `\b(19\d{2}|20\d{2})\b` extracts 1932 from an ISBN instead of the true year 2004.
- Known failure (FM2): agents average per-book averages instead of averaging all ratings directly within each decade

### 3. crmarenapro — CRM and Sales Operations
- Domain: customer relationship management
- Queries: 12 (most of any dataset)
- Databases: up to 6 databases across DuckDB, PostgreSQL, and SQLite
- Join key trap: 25% of ID fields have randomly added trailing spaces ("Lead123" vs "Lead123 "). TRIM() required before every join.
- Domain knowledge required: BANT qualification (Budget, Authority, Need, Timeline), handle time definition, sales cycle definition, transfer count policy, agent performance metrics
- Most complex dataset in DAB — 6 databases, 12 queries, heavy domain knowledge

### 4. deps_dev_v1 — NPM Package Dependencies
- Domain: software engineering
- Queries: 2
- Challenge: GitHub star counts and fork counts embedded in free-text package descriptions
- Text transformation type: data-independent (consistent format, regex works)
- Best pass@1 across all agents: 6% — second hardest dataset

### 5. github_repos — GitHub Repositories
- Domain: software engineering
- Queries: 4
- Challenge: README content analysis, commit message filtering by content, language detection
- Requires: detecting copyright information in README files, filtering commit messages that don't start with 'merge', 'update', or 'test'

### 6. googlelocal — Local Business Reviews
- Domain: local business/reviews
- Queries: 4
- Challenge: city and state location embedded in review prose, not in a dedicated column
- Text transformation type: data-independent (location extraction)

### 7. yelp — Yelp Businesses and Reviews
- Domain: local business/reviews
- Queries: 7 (second most of any dataset)
- Challenge: restaurant locations injected into review text during dataset construction — must extract from prose
- Known pattern: business attributes (WiFi, parking, credit cards) stored in nested structures
- Text transformation type: data-independent (location extraction from review text)

### 8. music_brainz_20k — Music Metadata
- Domain: music
- Queries: 3
- Challenge: data-dependent entity resolution — matching album names, release dates, and artists across sources. No fixed rules work; agent must reason about each row individually.
- Text transformation type: data-dependent

### 9. stockindex — Stock Market Indices
- Domain: financial markets
- Queries: 3
- Join key trap: full exchange names ("Tokyo Stock Exchange") must be mapped to abbreviated index symbols ("N225"). Too many to enumerate in hints — agent must infer the mapping at query time.
- Domain knowledge required: intraday volatility formula, must use `adj_close` not `close` for price comparisons
- Text transformation type: data-dependent (exchange name to symbol mapping)

### 10. stockmarket — Individual Stock Prices
- Domain: financial markets
- Queries: 5
- Scale trap: 2,754 tables — one per traded security. Highest API cost of any dataset.
- Domain knowledge required: ETF vs non-ETF classification, exchange identification (NYSE vs NASDAQ vs NYSE Arca), adjusted close requirement
- Known failure (FM4): agents query 25+ individual stocks one at a time instead of using UNION ALL — massive cost and time waste
- Must use `adj_close` not `close` for all price and volatility calculations

### 11. pancancer_atlas — Cancer Genomics
- Domain: medical research
- Queries: 3
- Domain knowledge required: CDH1 gene mutation criteria, histology type classification, log10-transformed gene expression, chi-square statistics
- Challenge: filtering patients with valid expression values and non-bracketed histology annotations

### 12. patents — Patent Filings
- Domain: intellectual property
- Queries: 3
- Score: 0% pass@1 for ALL five frontier models — completely unsolved by every agent
- Root cause (FM4): date formats like "dated 5th March 2019" and "March the 18th, 2019" cannot be handled by regex. Every agent tries regex and fails every time.
- Fix: use `dateutil.parser` or LLM-based date extraction instead of regex
- Opportunity: if your agent uses dateutil/LLM for patents dates, it will be the only agent in the benchmark that solves patents queries

```python
from dateutil import parser
date = parser.parse("dated 5th March 2019")   # works
date = parser.parse("March the 18th, 2019")   # works
# regex like \b(19\d{2}|20\d{2})\b will fail on these
```

---

## The Four Tools Every Agent Gets

```
list_db        → enumerate tables/collections in a database
query_db       → execute SQL or MongoDB query against a named database
execute_python → run Python code for data transformation and merging
return_answer  → return final answer and terminate execution
```

Each trial is capped at 100 iterations and 1 hour wall-clock time.

Tool results over 10,000 characters are truncated and written to a file. The agent must then use `execute_python` to read the full result from disk. Agents that don't do this (Gemini-2.5-Flash) return None and terminate — FM1.

---

## Agent Loop (ReAct Pattern)

```
Query arrives
    ↓
Agent reasons → issues tool call(s) → observes result → appends to context
    ↓
Repeat up to 100 iterations
    ↓
Agent calls return_answer to terminate
```

Multiple tool calls can be issued in a single iteration (parallel execution). Queries to different databases are independent and can run in parallel.

---

## Failure Mode Taxonomy (FM1–FM5)

From the paper's error analysis of 1,147 annotated failed trajectories.

### FM1 — Fails Before Planning
Agent makes no attempt to solve the query.
- FM1(no_tool_call): returns None in tool-call field → execution terminates immediately
- FM1(other): calls return_answer immediately with "I cannot solve this"
- Significant only for Gemini-2.5-Flash (63.4% of its failures)

### FM2 — Incorrect Plan (40% of completed-but-wrong trajectories)
The logical structure of the solution is wrong. Even perfect execution cannot produce the correct answer.
- Examples: averaging per-book averages instead of all ratings directly; adding LIMIT 200 when all rows are needed; missing a required filter (average rating = 5.0); stopping before all query requirements are met

### FM3 — Correct Plan, Wrong Data Selection (15% of completed-but-wrong)
Plan is right but agent queries the wrong column, table, or database.
- Example: searching for "English" in the `description` column when it's actually in the `details` column
- Least common failure mode — agents usually find the right data

### FM4 — Correct Plan and Data, Incorrect Implementation (45% of completed-but-wrong)
Plan is right, data is right, but the code is wrong.
- Examples: regex `\b(19\d{2}|20\d{2})\b` matches ISBN numbers as years; regex `MALE` matches inside "FEMALE" (missing word boundary `\bMALE\b`); joining on raw IDs without normalizing prefix differences; averaging averages instead of computing overall average
- Most common failure mode

### FM5 — Runtime Error
API failures, timeouts, token limit exceeded, 100-iteration limit hit.
- Rare for most agents; 6.6% for Kimi-K2

**The 85% rule: FM2 + FM4 = 85% of all wrong answers. FM3 = 15%. Agents are usually looking at the right data — they're computing the wrong thing or computing it incorrectly.**

---

## Key Findings from the Paper

### The 20% Exploration Rule
Agents that spend ~20% of tool calls on exploration (list_db, SELECT * LIMIT 5, schema inspection) outperform agents that spend less or more.
- Gemini-3-Pro: 20% exploration → 38% pass@1
- GPT-5-mini: 20% exploration → 30% pass@1
- Gemini-2.5-Flash: 10% exploration → 9% pass@1
- Kimi-K2: 24% but serial exploration → 23% pass@1 at 19x the cost

### Push Aggregation into SQL
GPT-5-mini averages 2.6:1 DB-to-Python ratio — pushes GROUP BY and MAX into SQL. Kimi-K2 averages 1.1:1 — fetches broad result sets and processes in Python. GPT-5-mini costs $67 total at 30% pass@1. Kimi-K2 costs $1,304 at 23%.

### Parallel Tool Calling Is Underutilized
Queries to different databases are independent and can run in parallel. Most agents parallelize in fewer than 1.5% of turns despite the capability existing. GPT-5.2 parallelizes most at 12.7% of turns.

### All Agents Use Regex for Text Extraction
No agent in the benchmark attempts dateutil, NER, or LLM-based extraction. This is why patents scores 0% and why fixing this is the single highest-value improvement available.

### Context Engineering Gap
PromptQL (with semantic layer pre-built before queries arrive) scores 51% vs 44% for the ReAct baseline using the same model (Claude-Opus-4.6). The 7-point gap comes entirely from context engineering — finding the right tables and columns before the query arrives. Datasets where the bottleneck is locating data see the largest improvements: yelp (+40pp), agnews (+35pp), stockindex (+34pp), stockmarket (+20pp).

---

## Evaluation Method

**pass@1**: fraction of queries answered correctly on the first attempt, averaged across n trials per query. Primary metric.

**Stratified averaging**: compute pass@1 per query → average across queries within each dataset → average across 12 datasets. Datasets with more queries don't get disproportionate weight.

**Validation**: ground-truth answer must appear as a substring of the agent's response. For set-valued answers, every element must appear. Deterministic — no LLM judge for scoring.

**Submission format**:
```json
[
  {
    "dataset": "bookreview",
    "query": "1",
    "run": 0,
    "answer": "2020s"
  }
]
```

One record per dataset × query × trial. Submit via GitHub PR to ucbepic/DataAgentBench.

---

## Dataset Load Order (from the Practitioner Manual)

Load in this order to match query coverage:
1. PostgreSQL databases first (most DAB queries)
2. SQLite
3. MongoDB
4. DuckDB

Start with Yelp — it contains multi-source data, nested JSON, missing values, and entity resolution challenges that mirror the full DAB problem space in a contained form.

---

## Per-Dataset Pass@1 Scores (from the Paper)

| Dataset | Gemini-3-Pro | GPT-5-mini | GPT-5.2 | Kimi-K2 | Gemini-2.5-Flash |
|---|---|---|---|---|---|
| agnews | — | — | — | — | — |
| bookreview | — | — | — | — | — |
| crmarenapro | — | — | — | — | — |
| deps_dev_v1 | 6% (best) | — | — | — | — |
| github_repos | — | — | — | — | — |
| googlelocal | — | — | — | — | — |
| music_brainz_20k | — | — | — | — | — |
| pancancer_atlas | — | — | — | — | — |
| patents | 0% | 0% | 0% | 0% | 0% |
| stockindex | — | — | — | — | — |
| stockmarket | — | — | — | — | — |
| yelp | — | — | — | — | — |
| **Overall** | **38%** | **30%** | **25%** | **23%** | **9%** |

Note: per-dataset breakdown for individual agents is in Table 3 of the paper. The table above shows confirmed values; fill in from the paper's Table 3 when running your own evaluation.

---

## Quick Reference: Dataset-Specific Fixes

| Dataset | Known trap | Fix |
|---|---|---|
| bookreview | bid_/bref_ prefix mismatch | Strip prefixes, join on numeric suffix |
| bookreview | Year extraction matches ISBNs | Test regex against sample values before using |
| crmarenapro | 25% of IDs have trailing spaces | TRIM() all ID fields before joining |
| stockmarket | 2,754 tables, serial queries | Use UNION ALL, not individual queries per stock |
| stockindex | Exchange name → symbol mapping | Infer mapping at query time, not from enumerated list |
| stockmarket/stockindex | Using raw close price | Always use adj_close for price comparisons |
| patents | Regex date extraction | Use dateutil.parser or LLM extraction |
| pancancer_atlas | MALE regex matches FEMALE | Use word boundaries: \bMALE\b |
| yelp/googlelocal | Location in review text | Extract from prose, not from dedicated column |
