# Local RAG System - Ollama + AnythingLLM + ChromaDB

A fully containerized, zero-configuration Retrieval-Augmented Generation (RAG) system that runs entirely offline with no API keys or external dependencies. Upload documents, ask questions, and receive AI-generated answers grounded in your own data.

## Overview

This project provides a complete, self-hosted RAG pipeline using three industry-standard open-source tools:

- **Ollama** - Local LLM runtime for serving models like Llama, Mistral, and DeepSeek
- **AnythingLLM** - Web-based interface for document management, chat, and workspace organization
- **ChromaDB** - Vector database for semantic search and efficient retrieval of embeddings

All services are containerized using Docker Compose, enabling a one-command deployment that requires no manual configuration or code modification.

## Architecture

The system architecture follows a modular pipeline pattern, with each component running in isolation within its own Docker container. The flow of data is as follows:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER INTERFACE                                 │
│                        AnythingLLM (Port 3001)                              │
│                  Document upload & chat interface                           │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    │ 1. Document uploaded via web UI
                                    │ 2. Text extracted and chunked
                                    │ 3. Each chunk sent for embedding
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EMBEDDING SERVICE                                 │
│                          Ollama (Port 11434)                                │
│                  Converts text chunks to vector embeddings                  │
│                  using models like nomic-embed-text                         │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    │ 4. Vectors stored with metadata
                                    │ 5. Indexed for semantic search
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           VECTOR DATABASE                                   │
│                           ChromaDB (Port 8000)                              │
│                   Persistent storage and retrieval of                       │
│                   document embeddings                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow (Document Upload Process)

1. **Document Submission**: A user uploads a document (PDF, DOCX, TXT, CSV, etc.) through the AnythingLLM web interface.

2. **Text Extraction and Chunking**: AnythingLLM parses the document, extracts the textual content, and divides it into semantic chunks. This chunking strategy ensures compatibility with the LLM's context window limitations and enables granular retrieval.

3. **Embedding Generation**: Each text chunk is passed to Ollama, where a configured embedding model (default: `nomic-embed-text`) converts the text into a high-dimensional vector representation.

4. **Vector Storage**: The generated embeddings, along with their associated metadata (source document, chunk position, etc.), are stored in ChromaDB. ChromaDB indexes the vectors to facilitate efficient nearest-neighbor searches.

### Data Flow (Query and Generation Process)

1. **User Query**: The user submits a question via the chat interface.

2. **Query Embedding**: AnythingLLM converts the user's question into a vector using the same embedding model.

3. **Semantic Search**: ChromaDB performs a similarity search to retrieve the most semantically relevant document chunks.

4. **Context Assembly**: The retrieved chunks are assembled into a context prompt.

5. **Response Generation**: Ollama's LLM (e.g., `llama3.1`, `mistral`) generates a response conditioned on the provided context and the user's query.

6. **Response Delivery**: The generated response is streamed back to the user through the web interface.

## Prerequisites

Ensure your system meets the following requirements before proceeding:

| Requirement | Specification |
| :--- | :--- |
| **Operating System** | Linux, macOS, or Windows with WSL2 |
| **Docker** | Docker Engine 20.10+ or Docker Desktop 4.0+ |
| **Docker Compose** | V2 (included with Docker Desktop) |
| **RAM** | 8GB minimum, 16GB+ recommended |
| **Storage** | 10GB+ free space for models and vector data |
| **GPU (Optional)** | NVIDIA GPU with CUDA support recommended for accelerated inference |

## Installation and Setup

### Step 1: Clone the Repository

```bash
git clone <your-repository-url>
cd local-rag-llm-stack
```

### Step 2: (Optional) Configure the LLM Model

Open the `docker-compose.yml` file and locate the Ollama service section. The default model is set to `llama3.1`. To use a different model, update the `command` field:

```yaml
ollama:
  image: ollama/ollama:latest
  command: serve
  # To pull a specific model, add:
  # command: serve && ollama pull mistral
```

Alternatively, you can pull models after the initial startup using the Docker exec command.

### Step 3: Deploy the Stack

Run the following command to build and start all containers:

```bash
docker compose -f docker_compose.yml -p local-rag-llm up --build
```

**Note**: The first deployment will take several minutes as Docker pulls the required images and initializes the containers.

### Step 4: Access the Application

Once all containers are running, open your browser and navigate to:

