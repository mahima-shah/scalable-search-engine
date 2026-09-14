# Scalable Search Engine

A distributed search engine built over a collection of **28,000+ HTML documents**, designed to turn a large set of web pages into fast, relevant search results.

The system preprocesses documents into a searchable index, distributes that index across multiple servers, and ranks results using both **TF-IDF** and **PageRank**.

**Tech:** Python · Flask · MapReduce · REST APIs · SQLite · HTML/CSS

> **Note:** This repository contains project documentation only. Source code is kept private in accordance with University of Michigan academic integrity policies.

---

## Overview

A search engine has two very different jobs.

First, it needs to process a large collection of documents and organize them so they can be searched efficiently. Then, when someone enters a query, it needs to quickly find matching pages and decide which results should appear first.

This project handles both.

```text
                         BUILDING THE INDEX

                       28,000+ HTML pages
                               │
                               ▼
                        Process documents
                               │
                               ▼
                       MapReduce pipeline
                               │
                               ▼
                         Inverted index
                               │
                               ▼
                    Split into index segments


                           SERVING A SEARCH

                           User query
                               │
                               ▼
                         Search Server
                               │
                      concurrent requests
                      ┌────────┼────────┐
                      ▼        ▼        ▼
                  Index 1   Index 2   Index 3
                      │        │        │
                      └────────┼────────┘
                               ▼
                        Combine results
                               │
                               ▼
                      Return top results
```

The result is a service-oriented search system that separates the expensive work of processing documents from the work needed to answer a user's search.

---

## 1. Turning Thousands of Pages Into Something Searchable

The first problem is scale.

If the search engine opened and scanned all 28,000+ documents every time someone entered a query, it would repeat a huge amount of work.

Instead, the documents are processed **before any searches happen**.

### Inverted Index

The core data structure is an **inverted index**.

It works similarly to the index at the back of a textbook. If you want to find every page discussing "algorithms," you do not reread the entire textbook. You look up "algorithms" in the index and immediately get the relevant page numbers.

A search engine does something similar:

```text
algorithm  ──→  Document 8, Document 42, Document 91
database   ──→  Document 3, Document 42, Document 77
python     ──→  Document 8, Document 19, Document 64
```

The real index also stores information needed to calculate how strongly each document matches a query.

This shifts much of the expensive computation to the indexing stage, allowing searches to work from structured data instead of repeatedly scanning the original documents.

---

## 2. Building the Index with MapReduce

Processing thousands of documents is itself a large data-processing problem.

The indexing pipeline uses **MapReduce** to divide that work into smaller stages.

At a high level:

```text
             Large document collection
                       │
                       ▼
                Map operations
              process documents
                       │
                       ▼
             Shuffle / group data
                       │
                       ▼
               Reduce operations
              aggregate information
                       │
                       ▼
                 Searchable index
```

Rather than treating the entire corpus as one large sequential job, MapReduce provides a way to distribute and organize the processing needed to construct the index.

The completed inverted index is then divided into **three segments**, with each segment served independently by an Index server.

```text
                     Inverted Index
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
            Segment 1  Segment 2  Segment 3
                │          │          │
                ▼          ▼          ▼
             Index      Index      Index
            Server 1   Server 2   Server 3
```

This means one server does not need to handle the entire index by itself.

---

## 3. Deciding Which Results Are Relevant

Finding documents containing a query is only part of search.

The harder question is:

**Which matching documents should appear first?**

The ranking system uses two signals.

### TF-IDF: How well does the page match the query?

**TF-IDF** measures how useful a term is for identifying a particular document.

The intuition is simple:

- A word becomes more meaningful when it appears prominently in a particular document.
- A word becomes less useful when it appears in almost every document.

For example, if someone searches:

```text
machine learning
```

a page that discusses those terms heavily should generally be considered more relevant than a page that mentions them only once.

TF-IDF provides the search engine with a measure of this **query relevance**.

### PageRank: How important is the page?

Matching the query still does not necessarily make a page a good result.

The project also uses **PageRank**, which estimates the importance of a page using the links between documents.

The basic intuition is that a page linked to by other important pages should itself receive more importance than an isolated page.

So the search engine considers two different questions:

