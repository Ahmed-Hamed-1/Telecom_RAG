# Telecom RAG Local Setup & Walkthrough

## 1. Project Structure
The `Telecom_RAG` project is a production-grade FastAPI microservice designed for handling telecom customer complaints using LangChain, FAISS, and Gemini 3.6 Flash. The architecture separates concerns into:
- **`config/`**: Contains runtime application configs (`config.yml`).
- **`src/config/`**: Parses `config.yml` and manages `.env` secrets.
- **`src/core/`**: Core utilities, specifically `ModelFactory` for singleton model loading.
- **`src/services/`**: Business logic layer (`rag_service.py` handles LLM generation and telemetry, `ingest_service.py` manages document embeddings).
- **`src/vectorstore/`**: Repository pattern for FAISS caching and I/O.
- **`src/routers/`**: FastAPI endpoint controllers.
- **`src/models/`**: Pydantic schemas for request/response validation.

## 2. Git Setup
The repository was verified using `git status`. It is already initialized and the working tree is clean. 

## 3. Virtual Environment
Created a localized Python environment to isolate dependencies:
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```
Verified Python `3.13.5` and Pip `25.1.1` successfully resolved to the `.venv`.

## 4. Dependency Installation
Installed required packages from `requirements.txt`:
```powershell
pip install -r requirements.txt
```
All major libraries (FastAPI, Langchain, Uvicorn, Sentence-Transformers, HuggingFace Hub, FAISS-CPU) were installed successfully.

## 5. Secrets Configuration
Created a local `.env` file based on `.env.example` to inject the `GOOGLE_API_KEY`. 
Tested via `config_parser.py` to ensure the key is securely loaded without echoing it to the terminal. 

## 6. Configuration System
`config/config.yml` stores static environment settings:
- **Vector DB**: `faiss_telecom_index`
- **Models**: `huggingface` (sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2) for embeddings and `gemini` (gemini-3.6-flash) for LLM. 
- **RAG tuning**: Chunk size 500, chunk overlap 100, $k$ retrieval 20, temperature 0.

## 7. ModelFactory & `@lru_cache`
In `src/core/factories.py`, `@lru_cache(maxsize=1)` is used heavily for instantiating the Gemini LLM and HuggingFace embeddings. 
- **Why Caching?** Loading 500MB+ embedding weights repeatedly on every HTTP request would cripple latency and crash RAM. 
- **What is Cached?** The instantiated Python class references of `HuggingFaceEmbeddings` and `ChatGoogleGenerativeAI`.

## 8. Services Architecture
- **`rag_service.py`**: Interacts with the vector DB and LangChain prompt templates to formulate answers in Egyptian Arabic. It also extracts LLM metadata to calculate `prompt_tokens`, `completion_tokens`, and `execution_time_seconds`.
- **Request Flow**: `HTTP Request -> Router -> Service -> Factory (LLM/Embeddings) & Vector Database -> Response`.

## 9. FastAPI Startup
Launched the API in daemon mode using:
```powershell
python main.py
```
The server bounds to `http://0.0.0.0:8000`. On startup, it pre-heats the models (downloading HuggingFace weights if needed) and attempts to cache FAISS in RAM.

## 10. Swagger UI
The interactive Swagger documentation is available locally at:
**[http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)**

## 11. Endpoint Tests
Tested the root health endpoint `GET /` which successfully returned:
```json
{"status":"healthy","app_name":"Telecom RAG API","version":"1.0.0","docs_url":"/docs"}
```
Tested a REAL end-to-end RAG query against `POST /api/v1/query` with payload:
```json
{"ticket": "النت فاصل عندي ولمبة DSL بتنور وتطفي"}
```
Successfully retrieved a 200 OK containing an Egyptian Arabic response adhering to ISP constraints, with full telemetry:
- **Execution Time**: ~21 seconds
- **Tokens**: 2507 input, 1852 output, 4359 total
- **Sources**: 20 chunks retrieved from FAISS

## 12. Pytest Results
Invoked the existing test suite utilizing `PYTHONPATH`:
```powershell
python -m pytest tests/
```
**Results**: `3 passed, 4 warnings in 75.34s`. The mocked `/api/v1/query` payload validations and integration queries succeeded perfectly.

## 13. Security Notes
- `git status` verifies `.env` is safely ignored.
- The `faiss_telecom_index/` directory is ignored by Git, ensuring huge ML binary files and client chunks do not bloat the repository.
- API Keys are not committed.

## 14. Gemini Runtime Troubleshooting
- **Original Failure**: Calling `/api/v1/query` failed with `500 Internal Server Error` and a Google error `API key not valid` or model availability error.
- **Root Cause 1 (Model Availability)**: Google deprecated `gemini-2.5-flash` for new API project users, returning `400 INVALID_ARGUMENT`. Upgrading to `gemini-3.6-flash` resolved the Google endpoint availability.
- **Root Cause 2 (Response Data Structure)**: In `ChatGoogleGenerativeAI` with `gemini-3.6-flash`, the returned `AIMessage.content` is structured as a list of content blocks (e.g. `[{'type': 'text', 'text': '...'}]`) rather than a simple string. Assigning this directly caused a Pydantic `ValidationError` in `QueryResponse`.
- **Resolution**: Updated `src/services/rag_service.py` to extract and concatenate text blocks safely into a single string. Updated `src/core/factories.py` to pass configured `temperature` and explicit `google_api_key` while logging the exact model loaded.
- **Working Configuration**:
  - `llm_model_name: "gemini-3.6-flash"`
  - `temperature: 0`
  - `k_retrieval: 20`
- **Verification**: Verified real RAG queries return HTTP 200 with full Arabic responses and complete token metrics.

## 15. Final Run Instructions
To run the server live:
1. Ensure `.venv` is activated: `.\.venv\Scripts\Activate.ps1`
2. Configure your live `GOOGLE_API_KEY` inside `.env`
3. Boot the server: `python main.py`
4. Go to `http://localhost:8000/docs` to ingest documents and perform queries.
