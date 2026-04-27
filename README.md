# CHESS SQL Framework

CHESS (Contextual Harnessing for Efficient SQL Synthesis) is an end-to-end Natural Language to SQL pipeline. It allows users to interact with their SQLite databases using plain English text or voice commands, translating them into accurate SQL queries and returning the results instantly.

## Features

- **Agentic Architecture**: Utilizes a three-stage pipeline (Information Retrieval $\rightarrow$ Schema Selection $\rightarrow$ Candidate Generation) powered by `Llama-3.3-70b` (via Groq) to intelligently prune irrelevant schema context and generate highly accurate SQL.
- **Voice-to-SQL**: Real-time voice query capabilities using WebSockets and Groq's `Whisper-large-v3` for fast, accurate speech recognition.
- **Semantic Caching**: Implements Redis-backed semantic caching with `sentence-transformers/all-MiniLM-L6-v2` embeddings, resolving repeated or similar queries in under 50ms without invoking the LLM pipeline.
- **Full-Stack Application**: 
  - **Backend**: High-performance FastAPI server.
  - **Frontend**: Modern React + Vite application served via Nginx.
- **Production-Ready Deployment**: Fully containerized using Docker with Kubernetes manifests for scalable orchestration and secure namespace-scoped secrets management.

## Project Structure

- `/backend` - FastAPI server handling the core API endpoints (`/api/query`, `/api/ws-voice`, etc.).
- `/frontend` - React based User Interface.
- `/src` - Core LLM agent logic (`IR.py`, `SS.py`, `CG.py`), indexing utilities, and speech processing.
- `/templates` - Prompt templates for the LLM agents.
- `/data` - Active databases and schema descriptions.
- `/index` - Generated database indices (`index.json`) for schema pruning.

## How to Run

You can run the CHESS SQL framework entirely locally using Python/Node, or containerize it using Docker and Kubernetes.

### Prerequisites
- Groq API Key
- Redis server (local or hosted)

### Environment Setup
Create a `.env` file in the root directory:
```env
GROQ_API_KEY=your_groq_api_key
REDIS_HOST=localhost
REDIS_PASSWORD=your_redis_password
```

### Option 1: Running Locally (Development)

This is the fastest way to test changes.

**1. Start Redis**
Ensure you have a Redis instance running on port `17436` (or update `backend/sqlapp.py` to match your Redis config).
```bash
redis-server --port 17436 --requirepass "your_redis_password"
```

**2. Start the Backend (FastAPI)**
```bash
pip install -r requirements.txt
uvicorn backend.sqlapp:app --host 0.0.0.0 --port 5000
```
*The API will be available at http://localhost:5000*

**3. Start the Frontend (React)**
In a new terminal:
```bash
cd frontend
npm install
npm run dev
```
*The UI will be available at http://localhost:5173*

### Option 2: Running via Docker

You can build and run the individual Docker containers.

**1. Build the Images**
```bash
# Build backend
docker build -t chess-backend:latest .

# Build frontend
cd frontend
docker build -t chess-frontend:latest .
```

**2. Run the Containers**
```bash
# Run backend (Make sure your REDIS_HOST in .env points to a reachable Redis IP)
docker run -d -p 5000:5000 --env-file .env chess-backend:latest

# Run frontend
docker run -d -p 80:80 chess-frontend:latest
```

### Option 3: Deployment (Kubernetes)

For production-like orchestration using Minikube or Docker Desktop.

**1. Load images into your cluster** (e.g., for Minikube)
```bash
minikube image load chess-backend:latest
minikube image load chess-frontend:latest
```

**2. Deploy Secrets and Services**
```bash
kubectl create secret generic chess-secrets --from-env-file=.env
kubectl apply -f k8s-backend.yaml
kubectl apply -f k8s-frontend.yaml
```

**3. Access the Application**
```bash
kubectl port-forward svc/frontend-service 8080:80
# Access via http://localhost:8080
```

---

## Deep Dive: How it Works

The CHESS SQL framework solves the problem of "schema overflow" (feeding too much database context into an LLM) by using a multi-stage filtering pipeline.

### 1. Database Indexing
Before querying, the system creates an `index.json` representing your database. It extracts:
- All tables and columns using `PRAGMA` statements.
- **Foreign Key Relationships** to understand how tables connect.
- **Sample Values**: For low-cardinality text columns, it extracts unique values. For numeric columns, it calculates MIN, MAX, and AVG.
- **Manual Descriptions**: It merges user-provided `.csv` descriptions to give the LLM business context for specific columns.

### 2. Semantic Caching (Redis)
When a natural language query arrives, it is first embedded using the `sentence-transformers/all-MiniLM-L6-v2` model. This embedding is compared against past successful queries stored in Redis. If a query with a cosine distance $< 0.10$ is found, the system immediately returns the cached SQL, bypassing the LLMs entirely.

### 3. The Three-Stage Agentic Pipeline (Llama-3.3-70b)
If there is a cache miss, the query enters the pipeline:

* **Stage 1: Information Retrieval (IR) Agent**
  The IR agent looks at the user's raw query and extracts the core **Entities** (e.g., "Alameda county") and **Keywords** (e.g., "Charter School"). It searches the `index.json` to identify which tables *might* be relevant.

* **Stage 2: Schema Selection (SS) Agent**
  The SS agent takes the tables identified by the IR agent and further prunes them. It filters down the exact columns needed to answer the query, completely discarding irrelevant tables and columns to create a "pruned schema."

* **Stage 3: Candidate Generation (CG) Agent**
  The CG agent receives the user's query alongside the highly pruned schema (formatted as `CREATE TABLE` DDL statements) and the foreign key relationships. Because the context window is now small and highly relevant, the Llama-3 model can generate highly accurate SQL without being confused by an entire massive database schema.

### 4. Execution
The generated SQL is executed locally against the SQLite database. If successful, the result set is returned to the user, and the Natural Language $\rightarrow$ SQL pair is saved to the Redis Semantic Cache.
