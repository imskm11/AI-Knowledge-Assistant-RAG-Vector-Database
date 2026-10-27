# 📚 AI Knowledge Assistant — RAG & Vector Database

A production-style **Retrieval-Augmented Generation (RAG)** assistant that lets you upload your own PDF or TXT documents and ask natural-language questions about their contents. Answers are grounded in your documents with **source citations**, so you can verify every response.

Built with **Streamlit · LangChain · ChromaDB · HuggingFace Embeddings · Groq (GPT-OSS 20B)**.

---

## 🎯 Overview

Most "AI chatbot" projects just wrap an LLM API. This project demonstrates the **full RAG pipeline** end-to-end:

1. **Ingestion** — parse PDF/TXT into clean text
2. **Chunking** — split with overlap to preserve context
3. **Embedding** — convert chunks to vectors locally (no API cost)
4. **Vector storage** — persist embeddings in ChromaDB
5. **Retrieval** — semantic search for the top-k most relevant chunks
6. **Generation** — LLM answers using only the retrieved context
7. **Evaluation** — measure retrieval quality with Hit Rate / MRR / Recall and generation quality with LLM-as-Judge

The system is designed to **reduce hallucination** (strict prompt + "not enough info" fallback) and to be **transparent** (source chunks shown for every answer).

---

## ✨ Features

- 📄 **Upload PDF or TXT** documents via a clean Streamlit UI
- 🔍 **Semantic search** over document chunks using vector similarity
- 💬 **Conversational Q&A** with persistent chat history
- 📎 **Source citations** — every answer shows the retrieved chunks it was based on
- 🧠 **Local embeddings** (`all-MiniLM-L6-v2`) — no embedding API costs
- ⚡ **Fast inference** via Groq's free-tier `openai/gpt-oss-20b`
- 🛡️ **Hallucination guardrails** — the model is instructed to refuse when context is missing
- 📊 **Built-in evaluation** — Hit Rate@k, MRR@k, Recall@k, and LLM-as-Judge faithfulness
- 💾 **Persistent vector store** — ChromaDB saves embeddings to disk

---

## 🏗️ Architecture
## 🏗️ Architecture
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PDF / TXT │ ──▶ │ Text Split │ ──▶ │ Embed │ ──▶ │ ChromaDB │
│ Upload │ │ (chunks) │ │ (MiniLM-L6) │ │ (vector DB) │
└──────────────┘ └──────────────┘ └──────────────┘ └──────┬───────┘
│
▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Answer + │ ◀── │ LLM (Groq) │ ◀── │ Prompt │ ◀── │ Top-k │
│ Sources │ │ GPT-OSS 20B │ │ Assembly │ │ Retrieval │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

text

---

## 🗂️ Project Structure
rag-knowledge-assistant/
├── app.py # Streamlit UI (upload, chat, sources)
├── rag_pipeline.py # Core RAG logic (load, chunk, embed, retrieve, generate)
├── evaluate_retrieval.py # Offline retrieval metrics (Hit Rate, MRR, Recall)
├── evaluate_faithfulness.py # LLM-as-Judge faithfulness check
├── eval_dataset.json # Manually labeled Q&A test set
├── requirements.txt
├── .env.example
└── README.md

text

---

## 🚀 Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/imskm11/rag-knowledge-assistant.git
cd rag-knowledge-assistant
2. Create a virtual environment
bash
python -m venv venv
source venv/bin/activate        # macOS / Linux
venv\Scripts\activate           # Windows
3. Install dependencies
bash
pip install -r requirements.txt
4. Configure your API key
Copy .env.example to .env and add a free Groq API key from console.groq.com:

