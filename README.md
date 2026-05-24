# 📦 Inventory Intelligence System

> **Agentic AI + RAG for supply chain management** — built on real Uninox Houseware data.

[![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)](https://python.org)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.2-orange)](https://langchain-ai.github.io/langgraph/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-green?logo=fastapi)](https://fastapi.tiangolo.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

---

## 🎯 What This Does

A production-ready **Inventory Intelligence System** that answers natural language questions about your supply chain — *"Which SKUs need reordering?", "Forecast demand for TBP-001 next quarter", "Who is the best supplier for RSH-001?"* — by combining:

- **Agentic reasoning** (LangGraph multi-agent orchestration)
- **RAG retrieval** (ChromaDB over 200+ Purchase Order PDFs and supplier catalogs)
- **ML models** (XGBoost demand forecasting + Isolation Forest anomaly detection)
- **Real supply chain data** (Uninox Houseware — kitchenware manufacturer, Delhi)

---

## 🏗️ Architecture

```
Data Sources (6 CSVs + 207 generated PDFs)
          │
          ├──► RAG Pipeline ──────────────────────────────────┐
          │    data_generation/ → ingestion/ → ChromaDB        │
          │                                                     ▼
          └──► ML Pipeline ──────────────────────────────► LangGraph Orchestrator
               feature_store/ → XGBoost + IsolationForest     │
                                                               │
                              ┌────────────────────────────────┤
                              ▼        ▼         ▼        ▼
                           Demand  Reorder  Supplier  Anomaly
                           Agent   Agent    Agent     Agent
                              └────────────┬───────────┘
                                           ▼
                                     FastAPI (REST)
                                           │
                                     Streamlit UI
```

---

## 🗂️ Project Structure

```
inventory-intelligence/
├── config.py                        # Central settings (imports everywhere)
├── requirements.txt                 # All dependencies, Python 3.11 compatible
├── Dockerfile / docker-compose.yml  # Containerised deployment
├── .env.example                     # Copy → .env, add OPENAI_API_KEY
│
├── data/
│   ├── raw/                         # Your 6 Uninox Houseware CSVs
│   ├── synthetic/                   # 200 PO PDFs + 7 catalog PDFs (generated)
│   └── processed/                   # features.parquet, trained models, ChromaDB
│
├── data_generation/                 # Generate PDFs + extend tabular data (Faker, SDV)
├── ingestion/                       # PDF parse → chunk → deduplicate
├── knowledge/
│   ├── vector_store/                # ChromaDB: embed + retrieve
│   └── feature_store/               # Feature engineering + XGBoost + IsolationForest
│
├── orchestrator/                    # LangGraph graph + router + 6 tools
├── agents/                          # 4 specialist agents (demand/reorder/supplier/anomaly)
│
├── api/                             # FastAPI: /query, /alerts, /forecast
├── frontend/                        # Streamlit: chat + charts + alerts dashboard
│
├── scripts/                         # setup.py (one-click) | ingest_docs.py | train_models.py
├── tests/                           # pytest: test_agents.py + test_api.py
└── notebooks/                       # EDA + demand forecasting + anomaly detection
```

---

## ⚡ Quick Start

### 1. Clone and install

```bash
git clone https://github.com/yourusername/inventory-intelligence.git
cd inventory-intelligence
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

> ⚠️ `sdv` installs PyTorch (~2GB). If slow: `pip install sdv --no-deps` then install missing ones manually.

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env:
#   DEMO_MODE=true          ← works without OpenAI key
#   OPENAI_API_KEY=sk-...   ← set this to enable LLM reasoning
```

### 3. One-click setup (generates PDFs, ingests docs, trains models)

```bash
python scripts/setup.py
```

This runs all three steps automatically:
- **Step 1** — Generate 200 PO PDFs + 7 supplier catalogs
- **Step 2** — Parse → chunk → embed into ChromaDB
- **Step 3** — Train XGBoost demand model + Isolation Forest

### 4. Start the services

**Terminal 1 — API:**
```bash
uvicorn api.main:app --reload --port 8000
```

**Terminal 2 — Frontend:**
```bash
streamlit run frontend/app.py
```

**Terminal 3 — MLflow (optional):**
```bash
mlflow server --host 0.0.0.0 --port 5000
```

Open:
- 🌐 Streamlit dashboard: `http://localhost:8501`
- 📖 API docs (Swagger): `http://localhost:8000/docs`
- 📊 MLflow: `http://localhost:5000`

### 5. Or run with Docker

```bash
docker-compose up --build
```

---

## 🔌 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/ping` | Health check — returns model status |
| `POST` | `/api/query` | Natural language inventory query |
| `POST` | `/api/query/stream` | Streaming SSE version |
| `GET` | `/api/alerts` | All reorder + anomaly alerts |
| `GET` | `/api/forecast/{sku_id}` | Demand forecast with CI |
| `GET` | `/api/forecast` | Forecast for all SKUs |

### Example — POST /api/query

```bash
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "Forecast demand for TBP-001 next 3 months"}'
```

```json
{
  "question": "Forecast demand for TBP-001 next 3 months",
  "intent": "demand",
  "sku_id": "TBP-001",
  "answer": "Demand Forecast for TBP-001 (next 3 months):\n  Month +1: 342 units (CI: 279–405)\n  Month +2: 318 units (CI: 260–376)\n  Month +3: 355 units (CI: 290–420)",
  "rag_context_used": true
}
```

---

## 🧪 Running Tests

```bash
pytest tests/ -v
```

Expected: **16 tests passing** across `test_agents.py` and `test_api.py`.

---

## 🤖 Demo Mode vs LLM Mode

| Feature | `DEMO_MODE=true` (default) | `DEMO_MODE=false` + API key |
|---|---|---|
| Demand forecasting | ✅ XGBoost model | ✅ XGBoost + LLM synthesis |
| Reorder alerts | ✅ Rule-based ROP | ✅ + LLM explanation |
| Supplier lookup | ✅ Data-driven | ✅ + RAG + LLM reasoning |
| Anomaly detection | ✅ Isolation Forest | ✅ + Root cause via LLM |
| Cost | Free | ~₹2–5 per session (GPT-3.5) |

---

## 📦 Data & SKUs

**Finished goods:** SC-001 (Sheet Cutlery), TBW-001 (Thali Basket Wire), PBP-001 (Plate Basket Pipe), PNT-001 (Pantry Unit), RSH-001 (Rolling Shutter), CHM-001 (Chimney), MGC-001 (Magic Corner), and 9 more.

**Raw materials:** RM-WR-001/002 (Wire), RM-FT-001 (Fittings), RM-CB-001/002 (Couger Box).

**Suppliers:** Mehta Wire Industries, Krishna Basket Works, Gupta Modular Systems, Sharma Hardware Co., Lakshmi Rolling Shutters, Patel Chimney Solutions, ATC Magic Corners, and 5 more.

---

## 🏆 Portfolio Highlights

- **End-to-end MLOps**: data generation → feature engineering → training → serving → CI/CD
- **Domain expertise**: real supply chain logic (ROP formula, lead times, ABC classification, SAFTA docs)
- **Production patterns**: async FastAPI, Docker multi-stage build, MLflow experiment tracking, pytest with mocking
- **Agentic AI**: LangGraph StateGraph with conditional routing — not just a chatbot wrapper

---

## 🛠️ Tech Stack

```
ML:       XGBoost · ARIMA (statsmodels) · Isolation Forest · scikit-learn
AI/RAG:   LangGraph · LangChain · ChromaDB · sentence-transformers (all-MiniLM-L6-v2)
API:      FastAPI · Pydantic v2 · Uvicorn
Frontend: Streamlit · Plotly
MLOps:    MLflow · Docker · GitHub Actions
Data:     Pandas · SDV · Faker · ReportLab · PyMuPDF · datasketch
```

---

## 📄 License

MIT — use freely for learning and portfolio purposes.
