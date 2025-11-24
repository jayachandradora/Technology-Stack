
Below is a **clear, full, detailed ASCII diagram** showing the **Search Path Flow** in Elasticsearch — from the client → coordinator node → shards → Lucene segments → physical files → back to the client.

This is the most detailed search-path diagram you will find that fits in text format.

---

# ✅ **SEARCH-PATH DIAGRAM: Elasticsearch Query → Shards → Segments → Files → Result**

```
              +-------------------------------------------+
              |             Client / Application           |
              +------------------------+--------------------+
                                       |
                                       |  Search Query
                                       |  (HTTP: GET /index/_search)
                                       |
                            +----------v-----------+
                            |    Coordinator Node  |
                            |(Node receiving query)|
                            +----------+-----------+
                                       |
                          (Fan-out search to all shards)
                                       |
         --------------------------------------------------------------------
         |                              |                                 |
 +-------v-------+               +-------v-------+                +-------v-------+
 |   Shard 0     |               |   Shard 1     |                |   Shard 2     |
 | (P or R copy) |               | (P or R copy) |                | (P or R copy) |
 +-------+-------+               +-------+-------+                +-------+-------+
         |                               |                                 |
         |                               |                                 |
         |      Lucene Search Inside Shard (mini-index)                    |
         |-----------------------------------------------------------------|
         |                               |                                 |
    +----v----+                     +-----v-----+                     +-----v-----+
    |Segment A|                     | Segment D |                     | Segment G |
    +----+----+                     +-----+-----+                     +-----+-----+
         |                                |                                 |
         |    +----------------------------------------------+             |
         |    | Lucene Segment Internals                      |             |
         |    |-----------------------------------------------|             |
         |    | *.tim  - Term Dictionary                      |             |
         |    | *.tip  - Term Index Pointers                  |             |
         |    | *.doc  - Posting Lists (doc IDs)              |             |
         |    | *.pos  - Term Positions                       |             |
         |    | *.fdx  |                                      |             |
         |    | *.fdt  | -> Stored Fields (original _source)  |             |
         |    | *.dvd  -> Doc Values (aggregations/sorting)   |             |
         |    +----------------------------------------------+             |
         |                                |                                 |
         |  Step 1: Term Lookup (query “name:john”)                          |
         |  Step 2: Get posting list (doc IDs)                               |
         |  Step 3: Score documents (TF-IDF/BM25)                            |
         |  Step 4: Load fields if needed                                    |
         |  Step 5: Return top N hits to shard                               |
         |                                |                                 |
    +----+----+                     +-----+-----+                     +-----+-----+
    |Segment B|                     | Segment E |                     | Segment H |
    +---------+                     +-----------+                     +-----------+

                          (Each shard merges its own segment results)

         --------------------------------------------------------------------
                                       |
                                       v
                            +----------+-----------+
                            |    Coordinator Node  |
                            |  Merge shard results |
                            |  Sort by score/order |
                            |  Remove duplicates   |
                            +----------+-----------+
                                       |
                                       v
              +------------------------+--------------------+
              |             Final Results → Client          |
              +---------------------------------------------+

```

---

# 🔍 **DETAILED STEP-BY-STEP: What Happens at Each Stage**

---

## **1. Client Sends Search Query**

Example:

```
GET /products/_search?q=name:iphone
```

---

## **2. Coordinator Node Selected**

Whichever node receives the HTTP request becomes the **coordinator**.
It:

* Finds all shards belonging to the index
* Chooses one copy of each shard (primary OR replica)
* Sends query to all of them **in parallel**

---

## **3. Query Sent to All Shards**

If index has 5 shards → **5 parallel Lucene searches** happen.

This is why Elasticsearch is extremely fast.

---

## **4. Shard-Level Search (Lucene Internal)**

Inside each shard:

```
Elasticsearch shard = Lucene index
Lucene index = many immutable segments
```

For each query, shard:

1. Searches **all segments**
2. Each segment looks up:

   * `*.tim` → term dictionary (find term)
   * `*.doc` → postings list (find doc IDs)
   * `*.pos` → phrase/match queries
   * `*.fdt` → actual document stored fields
   * `*.dvd` → doc values (for aggregations)
3. Each segment returns its own top hits
4. Shard merges results from its segments
5. Sends top results to coordinator

---

## **5. Coordinator Merges Shard Results**

Coordinator:

