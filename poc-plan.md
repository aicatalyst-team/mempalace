# PoC Plan: mempalace

## Project Classification
- **Type:** llm-app (local-first AI memory / CLI tool)
- **Key Technologies:** Python 3.9+, ChromaDB (vector store), PyYAML, semantic search, MCP server
- **ODH Relevance:** MemPalace demonstrates local-first AI memory management with a pluggable vector backend (ChromaDB). It is relevant to Open Data Hub as a RAG-adjacent tool that enables semantic search over project and conversation history without requiring external API calls — a pattern useful for developers working with AI coding assistants on OpenShift AI workbenches.

## PoC Objectives
What we want to prove:
1. The MemPalace CLI tool can be containerized and run inside an OpenShift/Kubernetes environment
2. The `init` command successfully creates a palace data structure
3. The `mine` command processes project files and indexes them into the ChromaDB vector store
4. The `search` command performs semantic retrieval and returns relevant results from the mined content
5. The `status` command accurately reports palace state (wings, rooms, drawers)

## Infrastructure Requirements
- **Inference Server:** none — MemPalace uses ChromaDB's built-in embedding model for semantic search; no external inference server needed
- **Vector Database:** in-memory (ChromaDB bundled as a Python dependency)
- **Embedding Model:** ChromaDB's default embedding model (bundled, no external download required)
- **GPU Required:** no — ChromaDB's default embedding model runs on CPU
- **Persistent Storage:** 1Gi PVC — needed for palace data directory (ChromaDB index + metadata)
- **Resource Profile:** medium (1Gi RAM, 500m CPU) — ChromaDB embedding and indexing need moderate memory
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: help-output
- **Description:** Verify the CLI is installed correctly and shows available commands
- **Type:** cli
- **Input:** `mempalace --help`
- **Expected:** Job exits 0, outputs usage info listing subcommands (init, mine, search, status, wake-up, etc.)
- **Timeout:** 30 seconds

### Scenario 2: init-palace
- **Description:** Initialize a new palace from a sample project directory
- **Type:** cli
- **Input:** Create a sample project directory with a README.md, then run `mempalace init /tmp/sample-project`
- **Expected:** Job exits 0, palace config files are created
- **Timeout:** 60 seconds

### Scenario 3: mine-project
- **Description:** Mine sample project files into the palace to create searchable drawers
- **Type:** cli
- **Input:** Create sample .md files, init palace, then run `mempalace mine /tmp/sample-project`
- **Expected:** Job exits 0, outputs mining progress showing files processed and drawers created
- **Timeout:** 120 seconds

### Scenario 4: search-palace
- **Description:** Search the palace for mined content, verifying semantic retrieval with ChromaDB
- **Type:** cli
- **Input:** After mining sample content about GraphQL, run `mempalace search 'GraphQL'`
- **Expected:** Job exits 0, outputs search results referencing the mined GraphQL content
- **Timeout:** 120 seconds

### Scenario 5: status-check
- **Description:** Verify palace status reporting after content has been mined
- **Type:** cli
- **Input:** After mining, run `mempalace status`
- **Expected:** Job exits 0, outputs palace summary with wing/room/drawer counts
- **Timeout:** 120 seconds

## Dockerfile Considerations
This is a **CLI tool**, not a server. The Dockerfile should:

- Use a Python 3.12 base image (slim variant)
- Install the `mempalace` package from the local source using `pip install .`
- ChromaDB and all dependencies will be installed transitively via pip
- **ENTRYPOINT** should be `["mempalace"]` so the container acts as the CLI tool
- **CMD** should default to `["--help"]` so running the container with no args shows usage
- **Do NOT add EXPOSE** — there is no port to expose. The main CLI does not listen on any port
- Note: The project also has a `mempalace-mcp` entry point (MCP server), but for PoC purposes we focus on the CLI. The MCP server uses stdio transport, not HTTP, so it also doesn't need EXPOSE
- The container needs write access to a directory for palace data (ChromaDB stores its index on disk). The default palace location should work with a PVC mount
- No model weights need to be downloaded — ChromaDB's default embedding function (`all-MiniLM-L6-v2` via `onnxruntime`) is installed as a pip dependency and downloads the model on first use. Consider running a warm-up step in the Dockerfile build (`python -c "import chromadb; chromadb.Client().get_or_create_collection('warmup')"`) to pre-cache the embedding model

## Deployment Considerations
- **Deployment model: Job** — MemPalace is a CLI tool. Each test scenario should be deployed as a Kubernetes **Job** that runs a specific command and exits. Do NOT deploy as a Deployment — the process exits immediately after running and would CrashLoopBackOff
- **Do NOT create a Service** — there is no port to expose. The CLI tool does not listen on any network port
- **Testing:** Verify each Job via exit code (`kubectl wait --for=condition=complete`) and inspect output via `kubectl logs`. Each scenario creates sample data, runs mempalace commands, and checks for expected output
- **PVC:** Mount a 1Gi PVC at a consistent path (e.g., `/data/palace`) so the ChromaDB index persists across commands within a single test scenario. Each scenario Job should use its own workspace to avoid interference
- **No LLM API needed:** MemPalace's core functionality (init, mine, search, status) works entirely offline with no API keys. The `--llm-refine` and `--llm-rerank` features are optional and not tested in this PoC
- **Resource requests:** 512Mi-1Gi RAM (ChromaDB + embedding model), 500m CPU. No GPU needed