```text
            TF-IDF
   "How well does this page
      match the query?"
             │
             │
             ▼
        Ranking Score
             ▲
             │
             │
           PageRank
     "How important is
        this page?"
```

Combining the two allows results to account for both **what a page says** and **how important that page is within the document network**.

---

## 4. What Happens When Someone Searches?

Once the index has been built, the system needs to answer queries quickly.

Suppose someone searches:

```text
machine learning
```

The request follows this path:

```text
                       "machine learning"
                               │
                               ▼
                     ┌───────────────────┐
                     │   Search Server   │
                     │      Flask        │
                     └─────────┬─────────┘
                               │
                      concurrent requests
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
          ┌────────────┐ ┌────────────┐ ┌────────────┐
          │   Index    │ │   Index    │ │   Index    │
          │  Server 1  │ │  Server 2  │ │  Server 3  │
          └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
                 │              │              │
                 │       ranked matches       │
                 └──────────────┼──────────────┘
                                ▼
                       Merge the results
                                │
                                ▼
                       Retrieve document
                          information
                                │
                                ▼
                        Display top results
```

### Step 1: Receive the query

The user-facing **Search server** receives the search terms.

### Step 2: Query the index

The Search server sends requests to all three Index servers through **REST APIs**.

These requests are made concurrently rather than waiting for one server to finish before contacting the next.

### Step 3: Score matching documents

Each Index server searches its portion of the inverted index and calculates scores for matching documents using TF-IDF and PageRank.

### Step 4: Merge the results

The Search server receives ranked matches from the Index servers and combines them into one result set.

### Step 5: Display the results

Information about the highest-ranked documents is retrieved and used to render the final search results for the user.

---

## 5. System Architecture

The application separates searching the index from presenting search results.

### Index Service

Three Flask Index servers are responsible for working with the inverted index.

Each server:

- loads one segment of the index
- receives queries through a REST API
- identifies matching documents
- calculates ranking scores
- returns its highest-ranked matches

### Search Service

The Flask Search server is responsible for the user-facing application.

It:

- accepts the user's search query
- sends concurrent requests to the Index servers
- merges their responses
- retrieves information about matching documents
- renders the final results

The separation creates a **service-oriented architecture**:

```text
                    USER-FACING SERVICE
                    ┌─────────────────┐
                    │  Search Server  │
                    └────────┬────────┘
                             │
                         REST APIs
                             │
               ┌─────────────┼─────────────┐
               │             │             │
               ▼             ▼             ▼
          ┌─────────┐   ┌─────────┐   ┌─────────┐
          │ Index 1 │   │ Index 2 │   │ Index 3 │
          └─────────┘   └─────────┘   └─────────┘
                  SEARCH / INDEX SERVICE
```

Each part of the system has a focused responsibility, while the services communicate through defined API interfaces.

---

## Technical Concepts

| Concept | Role in the Search Engine |
|---|---|
| **MapReduce** | Processes the document collection and helps construct the search index |
| **Inverted Index** | Maps terms to the documents that contain them |
| **TF-IDF** | Measures how relevant documents are to a query |
| **PageRank** | Represents the relative importance of documents using links |
| **Flask** | Powers the Search and Index web services |
| **REST APIs** | Allow the Search server and Index servers to communicate |
| **Concurrency** | Allows multiple Index servers to be queried without waiting for each sequentially |
| **SQLite** | Stores document information used by the Search service |

---

## What I Took Away From It

The most interesting part of this project was seeing how several concepts that I had learned separately fit together behind something as familiar as a search box.

The final system combines **data processing, information retrieval, algorithms, APIs, concurrency, databases, and web development** in one application.

It also made an important systems idea much more concrete for me: a lot of the work required to make a user-facing operation feel fast can happen long before the user performs that operation.

Instead of doing expensive document processing for every search, the system performs that work ahead of time, stores the results in an index, distributes that index across services, and leaves the query-time system with a much smaller problem to solve.

---

## Project Context

This project was built with a team as part of **EECS 485: Web Systems** at the **University of Michigan**.

This public repository is intentionally documentation-only. The implementation and course-provided materials are not included in accordance with University of Michigan academic integrity policies.