* Collects top hits from all shards
* Sorts by score or sort order
* Merges aggregations
* Applies post-filters
* Removes duplicates
* Applies pagination (`from`, `size`, `scroll`, `search_after`)

---

## **6. Final Results Returned to Client**

---

# ⚙️ **Why Search Must Check All Shards?**

Because each shard contains a **different subset** of the documents.

If index = 3 shards:

* Shard 0 → docs 0–33%
* Shard 1 → docs 34–66%
* Shard 2 → docs 67–100%

If you only search 1 shard, you only search 1/3 of your data.

Thus Elasticsearch MUST send the query to all shards.

---

Below is a **clear, step-by-step explanation** of Elasticsearch architecture, how shards work, what they store, and how to think about scaling shards when data grows.

---

# ✅ **1. Elasticsearch Architecture (High-Level)**

Elasticsearch is a **distributed search and analytics engine**. It runs as a **cluster** of servers (nodes) that hold data and process queries.

### **Main Components**

| Component    | Explanation                                                        |
| ------------ | ------------------------------------------------------------------ |
| **Cluster**  | A collection of one or more Elasticsearch nodes                    |
| **Node**     | A single running instance of Elasticsearch (like a server)         |
| **Index**    | A logical namespace for your data (similar to a database table)    |
| **Shard**    | A physical piece of an index—this is where data is actually stored |
| **Document** | The individual record stored in Elasticsearch (JSON object)        |

---

# ✅ **2. What is a Shard? (Simple Explanation)**

A **shard is a low-level data container** that stores a subset of your index's data.

### Two types of shards:

1. **Primary shard**
2. **Replica shard** – a copy of a primary shard

---

# 🔍 **What Does a Shard Actually Store?**

| Stored in shard         | Description                                 |
| ----------------------- | ------------------------------------------- |
| 🎯 **Documents**        | Actual JSON data                            |
| 📖 **Inverted index**   | For fast full-text search                   |
| 🗂 **Doc values**       | For aggregations and sorting                |
| 🧭 **Metadata**         | Mapping, segment data, index settings       |
| 📚 **Segments**         | Lucene files under the hood                 |
| 🔍 **Index statistics** | Document counts, deleted docs, segment info |

**You NEVER manage segments manually** — Elasticsearch + Lucene manage them.

---

# ✨ **Important Concept:**

### **Each shard is a complete Lucene index**

So when you create an Elasticsearch index with 5 shards, you are creating **5 Lucene indices**, each storing part of the data.

---

# ✅ **3. Why Sharding Exists?**

Because a single server cannot scale infinitely.

Shards allow Elasticsearch to:

✔ Distribute data across nodes
✔ Parallelize search
✔ Scale horizontally
✔ Add redundancy

---

# 🧩 **4. How Elasticsearch Places Documents in Shards**

When you index a document:

```
shard_number = hash(document_id) % number_of_primary_shards
```

This means:

* Each document goes to **exactly one** primary shard
* Replica shard receives a copy

---

# 🗄 **5. How Many Shards Should an Index Have?**

This is important because:

* Too many shards → memory wasted, overhead
* Too few shards → scaling problem later

### **Default: 1 primary shard per index**

For modern Elasticsearch (7.x onwards), the default is **1 primary shard**.

---

# 🚀 **6. What Happens When Data Grows? “I Have More Data — What Should I Do?”**

You have **3 scaling options**:

---

## **Option 1 — Increase number of shards before creating the index**

Example:

```json
PUT my_index
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
```

But **you CANNOT increase primary shards later**.

---

## **Option 2 — Use Index Lifecycle Management (ILM)**

Instead of one giant index, use **daily/weekly/monthly indices**:

```
logs-2025.11.01
logs-2025.11.02
logs-2025.11.03
```

You can search them all using index patterns:

```
logs-2025.11.*
```

This is the **best approach for large-scale systems**.

---

## **Option 3 — Rollover API**

Automatically create a new index when:

* Size > 50GB
* Docs > 50 million
* Age > 1 day

Example:

```json
POST logs-write/_rollover
{
  "conditions": {
    "max_size": "50gb",
    "max_docs": 50000000
  }
}
```

---

# ❗ **Why Can't You Add Primary Shards After Index Creation?**

Because the hash routing (document → shard) would break.

---

# 🧠 **7. How Many Shards Do You Actually Need?**

### **Rule of thumb**

* One shard: **10–50 GB** → good
* Avoid more than **20 shards per node**

