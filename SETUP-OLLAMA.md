# Archon + Ollama Setup Guide

This guide documents how to configure Archon with local Ollama for embeddings and chat, enabling a fully local RAG system with no external API dependencies.

## Prerequisites

- Docker & Docker Compose
- [Ollama](https://ollama.ai) installed and running locally
- Supabase account (free tier works) or local Supabase

## Quick Start

### 1. Clone and Configure

```bash
git clone -b stable https://github.com/coleam00/archon.git
cd archon
cp .env.example .env
```

Edit `.env` with your Supabase credentials:
```env
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_KEY=your-service-key-here
```

### 2. Database Setup

In your Supabase SQL Editor, run the contents of `migration/complete_setup.sql`.

### 3. Pull Ollama Models

```bash
# Embedding model (required)
ollama pull nomic-embed-text

# Chat model for summaries (optional but recommended)
ollama pull llama3.2
```

### 4. Start Archon

```bash
docker compose up --build -d
```

Wait for all services to be healthy:
```bash
docker compose ps
```

### 5. Configure Ollama Provider

Set the following credentials via API:

```bash
# Set Ollama as embedding provider
curl -X PUT http://localhost:8181/api/credentials/EMBEDDING_PROVIDER \
  -H "Content-Type: application/json" \
  -d '{"value": "ollama"}'

# Set embedding model
curl -X PUT http://localhost:8181/api/credentials/EMBEDDING_MODEL \
  -H "Content-Type: application/json" \
  -d '{"value": "nomic-embed-text"}'

# Set Ollama as LLM provider
curl -X PUT http://localhost:8181/api/credentials/LLM_PROVIDER \
  -H "Content-Type: application/json" \
  -d '{"value": "ollama"}'

# Set chat model for code summaries
curl -X PUT http://localhost:8181/api/credentials/MODEL_CHOICE \
  -H "Content-Type: application/json" \
  -d '{"value": "llama3.2"}'

# Set Ollama base URL (Docker → host)
curl -X PUT http://localhost:8181/api/credentials/LLM_BASE_URL \
  -H "Content-Type: application/json" \
  -d '{"value": "http://host.docker.internal:11434/v1"}'
```

Or configure via the UI at http://localhost:3737/settings.

## Services

| Service | Port | Purpose |
|---------|------|---------|
| archon-ui | 3737 | Web interface |
| archon-server | 8181 | Backend API |
| archon-mcp | 8051 | MCP server for AI assistants |

## Claude Code Integration

### Global Setup (all projects)

```bash
claude mcp add --transport http --scope user archon http://localhost:8051/mcp
```

### Per-Project Setup

```bash
cd /path/to/your/project
claude mcp add --transport http archon http://localhost:8051/mcp
```

### Verify Connection

```bash
claude mcp list
# Should show: archon: http://localhost:8051/mcp (HTTP) - ✓ Connected
```

## MCP Tools Available

Once connected, Claude Code can use these Archon tools:

### RAG (Knowledge Base)
- `rag_get_available_sources()` - List all indexed knowledge sources
- `rag_search_knowledge_base(query, source_id?, match_count?)` - Search documents
- `rag_search_code_examples(query, source_id?, match_count?)` - Find code snippets
- `rag_list_pages_for_source(source_id)` - List pages in a source
- `rag_read_full_page(source_id, page_url)` - Read full page content

### Projects
- `find_projects(query?, project_id?)` - Search or get projects
- `manage_project(action, ...)` - Create/update/delete projects

### Tasks
- `find_tasks(query?, task_id?, filter_by?, filter_value?)` - Search or filter tasks
- `manage_task(action, ...)` - Create/update/delete tasks

## Adding Knowledge

### Upload Documents

```bash
curl -X POST "http://localhost:8181/api/documents/upload" \
  -F "file=@/path/to/document.md" \
  -F "knowledge_type=technical"
```

### Crawl Websites

Via UI: Go to Knowledge Base → Add Source → Enter URL

Via API:
```bash
curl -X POST "http://localhost:8181/api/knowledge-items/crawl" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://docs.example.com", "knowledge_type": "technical"}'
```

## Remote Ollama (Optional)

If running Ollama on a different machine (e.g., a GPU server):

```bash
curl -X PUT http://localhost:8181/api/credentials/LLM_BASE_URL \
  -H "Content-Type: application/json" \
  -d '{"value": "http://your-ollama-host:11434/v1"}'
```

For Tailscale networks:
```bash
# Example: nucbox-zerodski.himalayan-tetra.ts.net:11434
curl -X PUT http://localhost:8181/api/credentials/LLM_BASE_URL \
  -H "Content-Type: application/json" \
  -d '{"value": "http://your-machine.your-tailnet.ts.net:11434/v1"}'
```

## Recommended Ollama Models

| Purpose | Model | Size | Notes |
|---------|-------|------|-------|
| Embeddings | `nomic-embed-text` | 274 MB | Required for RAG |
| Chat/Summaries | `llama3.2` | 2 GB | Fast, good quality |
| Chat/Summaries | `qwen2.5` | 4.7 GB | Better quality |
| Code | `codellama:70b` | 40 GB | For code-heavy tasks |

## Troubleshooting

### Embeddings not working

Check the embedding provider is set correctly:
```bash
curl -s http://localhost:8181/api/credentials | jq '.[] | select(.key | test("EMBED"))'
```

### MCP not connecting

1. Verify MCP server is healthy:
   ```bash
   curl http://localhost:8051/health
   ```

2. Check Claude Code config:
   ```bash
   claude mcp list
   ```

3. Ensure Archon is in user scope for global access:
   ```bash
   cat ~/.claude.json | jq '.mcpServers'
   ```

### Docker can't reach Ollama

The URL `http://host.docker.internal:11434` allows Docker containers to reach the host machine. If this doesn't work:

- **Linux**: Use `http://172.17.0.1:11434` (Docker bridge IP)
- **Or**: Run Ollama in Docker on the same network

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Your Machine                          │
│                                                          │
│  ┌──────────────┐     ┌──────────────────────────────┐  │
│  │   Ollama     │     │      Docker Stack            │  │
│  │   :11434     │◄────│  ┌─────────┐ ┌─────────┐    │  │
│  │              │     │  │ archon- │ │ archon- │    │  │
│  │ nomic-embed  │     │  │ server  │ │   mcp   │    │  │
│  │ llama3.2     │     │  │  :8181  │ │  :8051  │    │  │
│  └──────────────┘     │  └─────────┘ └────┬────┘    │  │
│                       │  ┌─────────┐      │         │  │
│                       │  │ archon- │      │         │  │
│                       │  │   ui    │      │         │  │
│                       │  │  :3737  │      │         │  │
│                       │  └─────────┘      │         │  │
│                       └──────────────────────────────┘  │
│                                           │              │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Claude Code / IDE                    │   │
│  │                      │                            │   │
│  │    MCP Connection ───┘                            │   │
│  │    (rag_search, find_tasks, manage_project...)   │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## License

Archon is open source. See the main repository for license details.