```
http://localhost:3001
```

## Configuration and Usage Guide

### Workspace Creation

1. Navigate to `http://localhost:3001`
2. Click the "New Workspace" button
3. Provide a name and description for your workspace
4. Select your preferred LLM and embedding model from the dropdown menu
5. Confirm the workspace creation

### Uploading Documents

1. Within your workspace, locate the document upload section
2. Click the upload icon or drag-and-drop your files
3. Supported file formats include: PDF, DOCX, TXT, CSV, HTML, and more
4. AnythingLLM will automatically process, chunk, and embed the document
5. For large documents, you may be prompted to confirm embedding via a "Save and Embed" button

### Querying Your Data

1. Navigate to the chat interface within your workspace
2. Enter your question in the chat input field
3. The system will retrieve relevant context from your documents and generate a response
4. All interactions are stored locally and persist across sessions

### Stopping the Services

To gracefully stop all containers:

```bash
docker compose -f docker_compose.yml -p local-rag-llm down
```

To completely remove all containers, volumes, and networks:

```bash
docker compose -f docker_compose.yml -p local-rag-llm down -v
```

## Model Customization

### Available Models

You can use any model supported by Ollama. Popular options include:

- `llama3.1`: Meta's latest 8B parameter model
- `llama3.2`: Lightweight 3B and 1B versions
- `mistral`: 7B parameter model
- `deepseek-r1`: DeepSeek's reasoning model
- `gemma2`: Google's 9B and 27B parameter models

### Pulling Additional Models

To download a model after the stack is running:

```bash
docker exec -it ollama ollama pull <model-name>
```

To update the default model in the configuration, modify the `OLLAMA_MODEL` environment variable in the `docker_compose.yml` file.

## Persistent Storage

All data is persisted through Docker volumes. The following volumes are configured:

| Volume | Mount Path | Purpose |
| :--- | :--- | :--- |
| `ollama_data` | `/root/.ollama` | Stores downloaded models |
| `anythingllm_storage` | `/app/storage` | Stores uploaded documents and workspace data |
| `chromadb_data` | `/chroma/chroma` | Stores vector embeddings and indices |

These volumes ensure that models, documents, and embeddings survive container restarts and updates.

## Hardware Considerations

### Minimum Requirements

- **CPU**: Any modern x86_64 processor with 4+ cores
- **RAM**: 8GB (sufficient for small models like `llama3.2:3b`)
- **Storage**: 10GB for base setup, additional space for models and documents

### Recommended Specifications for Optimal Performance

- **GPU**: NVIDIA GPU with at least 4GB VRAM (enables hardware acceleration)
- **RAM**: 16GB or more
- **Storage**: 50GB+ for multiple models and large document collections

### Performance Considerations

| Model Size | Approximate RAM Usage | Inference Speed (CPU) | Inference Speed (GPU) |
| :--- | :--- | :--- | :--- |
| 1B parameter | 2-3GB | ~3-5 tokens/sec | ~15-25 tokens/sec |
| 3B parameter | 4-6GB | ~5-10 tokens/sec | ~30-50 tokens/sec |
| 7B parameter | 8-12GB | ~1-3 tokens/sec | ~20-40 tokens/sec |
| 8B parameter | 10-14GB | ~1-2 tokens/sec | ~15-30 tokens/sec |

## Troubleshooting

### Common Issues

**Issue**: Port 3001 is already in use
- **Solution**: Modify the port mapping in the `docker_compose.yml` file, or stop the conflicting service

**Issue**: Model fails to load
- **Solution**: Ensure sufficient RAM is available. Try switching to a smaller model.

**Issue**: Slow response times
- **Solution**: Reduce the model size, or ensure GPU acceleration is properly configured

**Issue**: Docker services fail to start
- **Solution**: Verify Docker is running and ensure all ports are available

## License

This project orchestrates third-party open-source software. Each component is governed by its respective license:

- [Ollama](https://github.com/ollama/ollama) - MIT License
- [AnythingLLM](https://github.com/Mintplex-Labs/anything-llm) - MIT License
- [ChromaDB](https://github.com/chroma-core/chroma) - Apache License 2.0

The Docker Compose configuration is provided under the MIT License.

---

**Note**: This system is designed for local, self-hosted use. All data remains on your hardware and does not leave your environment. No telemetry, analytics, or external API calls are made.