### For big clusters:

* Typically **1–3 shards per index** is enough
* For logs, use ILM + rollover

---

# 🔬 **8. Example: Scaling Data**

### 📌 Suppose you need to store 10 TB of logs

### Wrong approach ❌

* Create 1 index
* With 500 shards
* It will kill your cluster

### Correct approach ✔

* Use ILM
* Daily indices of size ~30–50GB
* Each index = 1 or 2 primary shards

Result:

* Cluster stays healthy
* Fast searches
* Balanced storage

---

# 📦 **9. Summary Table — Shards Explained**

| Concept           | Meaning                                                  |
| ----------------- | -------------------------------------------------------- |
| **Shard**         | A Lucene index containing part of your data              |
| **Primary shard** | Holds original data                                      |
| **Replica shard** | Copy for failover + read scaling                         |
| **Shard size**    | Ideally 20–50GB                                          |
| **More data?**    | Use ILM / rollover instead of manually increasing shards |
| **Scaling**       | Add nodes → shards redistribute automatically            |

---

Below is **both a diagram** (ASCII text diagram so it works everywhere) **and a full explanation of the search flow** — how Elasticsearch searches across shards, then inside Lucene segments, and finally reaches physical files on disk.

---

# 📌 **1. DIAGRAM: Elasticsearch Search Flow (High-Level)**

```
                   +------------------------------+
                   |        Client / App          |
                   +---------------+--------------+
                                   |
                             (HTTP Query)
                                   |
                   +---------------v--------------+
                   |       Coordinator Node       |
                   | (any node receiving request) |
                   +---------------+--------------+
                                   |
                     Fan-out search request to
                     all shards of the index
                                   |
    -----------------------------------------------------------------
    |                               |                               |
+---v---+                       +----v----+                     +----v----+
|Shard 0|                       | Shard 1 |                     | Shard 2 |
|(Primary/Replica)              |(Primary/Replica)              |(Primary/Replica)
+---+---+                       +----+----+                     +----+----+
    |                                 |                               |
    |                                 |                               |
    |                        Lucene Index (per shard)                 |
    |                        +-----------------------+                |
    |                        |      Segments         |                |
    |                        | +-------+  +-------+  |                |
    |                        | |Seg A  |  |Seg B  |  |                |
    |                        | +-------+  +-------+  |                |
    |                        +-----------------------+                |
    |                                 |                               |
    |                        Reads actual segment files               |
    |                        on disk (physical files):                |
    |                        - *.tim (terms)                          |
    |                        - *.doc (postings)                       |
    |                        - *.fdx / *.fdt (stored fields)          |
    |                        - *.dvd (doc values)                     |
    |                                 |                               |
    ---------> returns partial results to coordinator <---------------
                                   |
                   +---------------v--------------+
                   |     Coordinator Node         |
                   | Merges, sorts, reduces       |
                   +---------------+--------------+
                                   |
                                Results
```

---

# 📌 **2. Does Elasticsearch Search All Shards?**

**Yes — for search queries, Elasticsearch searches ALL relevant shards.**

Meaning:

* If your index has **5 primary shards**
* And **1 replica**
* Total shards = 10

A search request is sent to:

⭐ One copy of each shard (primary OR replica), chosen automatically.

So a 5-shard index → **5 parallel search operations**.

---

# 📌 **3. Detailed Search Flow (Step-by-Step)**

Let’s go deeper, like Elasticsearch internals:

---

## **Step 1 — Client Sends Search Query**

Example:

```
GET my_index/_search?q=user:john
```

---

## **Step 2 — Coordinator Node Chosen**

Whichever node receives your HTTP request → becomes the **coordinator node**.

It decides:

* which shards to query
* which replicas to use
* how to merge results

---

## **Step 3 — Coordinator Sends Query to Each Shard**

If index has 5 shards → coordinator sends **5 parallel queries**.

This is how Elasticsearch is fast — it uses parallel search.

---

## **Step 4 — Shard Level: Query Enters Lucene**

Inside each shard:

```
Elasticsearch shard = Lucene index
```

Lucene does:

1. Finds matching **terms** in the *inverted index*
2. Finds **doc IDs** in postings list
3. Reads stored fields or doc values if needed

---

# 📌 **4. How Lucene Finds Your Document (Deep Internal Flow)**

### **Lucene index is made of SEGMENTS**

Each shard has many segments:

