# SCM Assistant — Supply Chain RAG Chatbot
### Trinamix Inc | Junior AI Engineer Hiring Task | Ref: TX-JrAI-003

---

## 🔗 Public Chatbot URL
**https://cloud.flowiseai.com/chatbot/9f67a51b-a668-4467-9728-cda7bcae5151**

> ⚠️ The chatflow is kept public and active for evaluation purposes.

---

## 📁 Repository Structure

```
scm-assistant-bot/
├── scm_assistant.json        ← Exported Flowise chatflow
├── README.md                 ← This file
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

## 🧠 Tech Stack

| Component | Choice | Details |
|---|---|---|
| **Platform** | Flowise Cloud | flowiseai.com |
| **LLM** | ChatGroq | `llama-3.3-70b-versatile` |
| **Embeddings** | HuggingFace Inference | `sentence-transformers/all-MiniLM-L6-v2` (384 dimensions) |
| **Vector Store** | Qdrant Cloud | Free tier — 1132 points, Cosine similarity |
| **Record Manager** | PostgreSQL | Neon Cloud free tier |

---

## 📂 Data Files Used

| File | Description |
|---|---|
| `supplier_performance_data.csv` | 2,000 purchase orders · 116 suppliers · 27 columns |
| `SupplyChain_Governance_Policy_v3_2_1.pdf` | 10-section supplier governance policy |

---

## ⚙️ Chunk Configurations Tested

### Configuration 1 — CSV File Loader (Initial Attempt)

| Parameter | Value |
|---|---|
| Loader | CSV File Loader |
| Chunk Size | 1000 |
| Chunk Overlap | 100 |
| Text Splitter | Recursive Character Text Splitter |
| **Result** | **Only 100/2000 rows loaded (row limit hit)** |

**Observation:** Flowise's CSV loader has a default row cap of 100 which prevents full ingestion of the 2000-row dataset. Retrieval was severely limited and answers were incomplete.

---

### Configuration 2 — Text File Loader (Final Configuration ✅)

| Parameter | Value |
|---|---|
| Loader | Text File Loader |
| File Format | Plain `.txt` (CSV converted to one record per line) |
| Chunk Size | 500 |
| Chunk Overlap | 50 |
| Separator | `\n\n` (double newline between records) |
| **Result** | **1132 chunks added successfully** |

**Observation:** Converting the CSV to a plain text file with `\n\n` as separator allowed each PO record to become its own chunk. Smaller chunk size (500) kept records within token limits while preserving all 27 fields per record. This significantly improved retrieval accuracy over Config 1.

---

## 🗄️ Vector Store Details

| Setting | Value |
|---|---|
| Collection Name | `scm-collection` |
| Total Points | 1132 |
| Vector Dimensions | 384 |
| Similarity Metric | Cosine |
| Status | 🟢 GREEN |

---

## 💬 Sample Q&A — Verbatim Expected Answers

### Q1: Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?

**Answer:**
11 Tier-3 suppliers: Dravex Components India, Plataforma Metales SA, Maghreb Castworks, Helios Pack Greece, Cerromax Mineria, Orinoco Pack SAPI, Quetzal Textiles, Sibertek Molding, Archipelago PCB Corp, Varna Electronics EAD, Deltaforge Vietnam. All are High Risk with an active flag → Level 3 Activate per Policy §9 (CPO escalation + alternate supplier at minimum 40% volume).

---

### Q2: Which suppliers qualify for the annual Volume Rebate Program and how many are there?

**Answer:**
19 suppliers qualify: Borealis Composites, Crestline Chemical Supply, Fenwick Alloy Solutions, Hanguk Circuit Works, Hokkaido Alloy Tech, Krauss-Polymex GmbH, Lakeshore Components, Lumivex Semiconductor NL, Maplewood Polymer Corp, Norbec Alloy Works, Nordloom Finland Oy, Orrentek Precision Mfg, Ostwind Composites AG, PrecisionForge Taiyuan, Solveig Eco Packaging, Straits Packaging Hub, Tasman Circuit Boards, Toreval Electronics, Valdoro Special Alloys. Criteria (Policy §4.2): Tier-1 + OTD ≥ 93% + Defect < 0.5% + Sustainability Score ≥ 85.

---

### Q3: Which region has the highest total PO value, and does it breach the concentration limit?

**Answer:**
EMEA at $193,987,179.91 — approximately 48.5% of total spend ($399,563,494.10). This breaches the 45% regional concentration cap (Policy §5.3), requiring a Diversification Plan within 60 days.

---

### Q4: Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?

**Answer:**
11 suppliers (Compliance Score < 60): Deltaforge Vietnam, Maghreb Castworks, Helios Pack Greece, Cerromax Mineria, Orinoco Pack SAPI, Varna Electronics EAD, Quetzal Textiles, Plataforma Metales SA, Archipelago PCB Corp, Dravex Components India, Sibertek Molding. SWL restricts new PO issuance to 20% of prior quarter volume (Policy §3.4).

---

### Q5: Which product category has the highest average defect rate and does it exceed the Tier-2 limit?

**Answer:**
Mechanical Components — average 2.12% across 360 POs. Below the Tier-2 ceiling of 2.50% (Policy §3.2), so no breach — but approaching the limit.

---

## 🏗️ Chatflow Architecture

```
[HuggingFace Inference Embedding]
  Model: sentence-transformers/all-MiniLM-L6-v2
           │
           ▼ (Embeddings)
      [Qdrant Vector Store]
  Collection: scm-collection | Dim: 384 | Cosine
           │
           ▼ (Vector Store Retriever)    [ChatGroq LLM]
                                    Model: llama-3.3-70b-versatile
                                           │
           ┌────────────────────────────────┘
           ▼
