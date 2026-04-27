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

## Getting Started

### Prerequisites
- Docker and Kubernetes (e.g., minikube or Docker Desktop)
- Groq API Key

### Environment Setup
Create a `.env` file in the root directory with the following configuration:
```env
GROQ_API_KEY=your_groq_api_key
REDIS_HOST=localhost
REDIS_PASSWORD=your_redis_password
```

### Deployment (Kubernetes)
1. Start your local Kubernetes cluster.
2. Create the required secrets:
   ```bash
   kubectl create secret generic chess-secrets --from-env-file=.env
   ```
3. Apply the backend and frontend manifests:
   ```bash
   kubectl apply -f k8s-backend.yaml
   kubectl apply -f k8s-frontend.yaml
   ```

### Running Locally (Development)
1. Install backend dependencies: 
   ```bash
   pip install -r requirements.txt
   ```
2. Start a Redis server locally on port 17436 (as configured in `backend/sqlapp.py`).
3. Run the backend API: 
   ```bash
   uvicorn backend.sqlapp:app --host 0.0.0.0 --port 5000
   ```
4. Navigate to `frontend` and start the React dev server: 
   ```bash
   cd frontend
   npm install
   npm run dev
   ```

## How it Works

1. **Indexing**: The framework parses the active SQLite database and provided CSV descriptions to build an `index.json`. This index contains column details, sample values, and foreign key relationships.
2. **Querying**: 
   - A user asks a question via text or voice.
   - The system checks the **Redis Semantic Cache** for highly similar past queries.
   - On a cache miss, the **IR (Information Retrieval)** agent extracts keywords and entities.
   - The **SS (Schema Selection)** agent prunes the database schema to only the relevant tables and columns based on the keywords.
   - The **CG (Candidate Generation)** agent receives the heavily pruned schema and generates the final SQL query.
   - The SQL is executed against the database and results are returned to the user, while the successful query is cached for future use.
