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

## Using the Application

The application is a two-step interface: first you build an index from your database, then you query it in natural language.

---

### Step 1: Build Your Index (`index.json`)

Before you can query, you need to index your database. Navigate to the **Build Index** screen (Step 1 in the UI).

#### What files do you need to upload?

| File | Required | Description |
|---|---|---|
| `database.sqlite` or `database.db` | ✅ Yes | Your SQLite database file |
| `<table_name>.csv` per table | ❌ Optional | Description CSVs to enrich column context |

#### What are the Description CSVs?
Each `.csv` file maps column names to human-readable descriptions. This helps the LLM understand what a column like `FRPM_Count` actually means. Each CSV should be named after its corresponding table (e.g., `schools.csv`) and must contain these columns:

```
original_column_name, value_description
```

Example `schools.csv`:
```csv
original_column_name,value_description
CDSCode,Unique identifier for each school in California
FundingType,"Type of funding: Directly funded or Locally funded"
```

> **Tip**: CSVs are optional but significantly improve query accuracy for databases with unclear column names.

#### How to build the index
1. Click **Choose File** under *SQLite Database File* and select your `.sqlite` or `.db` file.
2. (Optional) Click **Choose Files** under *Value Description CSVs* and select one or more `.csv` files.
3. Click **Upload & Build index.json**.
4. A progress bar will track the stages: reading schema, sampling values, mapping foreign keys, and writing the index.
5. Once complete, click **Continue to Query Interface**.

The generated `index.json` is saved to the `/index` directory on the server and will be used for all subsequent queries.

---

### Step 2: Querying Your Database

After building the index, you are taken to the **Query Interface** (Step 2 in the UI).

#### Typing a Query
1. Click the text box at the bottom of the screen.
2. Type your question in plain English, for example:
   - *"Which schools have the highest SAT math scores?"*
   - *"Show me the top 10 schools by enrollment in Alameda county"*
   - *"Find charter schools in San Francisco with free meal eligibility over 50%"*
3. Press **Enter** to send (or **Shift+Enter** to add a new line).
4. The assistant will reply with:
   - A natural language answer
   - The generated SQL query (with a copy button)
   - A paginated results table

#### Using Suggestion Pills
When the chat is empty, clickable example queries appear as suggestion pills. Click any of them to instantly fire a pre-written query against your database.

---

### Step 3: Voice Querying

The application supports real-time voice-to-SQL via your microphone.

#### How to use it
1. Click the **🎙️ microphone button** to the right of the text input.
   - Your browser will ask for microphone permission — click **Allow**.
2. Speak your question clearly, e.g. *"List all schools in San Francisco."*
   - A live status bubble (`🎤 Listening...`) will appear in the chat.
3. Click the **⏹ stop button** when you are done speaking.
4. Your speech is transcribed via **Whisper-large-v3** and the transcript appears in the chat bubble.
5. The query runs through the pipeline and results are returned exactly as they would be for a typed query.

> **Note**: Voice querying requires microphone access and a running backend WebSocket connection at `/api/ws-voice`.
