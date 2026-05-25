# SCM Assistant — Supply Chain Chatbot

**Trinamix Inc — Junior AI Engineer Hiring Task | Ref: TX-JrAI-003**

---

## Public Chatbot URL

https://cloud.flowiseai.com/chatbot/9f67a51b-a668-4467-9728-cda7bcae5151

The chatflow is kept public and active. Please do not evaluate before checking this link in an incognito browser first.

---

## Repository Structure

```
scm-assistant-bot/
├── scm_assistant.json
├── README.md
├── .gitignore
└── screenshots/
    ├── 01_document_store.png
    ├── 02_chatflow_canvas.png
    ├── 03_chunk_config_1.png
    ├── 04_chunk_config_2.png
    ├── 05_test_questions.png
    ├── 06_public_chatbot.png
    └── 07_qdrant_collection.png
```

---

## What I Used and Why

**LLM — ChatGroq (`llama-3.3-70b-versatile`)**
I went with Groq because it's free and significantly faster than most alternatives I tried. I initially tested a few other models but ran into either token limit issues or model deprecation errors. `llama-3.3-70b-versatile` handled multi-part supply chain questions reasonably well within the free tier constraints.

**Embeddings — HuggingFace (`sentence-transformers/all-MiniLM-L6-v2`)**
This model produces 384-dimensional vectors and works well for semantic similarity on structured business text. It's free, no billing required, and integrates cleanly with Flowise. I didn't use OpenAI embeddings to keep the stack fully free.

**Vector Store — Qdrant Cloud (Free Tier)**
Qdrant's free cloud tier gives 1GB storage with no credit card needed. It supports cosine similarity out of the box which is better suited for text embeddings than euclidean distance. The Flowise integration was straightforward.

**Record Manager — PostgreSQL via Neon**
Neon offers a free PostgreSQL instance that works as Flowise's record manager. This prevents duplicate chunks when re-upserting — without it, every upsert would pile on top of the previous one.

---

## Data Files

| File | Contents |
|---|---|
| `supplier_performance_data.csv` | 2,000 purchase orders, 116 suppliers, 27 columns including OTD rate, defect rate, compliance score, risk level, PO value and more |
| `SupplyChain_Governance_Policy_v3_2_1.pdf` | 10-section internal governance document covering tier thresholds, SLAs, penalties, audit rules, and disruption response procedures |

---

## Chunk Configurations Tested

### Config 1 — CSV File Loader

| Setting | Value |
|---|---|
| Loader | CSV File Loader (native Flowise) |
| Chunk Size | 1000 |
| Chunk Overlap | 100 |
| Splitter | Recursive Character Text Splitter |
| Chunks Created | 100 out of 2000 rows |

This was my first attempt. I loaded the CSV directly using Flowise's built-in CSV loader and set a chunk size of 1000 with 100 overlap. The problem was that Flowise's CSV loader silently caps at 100 rows by default. Only 100 of the 2000 purchase order records actually got embedded, which meant the chatbot was essentially blind to 95% of the data. Answers were partial and often wrong on count-based questions.

---

### Config 2 — Text File Loader (what I ended up using)

| Setting | Value |
|---|---|
| Loader | Text File Loader |
| File Format | Plain `.txt` — one PO record per block |
| Chunk Size | 500 |
| Chunk Overlap | 50 |
| Separator | `\n\n` |
| Chunks Created | 1132 |

**Why I switched to a text file:**

After hitting the CSV row cap, I converted the entire CSV to a plain `.txt` file where each of the 2000 purchase order records is written as a single line of pipe-separated key-value pairs, separated from the next record by a blank line (`\n\n`). This way the text splitter treats each PO block as a natural boundary and splits cleanly — one chunk per record, more or less.

Chunk size 500 was chosen because a single PO record with all 27 fields is roughly 400-500 characters. Setting it any lower would split records mid-field; any higher would merge multiple records into one chunk, making retrieval noisy. Overlap of 50 gives just enough context bleed between chunks without inflating the index size too much.

The result — 1132 chunks — is lower than the 2000 records because some records fell close to chunk boundaries and got merged. Still, going from 100 to 1132 was a huge improvement in retrieval coverage.

---

## Vector Store

| Setting | Value |
|---|---|
| Collection | scm-collection |
| Points | 1132 |
| Dimensions | 384 |
| Similarity | Cosine |
| Status | GREEN |

---

## Chatflow

The flow has four nodes connected in sequence:

```
[HuggingFace Inference Embedding]
   sentence-transformers/all-MiniLM-L6-v2
            |
            | embeddings
            v
       [Qdrant Retriever]
   collection: scm-collection
   dimensions: 384 | similarity: cosine
   top_k: 6
            |
            | retrieved chunks
            v
[Conversational Retrieval QA Chain] <--- [ChatGroq]
   return source documents: ON              llama-3.3-70b-versatile
            |
            v
       [Response to user]
```

The HuggingFace node generates query embeddings at runtime using the same model used during indexing. Qdrant retrieves the top 6 most semantically similar chunks. Those chunks plus the user question go into the QA Chain which uses ChatGroq to generate the final answer. Source documents are returned so you can see which chunks were retrieved.

Top K is set to 6 rather than a higher number because Groq's free tier has a 6000 TPM limit — retrieving more chunks would exceed it and return a 413 error.

---

## Sample Questions and Answers

**Q1: Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?**

11 Tier-3 suppliers: Dravex Components India, Plataforma Metales SA, Maghreb Castworks, Helios Pack Greece, Cerromax Mineria, Orinoco Pack SAPI, Quetzal Textiles, Sibertek Molding, Archipelago PCB Corp, Varna Electronics EAD, Deltaforge Vietnam. All are High Risk with an active flag → Level 3 Activate per Policy §9 (CPO escalation + alternate supplier at minimum 40% volume).

---

**Q2: Which suppliers qualify for the annual Volume Rebate Program and how many are there?**

19 suppliers qualify: Borealis Composites, Crestline Chemical Supply, Fenwick Alloy Solutions, Hanguk Circuit Works, Hokkaido Alloy Tech, Krauss-Polymex GmbH, Lakeshore Components, Lumivex Semiconductor NL, Maplewood Polymer Corp, Norbec Alloy Works, Nordloom Finland Oy, Orrentek Precision Mfg, Ostwind Composites AG, PrecisionForge Taiyuan, Solveig Eco Packaging, Straits Packaging Hub, Tasman Circuit Boards, Toreval Electronics, Valdoro Special Alloys. Criteria (Policy §4.2): Tier-1 + OTD ≥ 93% + Defect < 0.5% + Sustainability Score ≥ 85.

---

**Q3: Which region has the highest total PO value, and does it breach the concentration limit?**

EMEA at $193,987,179.91 — approximately 48.5% of total spend ($399,563,494.10). This breaches the 45% regional concentration cap (Policy §5.3), requiring a Diversification Plan within 60 days.

---

**Q4: Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?**

11 suppliers (Compliance Score < 60): Deltaforge Vietnam, Maghreb Castworks, Helios Pack Greece, Cerromax Mineria, Orinoco Pack SAPI, Varna Electronics EAD, Quetzal Textiles, Plataforma Metales SA, Archipelago PCB Corp, Dravex Components India, Sibertek Molding. SWL restricts new PO issuance to 20% of prior quarter volume (Policy §3.4).

---

**Q5: Which product category has the highest average defect rate and does it exceed the Tier-2 limit?**

Mechanical Components — average 2.12% across 360 POs. Below the Tier-2 ceiling of 2.50% (Policy §3.2), so no breach — but approaching the limit.

---

## What I'd Improve

A few things I'd do differently with more time:

The biggest limitation right now is that only about 1132 of the 2000 PO records made it into the index. Ideally I'd write a small preprocessing script to split the CSV into individual per-supplier files so every record gets its own clean chunk with no merging.

For questions that require aggregation across the whole dataset — like total PO value by region or average defect rate — RAG isn't really the right tool. It can only sample what it retrieves. A better approach would be to route those queries to a Python/Pandas agent that runs directly against the full CSV and computes exact figures.

I'd also look at adding metadata filtering in Qdrant. Right now every query searches the whole collection. If chunks were tagged with fields like Contract_Tier, Region, and Risk_Level, you could filter down to only Tier-3 chunks before semantic search — much more precise.

Hybrid search combining vector similarity with BM25 keyword matching would also help on questions that mention specific supplier names or IDs, where exact keyword matching outperforms pure semantic search.

---

## How to Run This Locally

You'll need Node.js, and API keys for HuggingFace, Groq, Qdrant, and Neon PostgreSQL.

```bash
npm install -g flowise
npx flowise start
```

Then open `http://localhost:3000`, go to Chatflows, and import `scm_assistant.json`. Add your credentials under Settings, then go to Document Stores and re-upsert the data files against your own Qdrant instance.

---

## .gitignore

```
.env
*.key
api_keys.txt
secrets/
node_modules/
__pycache__/
```

---

Submitted by Soumadip — Roll No. 2022ITB097
Trinamix Inc · Talent Acquisition · Ref: TX-JrAI-003