```
Segment A
Segment B
Segment C
```

Every segment is a mini index.

### **Search happens in all segments:**

```
Search Query → Segment A → match?
             → Segment B → match?
             → Segment C → match?
```

---

## **Inside a Lucene Segment: Physical Files**

Each segment contains files:

| File Type   | Purpose                                    |
| ----------- | ------------------------------------------ |
| `*.tim`     | term dictionary                            |
| `*.tip`     | term index pointers                        |
| `*.doc`     | postings list (document IDs for each term) |
| `*.pos`     | term positions (for phrase search)         |
| `*.fdx/fdt` | stored fields (your original JSON)         |
| `*.dvd/dvm` | doc values (aggregations, sorting)         |

So a query like:

```
name: "john"
```

Search flow:

1. Look up `"john"` in `*.tim` file
2. Get postings list from `*.doc`
3. Retrieve docIDs
4. Load fields from `*.fdt` or doc values
5. Compute score

---

# 📌 **5. Coordinator Node Merges Results**

After each shard returns partial results (top N hits):

The coordinator:

* Merges scores
* Sorts results
* Removes duplicates
* Returns final hits

---

# 📌 **6. Summary — The Full Path**

```
Your search → Coordinator node → All shards → All segments
→ Lucene term dictionary → postings list → stored fields
→ shard-level results → coordinator → final result
```

---

# 📌 **7. Full Search Flow in One Diagram**

```
Client
  |
  v
Coordinator Node
  |
  +--> Shard 0 (Lucene)
  |       |
  |       +--> Segment A → physical files (*.tim, *.doc, *.pos...)
  |       +--> Segment B → physical files
  |
  +--> Shard 1
  |
  +--> Shard 2
  |
  ... etc
  |
  v
Coordinator merges results
  |
  v
Final response to client
```

---

# 📌 **8. Would You Like a More Visual (graphical) Diagram?**

I can generate a **flowchart image** using my diagram generator if you want an actual image file.

Options:
📘 Architecture Diagram
📗 Search Flow Diagram
📙 Shard/Segment Internal Diagram
📕 Write-Path Diagram (Indexing flow)

Below are **two detailed ASCII diagrams** as requested:

# ✅ **1. Shard / Segment Internal Diagram**

This shows **how a shard contains segments**, and how segments contain Lucene physical files (where documents, terms, postings, etc. are stored).

```
                          Elasticsearch Shard
                     (A Shard = One Lucene Index)
                     --------------------------------
                     |                              |
                     |        Lucene Segments       |
                     |    (Immutable Mini-Indexes)  |
                     |                              |
                     |   +-----------------------+  |
                     |   |      Segment A        |  |
                     |   |-----------------------|  |
                     |   | *.tim  (term dict)    |  |
                     |   | *.tip  (term index)   |  |
                     |   | *.doc  (postings)     |  |
                     |   | *.pos  (positions)    |  |
                     |   | *.fdx  (field index)  |  |
                     |   | *.fdt  (stored fields)|  |
                     |   | *.dvd  (doc values)   |  |
                     |   +-----------------------+  |
                     |                              |
                     |   +-----------------------+  |
                     |   |      Segment B        |  |
                     |   |-----------------------|  |
                     |   | same types of files   |  |
                     |   | *.tim/*.doc/*.fdt/... |  |
                     |   +-----------------------+  |
                     |                              |
                     |   +-----------------------+  |
                     |   |      Segment C        |  |
                     |   | (newest data)         |  |
                     |   +-----------------------+  |
                     |                              |
                     --------------------------------
```

### Key Points:

* **Segments are append-only** (immutable)
* New indexing → creates **new segments**
* Background merge process → combines small segments into bigger ones
* Deleting a document only marks it deleted; real deletion happens in merges

---

# ✅ **2. Write-Path Diagram (Indexing Flow)**

This shows **exactly what happens when you send a document to Elasticsearch**:

```
                         +------------------------------+
                         |       Client / Application   |
                         +---------------+--------------+
                                         |
                        (HTTP Index Request: POST /index/_doc)
                                         |
                         +---------------v--------------+
                         |     Coordinator Node         |
                         | (node receiving your write)  |
                         +---------------+--------------+
                                         |
                          1. Route Document Based on ID
                                         |
                                         v
                         +-------------------------------+
                         |      Target Primary Shard     |
                         | (computed by hash(id) % N)    |
                         +------------------+------------+
                                         |
                                         |
                       2. Write to In-Memory Buffer (RAM)
                                         |
                                         v
                         +-------------------------------+
                         |  In-Memory Indexing Buffer    |
                         |  (not flushed yet)            |
                         +-------------------------------+
                                         |
                                         |
                        3. Added to Transaction Log (WAL)
                        (Ensures durability; fsync every 1s)
                                         |
                                         v
                         +-------------------------------+
                         |   Translog (Write Ahead Log) |
                         |   translog-N.tlog             |
                         +-------------------------------+
                                         |
                                         |
                         4. Flush / Refresh Cycle:
                            - Refresh (1s): makes segment visible
                            - Flush: writes to disk & clears translog
                                         |
                                         v
               +--------------------------------------------------------+
               |                 Lucene Segment Creation                |
               | (new documents become new small segment on disk)       |
               |                                                        |
               | Segment X:                                             |
               |   *.tim  - term dictionary                             |
               |   *.doc  - postings list                               |
               |   *.fdt/.fdx - stored fields                           |
               |   *.dvd     - doc values                               |
               +--------------------------------------------------------+
                                         |
                                         |
                                5. Replica Sync
                       Coordinator sends same document
                       to all replica shards
                                         |
                                         v
                         +-------------------------------+
                         |      Replica Shards           |
                         |   Apply same write steps      |
                         +-------------------------------+
                                         |
                                         |
                                6. Segment Merging
                 Background process merges segments into larger ones
                                         |
                                         v
                         +-------------------------------+
                         |   Merged Segment (optimized)  |
                         +-------------------------------+

```

---

# 🧩 **Understanding Each Step of Write Path**

### **1. Coordinator Node Picks Primary Shard**

Uses routing formula:

```
shard = hash(document_id) % number_of_shards
```

### **2. Write goes to In-Memory Buffer**

Not searchable yet.

### **3. Translog**

Ensures durability:

* Crashes won't lose data
* Flushed every 1 second (configurable)

### **4. Refresh (makes data searchable)**

Default: every 1 second
Creates a small new segment → search can see it.

### **5. Replica sync**

Replicas repeat the exact same process.

### **6. Merge Process**

Elasticsearch tries to keep:

* Few large segments (good)
* Avoid many small segments (bad)

---

# 📌 If you want, I can also generate:

✔ **Search-path diagram** (how query flows through segments)
✔ **Cluster-level data distribution diagram**
✔ **ILM and rollover architecture diagram**
✔ **Node & shard allocation diagram**

Absolutely — here are all **three diagrams** you requested, explained clearly and designed so you can use them for architecture documentation or learning.

---

# ✅ **1. Cluster-Level Data Distribution Diagram**

This diagram shows how an Elasticsearch cluster distributes **primary** and **replica** shards across nodes for high availability and parallel search/load distribution.

```
                   +---------------------------------------------+
                   |                ES Cluster                   |
                   +---------------------------------------------+

       +--------------------+       +--------------------+       +--------------------+
       |      Node A        |       |      Node B        |       |      Node C        |
       | (Master/ Data)     |       |   (Data Node)      |       |   (Data Node)      |
       +---------+----------+       +---------+----------+       +---------+----------+
                 |                            |                            |
                 |                            |                            |
         +-------v-------+            +-------v-------+            +-------v-------+
         |  Shard 0 (P)  |            |  Shard 0 (R)  |            |               |
         +---------------+            +---------------+            |               |
                                                                    |               |
         +---------------+            +---------------+            +---------------+
         |  Shard 1 (R)  |            |  Shard 1 (P)  |            |  Shard 2 (R)  |
         +---------------+            +---------------+            +---------------+

         +---------------+            +---------------+            +---------------+
         |  Shard 2 (P)  |            |  Shard 2 (R)  |            |  Shard 0 (P)  |
         +---------------+            +---------------+            +---------------+
```

### Key Rules Illustrated:

* **Primary and replica of the same shard NEVER stay on the same node**
* Elasticsearch automatically balances shards across available nodes
* Adding a new node → cluster can redistribute shards automatically

---

# ✅ **2. ILM (Index Lifecycle Management) & Rollover Architecture Diagram**

This diagram explains how Elasticsearch automatically creates new indices when size/age exceeds thresholds, and how old indices move through hot → warm → cold → delete phases.