[Conversational Retrieval QA Chain]
  Top K: 6 | Return Source Documents: ON
           │
           ▼
      [Chat Output]
```

---

## 🚀 How to Run Locally

### Prerequisites
- Flowise installed (`npm install -g flowise`)
- API keys: HuggingFace, Qdrant, Groq, Neon PostgreSQL

### Steps
1. Clone this repo
2. Start Flowise: `npx flowise start`
3. Go to `http://localhost:3000`
4. Import `scm_assistant.json` via **Chatflows → Import**
5. Add your credentials in **Settings → Credentials**
6. Open Document Store and re-upsert with your own Qdrant instance
7. Start chatting!

---

## 🔧 What I'd Improve

1. **Full 2000-row ingestion** — Split the CSV into per-supplier text files so all PO records are indexed without row limits.

2. **Hybrid search (BM25 + Vector)** — Combine keyword search for exact supplier names/IDs with semantic search for policy questions, improving precision on both query types.

3. **Metadata filtering** — Tag each chunk with structured metadata (Contract_Tier, Region, Risk_Level) and use Qdrant's payload filters to narrow retrieval before semantic search.

4. **Dedicated analytics tool** — For aggregation questions (total PO value by region, average defect rate), route to a Python/SQL analytics agent instead of relying on RAG which can only see sampled chunks.

5. **Query reranking** — Add a cross-encoder reranker (e.g. `cross-encoder/ms-marco-MiniLM-L-6-v2`) to reorder retrieved chunks by relevance before passing to the LLM.

6. **Larger context model** — Use a model with 32k+ token context window to process more retrieved chunks per query without hitting token limits.

7. **Conversational memory** — Add a persistent memory buffer so follow-up questions can reference earlier answers in the same session.

---

## 📸 Screenshots

| Screenshot | Description |
|---|---|
| `01_document_store.png` | Document Store with both files loaded and chunk counts |
| `02_chatflow_canvas.png` | Full chatflow with all 4 nodes connected |
| `03_chunk_config_1.png` | Config 1: CSV loader, chunk 1000/100 |
| `04_chunk_config_2.png` | Config 2: Text file loader, chunk 500/50 |
| `05_test_questions.png` | Chat panel showing Q&A in action |
| `06_public_chatbot.png` | Public chatbot URL verified in browser |
| `07_qdrant_collection.png` | Qdrant dashboard: 1132 points, 384 dim, GREEN |

---

## 📋 .gitignore

```
.env
*.key
api_keys.txt
secrets/
node_modules/
__pycache__/
*.pyc
```

---

*Submitted by: Soumadip (Roll No: 2022ITB097)*  
*Trinamix Inc · Talent Acquisition · Ref: TX-JrAI-003*