env
GROQ_API_KEY=your_groq_api_key_here
5. Run the app
bash
streamlit run app.py
Open the URL that Streamlit prints (usually http://localhost:8501), upload a PDF or TXT file, and start asking questions.

🧠 How It Works
Document ingestion
PyPDFLoader and TextLoader extract text from uploaded files. Metadata (page numbers, source) is preserved so we can cite it later.

Chunking strategy
RecursiveCharacterTextSplitter with chunk_size=1000 and chunk_overlap=150. The recursive splitter tries paragraph breaks first, then sentence breaks, then words — this avoids cutting critical context mid-sentence across chunks.

Embeddings
sentence-transformers/all-MiniLM-L6-v2 runs locally (~80 MB). This means:

Zero embedding API cost

No data leaves your machine during indexing

Fast CPU inference for demo-scale documents

Vector database
ChromaDB persists chunks + embeddings to ./chroma_db. Each chunk is tagged with a unique chunk_id and page number for citation and evaluation.

Retrieval
Top-k=4 chunks by cosine similarity. This is a tunable trade-off: higher k gives more context but risks diluting the prompt with irrelevant chunks.

Generation
The prompt explicitly instructs the LLM:

"If the answer is not in the context, say 'I don't have enough information in the uploaded documents to answer that.'"

This is the single most important line in a RAG prompt — it prevents the model from falling back on its pretrained knowledge.

📊 Evaluation
The project ships with two complementary evaluation scripts:

1. Offline retrieval metrics (evaluate_retrieval.py)
Runs against a hand-labeled eval_dataset.json with no LLM calls, so it's cheap to run and easy to include in CI.

bash
python evaluate_retrieval.py
Metrics computed:

Metric	What it measures
Hit Rate@k	% of questions where ≥1 relevant chunk appears in top-k
MRR@k	Mean reciprocal rank of the first relevant chunk
Recall@k	% of all relevant chunks that were retrieved
2. Faithfulness via LLM-as-Judge (evaluate_faithfulness.py)
Uses the same Groq LLM as a judge to verify that every claim in the generated answer is supported by the retrieved context.

bash
python evaluate_faithfulness.py
📈 Evaluation Results
Evaluated on a manually labeled test set of 10 questions against a business document, using k=4:

Metric	Score
Hit Rate@4	0.800
MRR@4	0.542
Recall@4	0.800
Faithfulness	0.80
💡 Analysis: The system demonstrates strong retrieval capability, successfully finding relevant context for 8 out of 10 questions. The MRR of 0.542 indicates that the relevant chunk often appears at rank 1 or 2. A faithfulness score of 0.80 confirms that the LLM is generating answers grounded in the retrieved text rather than hallucinating.

🧪 Tech Stack
Layer	Choice	Why
UI	Streamlit	Fastest path to a usable, portfolio-ready interface
Orchestration	LangChain	Clean abstractions for loaders, splitters, retrievers, chains
Vector DB	ChromaDB	Lightweight, persistent, easy to demo
Embeddings	HuggingFace all-MiniLM-L6-v2	Free, local, 384-dim, strong on semantic search benchmarks
LLM	Groq · openai/gpt-oss-20b	Fast free-tier inference, good instruction-following
Config	python-dotenv	Keeps secrets out of source control
🗺️ Roadmap / Future Work
□ Expand the evaluation set to 20+ labeled questions for larger-scale testing
□ Tune k and chunk size — current MRR suggests some relevant chunks are ranked at position 2 or 3
□ Multi-document collections with per-document filtering
□ Hybrid retrieval (BM25 + vector) for keyword-sensitive questions
□ Reranker (e.g., Cohere or bge-reranker-base) to improve top-k precision
□ Streaming token-by-token answers in the UI
□ Docker Compose deployment
□ Replace Groq with a local LLM (Ollama) for full offline operation
⚠️ Limitations
Optimized for demo-scale documents (single-file uploads). For large corpora, batch ingestion and a dedicated vector store (Qdrant, Weaviate) would be more appropriate.

Retrieval quality depends heavily on chunk size and overlap — this project uses sensible defaults but they should be tuned per domain.

No streaming yet — responses appear after full generation.

🤝 Contributing
Issues and PRs are welcome. If you fork and improve the evaluation, please share the metrics — benchmarking RAG setups publicly helps the whole community.



🙏 Acknowledgements
LangChain for the RAG building blocks

Chroma for the persistent vector store

HuggingFace for the free embedding model

Groq for affordable, fast LLM inference

Streamlit for the UI framework

👤 Author
Sumit Kr Mishra

GitHub: github.com/imskm11

LinkedIn: linkedin.com/in/imsumit-mishra

Email: skm1172000@gmail.com

Built as a portfolio project to demonstrate end-to-end RAG system design, LLM integration, and quantitative evaluation of retrieval quality.