```
              +------------------------------------------------------+
              |        Index Lifecycle Management (ILM)              |
              +------------------------------------------------------+

         (Write Alias -> logs-write)    (Search reads all indices via logs-*)
                     |
                     v
      +------------------------------------------+
      |              Rollover Trigger            |
      |  Conditions:                             |
      |   - max_size: 50GB                       |
      |   - max_docs: 50M                        |
      |   - max_age: 1d                          |
      +------------------------------------------+

                      |
                      v

       +-------------------------+       +----------------------------+
       |   logs-000001 (hot)    | ----> | logs-000002 (hot)          |
       | (Write index active)   |       | (New write target)         |
       +-----------+-------------+       +----------------------------+
                   |
                   v
      +------------+-------------------------------------------+
      |   Move to Warm Tier (cheaper nodes)                    |
      |   - Reduced replicas                                    |
      |   - Force merge (optimize)                              |
      +------------+-------------------------------------------+
                   |
                   v
      +------------+-------------------------------------------+
      |   Move to Cold Tier (slow & cheap storage)             |
      |   - Frozen indices                                      |
      +------------+-------------------------------------------+
                   |
                   v
      +------------+-------------------------------------------+
      |   Delete Phase                                         |
      |   - Index removed based on retention policy            |
      +--------------------------------------------------------+
```

### Key Concepts:

* **Hot tier** → fresh data, high query volume
* **Warm tier** → older, rarely updated
* **Cold/Frozen tier** → very old, rarely queried
* **Rollover** → automatically creates new index based on size, age, or doc count
* **Alias logs-write** always points to the current write index

---

# ✅ **3. Node & Shard Allocation Diagram**

This shows how Elasticsearch allocates shards across nodes and how replicas ensure failover.

```
                        +----------------------------------+
                        |      Elasticsearch Cluster       |
                        +----------------------------------+

  +---------------------------+   +---------------------------+   +---------------------------+
  |         Node 1           |   |         Node 2           |   |         Node 3           |
  |  (Master + Data)         |   |        (Data)             |   |        (Data)             |
  +---------------------------+   +---------------------------+   +---------------------------+

  |  Index: orders (3 shards, 1 replica)                      |

          Index = orders
          Shards = 3 (P0, P1, P2)
          Replicas = 1 (R0, R1, R2)

   ------------------------------------------------------------------------------

  Node 1:                           Node 2:                           Node 3:
  +-------------------+             +-------------------+             +-------------------+
  | Shard 0 (Primary) |             | Shard 0 (Replica) |             | Shard 1 (Replica) |
  +-------------------+             +-------------------+             +-------------------+

  +-------------------+             +-------------------+             +-------------------+
  | Shard 1 (Primary) |             | Shard 2 (Primary) |             | Shard 2 (Replica) |
  +-------------------+             +-------------------+             +-------------------+

                                                                      +-------------------+
                                                                      | Shard 1 (Primary) |
                                                                      +-------------------+
```

### What this shows:

* Every primary shard is on a different node
* Replicas are placed on nodes **where the primary is NOT present**
* If a node fails → its primary shards are promoted from replicas

---

![image](https://user-images.githubusercontent.com/115500959/206844468-d6f4ad1b-be52-4d5f-8ac4-4ff7ea6a8db8.png)

Based on the Lucene library, Elasticsearch provides search capabilities. It provides a distributed, multitenant-capable full-text search engine with an HTTP web interface and schema-free JSON documents. The diagram below shows the outline.

𝐅𝐞𝐚𝐭𝐮𝐫𝐞𝐬 of ElasticSearch:<br>
🔹 Real-time full-text search<br>
🔹 Analytics engine<br>
🔹 Distributed Lucene<br>

ElasticSearch use cases:<br>
🔹 Product search on an eCommerce website<br>
🔹 Log analysis<br>
🔹 Auto completer, spell checker<br>
🔹 Business intelligence analysis<br>
🔹 Full-text search on Wikipedia<br>
🔹 Full-text search on StackOverflow<br>

The core of ElasticSearch lies in the data structure and indexing. It is important to understand how ES builds the 𝐭𝐞𝐫𝐦 𝐝𝐢𝐜𝐭𝐢𝐨𝐧𝐚𝐫𝐲 using 𝐋𝐒𝐌 𝐓𝐫𝐞𝐞 (Log-Strucutured Merge Tree).

**Watch this Video** : https://www.youtube.com/watch?v=NxpZyQVO0K4&ab_channel=GeorgeBridgeman
