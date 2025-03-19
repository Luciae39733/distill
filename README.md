# Distill

[![CI](https://github.com/Siddhant-K-code/distill/actions/workflows/ci.yml/badge.svg)](https://github.com/Siddhant-K-code/distill/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/Siddhant-K-code/distill)](https://github.com/Siddhant-K-code/distill/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Go Report Card](https://goreportcard.com/badge/github.com/Siddhant-K-code/distill)](https://goreportcard.com/report/github.com/Siddhant-K-code/distill)


[![Build with Ona](https://ona.com/build-with-ona.svg)](https://app.ona.com/#https://github.com/siddhant-k-code/distill)

**Your agent remembers what matters.**

Distill gives LLM agents persistent, deduplicated memory that survives across sessions. It prevents repeated re-learning, surfaces conflicting information before it causes mistakes, and compresses aging context so the signal stays high.

> Other tools compress what goes into your agent. Distill controls what your agent *remembers* — across sessions, without conflicts, ranked by what matters now.

**[Learn more →](https://distill.siddhantkhare.com)**

> 📖 Distill implements the 4-layer context engineering stack described in **[The Agentic Engineering Guide](https://agents.siddhantkhare.com/05-context-engineering-stack/)**, a free open book on AI agent infrastructure.

```
RAG / tools / memory / docs
          ↓
        Distill
  (remember · dedupe · compress · cache)
          ↓
         LLM
```

## The Problem

Agents forget. Every new session starts from zero — the same constraints, preferences, and facts have to be re-established. When context does persist, 30-40% of it is semantically redundant, and contradictory information sits side by side with no signal about which version is current.

This causes non-deterministic outputs, confused reasoning, and failures that only show up at scale. Better prompts don't fix it. The agent needs memory it can trust.

## How It Works

No LLM calls. Fully deterministic. ~12ms overhead.

| Stage | What it does |
|-------|-------------|
| **Remember** | Persistent memory across sessions with write-time dedup, expiry, and sensitivity tagging |
| **Deduplicate** | Cluster semantically similar chunks, keep one representative per cluster |
| **Compress** | Extractive compression to remove noise and preserve signal |
| **Summarize** | Progressively condense conversation history as turns age |
| **Cache** | Annotate stable prefixes with `cache_control`, track TTL per prefix |

All stages chain together via `POST /v1/pipeline` or `distill pipeline` CLI. Memory is available via `--memory` flag.

### Dedup pipeline

```
Query → Over-fetch (50) → Cluster → Select → MMR Re-rank (8) → LLM
```

1. **Over-fetch** - retrieve 3-5x more chunks than needed
2. **Cluster** - group semantically similar chunks (agglomerative clustering)
3. **Select** - pick the best representative from each cluster
4. **MMR Re-rank** - balance relevance and diversity

**Result:** Deterministic, diverse context. No LLM calls. Fully auditable.

## Installation

### Binary (Recommended)

After a release is published, replace `vX.Y.Z-alpha.N` below with its tag:

```bash
set -euo pipefail

TAG=vX.Y.Z-alpha.N
VERSION=${TAG#v}
case "$(uname -s)/$(uname -m)" in
  Darwin/arm64) TARGET=darwin_arm64 ;;
  Darwin/x86_64) TARGET=darwin_amd64 ;;
  Linux/x86_64) TARGET=linux_amd64 ;;
  Linux/aarch64|Linux/arm64) TARGET=linux_arm64 ;;
  *) echo "Unsupported platform: $(uname -s)/$(uname -m)" >&2; exit 1 ;;
esac

BASE_URL="https://github.com/Siddhant-K-code/distill/releases/download/${TAG}"
ARCHIVE="distill_${VERSION}_${TARGET}.tar.gz"
curl -fLO "${BASE_URL}/${ARCHIVE}"
curl -fLO "${BASE_URL}/checksums.txt"

CHECKSUM=$(grep "  ${ARCHIVE}$" checksums.txt)
if command -v sha256sum >/dev/null 2>&1; then
  printf '%s\n' "$CHECKSUM" | sha256sum --check -
else
  printf '%s\n' "$CHECKSUM" | shasum -a 256 --check -
fi

tar -xzf "$ARCHIVE"
sudo mkdir -p /usr/local/bin
sudo install -m 0755 distill /usr/local/bin/distill
distill handoff --help
```

Or download directly from the [releases page](https://github.com/Siddhant-K-code/distill/releases).

### Go Install

```bash
go install github.com/Siddhant-K-code/distill@latest
```

### Docker

```bash
docker pull ghcr.io/siddhant-k-code/distill:latest
docker run -p 8080:8080 -e OPENAI_API_KEY=your-key ghcr.io/siddhant-k-code/distill
```

### Build from Source

```bash
git clone https://github.com/Siddhant-K-code/distill.git
cd distill
go build -o distill .
```

## Development

```bash
make build        # compile ./distill
make test         # go test ./...
make check        # fmt + vet + test
make test-cover   # test with coverage report
make bench        # run benchmarks
make distill-lock-demo # deterministic lock/build/verify demo
make handoff-v0-demo # offline review-only decision handoff demo
make lint         # golangci-lint (requires golangci-lint in PATH)
make docker-build # build Docker image
make help         # list all targets
```

## Distill Lock v0: context is a build artifact

Runtime prompt assembly hides source drift. Distill Lock freezes a reviewed
source inventory and configuration, then builds a portable context artifact
without a model, provider, or network call:

```bash
distill lock path/to/config.json --output path/to/context.lock.json
distill build path/to/context.lock.json --output path/to/context-output
distill verify path/to/context-output
distill verify path/to/context-output --expected-lock-sha256 <trusted-digest>

# Synthetic fixture: build, offline verify, repeatability, and mutation proof.
make distill-lock-demo
```

A configuration explicitly lists selected and excluded UTF-8 text/code files.
`lock` records original and normalized SHA-256 identities, NFC/LF-normalized
chunks, exact-duplicate decisions, the deterministic token budget, and every
inclusion/exclusion reason. `build` refuses any missing, changed, newly
unexpected, unsafe, or unsupported input; it never relocks automatically.
`verify` checks the complete allowlisted file set, canonical JSON, schemas,
lengths, hashes, relationships, and exact bundle regeneration offline.
Lock output cannot overwrite its configuration. A post-publication directory
sync failure is reported explicitly as `published but durability unconfirmed`;
verify the named output instead of retrying blindly.
Without the optional out-of-band lock digest it proves internal consistency,
not authenticity; use `--expected-lock-sha256` when the artifact directory may
be attacker-replaceable.

The fresh output directory contains:

```text
context.bundle.md
context.lock.json
context.manifest.json
SHA256SUMS
```

**Guarantee:** the same locked sources, configuration, and supported
runtime/tool identities produce byte-identical bundle, manifest, lockfile, and
digests on supported macOS/Linux systems with Go 1.24 through Go 1.26. Other
Go runtime versions fail closed. Original CRLF/CR hashes remain distinct while
normalized LF content can deduplicate exactly. Builds use native atomic
no-replace publication into a nonexistent directory, so failures cannot
replace a previous valid artifact.

**Non-guarantees and non-goals:** v0 does not provide semantic or
near-duplicate matching, a vector database, agent framework, model adapter,
decision replay, dashboard, Jev integration, or deterministic model output.
The existing semantic Distill pipeline remains available and unchanged.

See [the frozen Distill Lock v0 specification](docs/distill-lock-v0.md) for the
canonical JSON rules, algorithms, failure behavior, and a complete
configuration example.

## Handoff v0 alpha: decisions become review artifacts

Handoff freezes one exported Markdown conversation and a local Markdown docs
tree, then verifies a strict proposal produced by an external coding agent:

```bash
distill handoff prepare --conversation export.md --docs ./docs --out ./request
distill handoff verify --request ./request/handoff.request.json \
  --expected-request-sha256 <digest-printed-by-prepare> \
  --proposal ./proposal.json --out ./review
```

Distill makes no model or network call, never changes source docs, never applies
a patch, and accepts only `require_review` routes. Request and review bundles
use private `0700` directories and `0600` files, and verification requires the
trusted request SHA-256 printed by `prepare`. The deterministic guarantee covers
normalization, identities, evidence checks, patch validation, receipts, review
rendering, and checksum-bound patch files—not probabilistic extraction or patch
drafting. This is an experimental Markdown/local-repository alpha, not general
availability or evidence of adoption. Run `make handoff-v0-demo` or see the
[tester-ready retrospective pilot](docs/handoff-pilot-kit.md) and
[bounded protocol](docs/handoff-v0.md).

For Distill Lock, the separate research-only
[offline excluded-pilot harness](research/context-is-a-build-artifact/pilot/README.md)
generates synthetic, zero-provider-call protocol fixtures and is not wired into
the public `distill` CLI.

The research-only
[Handoff semantic-draft compiler experiment](research/handoff-semantic-draft-compiler/README.md)
tests semantic model output plus deterministic proposal construction without
changing Handoff product behavior.

## Quick Start

### 1. Standalone API (No Vector DB Required)

Start the API server and send chunks directly:

```bash
export OPENAI_API_KEY="your-key"  # For embeddings
distill api --port 8080
```

Deduplicate chunks:

```bash
curl -X POST http://localhost:8080/v1/dedupe \
  -H "Content-Type: application/json" \
  -d '{
    "chunks": [
      {"id": "1", "text": "React is a JavaScript library for building UIs."},
      {"id": "2", "text": "React.js is a JS library for building user interfaces."},
      {"id": "3", "text": "Vue is a progressive framework for building UIs."}
    ]
  }'
```

Response:

```json
{
  "chunks": [
    {"id": "1", "text": "React is a JavaScript library for building UIs.", "cluster_id": 0},
    {"id": "3", "text": "Vue is a progressive framework for building UIs.", "cluster_id": 1}
  ],
  "stats": {
    "input_count": 3,
    "output_count": 2,
    "reduction_pct": 33,
    "latency_ms": 12
  }
}
```

**With pre-computed embeddings (no OpenAI key needed):**

```bash
curl -X POST http://localhost:8080/v1/dedupe \
  -H "Content-Type: application/json" \
  -d '{
    "chunks": [
      {"id": "1", "text": "React is...", "embedding": [0.1, 0.2, ...]},
      {"id": "2", "text": "React.js is...", "embedding": [0.11, 0.21, ...]},
      {"id": "3", "text": "Vue is...", "embedding": [0.9, 0.8, ...]}
    ]
  }'
```

### 2. With Vector Database

Connect to Pinecone or Qdrant for retrieval + deduplication:

```bash
export PINECONE_API_KEY="your-key"
export OPENAI_API_KEY="your-key"

distill serve --index my-index --port 8080
```

Query with automatic deduplication:

```bash
curl -X POST http://localhost:8080/v1/retrieve \
  -H "Content-Type: application/json" \
  -d '{"query": "how do I reset my password?"}'
```

### 3. MCP Integration (AI Assistants)

Works with Claude, Cursor, Amp, and other MCP-compatible assistants:

```bash
# Dedup only
distill mcp

# With memory and sessions
distill mcp --memory --session
```

Add to Claude Desktop (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "distill": {
      "command": "/path/to/distill",
      "args": ["mcp", "--memory", "--session"],
      "env": {
        "OPENAI_API_KEY": "your-key"
      }
    }
  }
}
```

See [mcp/README.md](mcp/README.md) for more configuration options.

## Context Memory

Persistent memory that accumulates knowledge across agent sessions. Memories are deduplicated on write, ranked by relevance + recency on recall, and compressed over time through hierarchical decay.

Enable with the `--memory` flag on `api` or `mcp` commands.

### CLI

```bash
# Store a memory
distill memory store --text "Auth uses JWT with RS256 signing" --tags auth --source docs

# Recall relevant memories
distill memory recall --query "How does authentication work?" --max-results 5

# Remove outdated memories
distill memory forget --tags deprecated

# View statistics
distill memory stats
```

### API

```bash
# Start API with memory enabled
distill api --port 8080 --memory

# Store (with auto sensitivity classification)
curl -X POST http://localhost:8080/v1/memory/store \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "session-1",
    "entries": [{"text": "Auth uses JWT with RS256", "tags": ["auth"], "source": "docs", "auto_classify": true}]
  }'

# Recall (with task-relevance boosting)
curl -X POST http://localhost:8080/v1/memory/recall \
  -H "Content-Type: application/json" \
  -d '{"query": "How does auth work?", "max_results": 5, "boost_tags": ["auth"], "min_relevance": 0.3}'

# Expire a memory (soft delete)
curl -X POST http://localhost:8080/v1/memory/expire \
  -H "Content-Type: application/json" \
  -d '{"id": "memory-id"}'

# Supersede with a newer version
curl -X POST http://localhost:8080/v1/memory/supersede \
  -H "Content-Type: application/json" \
  -d '{"id": "old-memory-id", "new_id": "new-memory-id"}'
```

### MCP

Memory tools are available in Claude Desktop, Cursor, and other MCP clients when `--memory` is enabled:

```bash
distill mcp --memory
```

Tools exposed: `store_memory`, `recall_memory`, `forget_memory`, `memory_expire`, `memory_supersede`, `memory_stats`.

### How Decay Works

Memories compress over time based on access patterns:

```
Full text → Summary (~20%) → Keywords (~5%) → Evicted
  (24h)        (7 days)         (30 days)
```

Accessing a memory resets its decay clock. Configure via `distill.yaml`:

```yaml
memory:
  db_path: distill-memory.db
  dedup_threshold: 0.15
  conflict_threshold: 0.35
```

## Session Management

Token-budgeted context windows for long-running agent sessions. Push context incrementally - Distill deduplicates, compresses aging entries, and evicts when the budget is exceeded.

Enable with the `--session` flag on `api` or `mcp` commands.

### CLI

```bash
# Create a session with 128K token budget
distill session create --session-id task-42 --max-tokens 128000

# Push context as the agent works
distill session push --session-id task-42 --role user --content "Fix the JWT validation bug"
distill session push --session-id task-42 --role tool --content "$(cat auth/jwt.go)" --source file_read --importance 0.8

# Read the current context window
distill session context --session-id task-42

# Clean up when done
distill session delete --session-id task-42
```

### API

```bash
# Start API with sessions enabled
distill api --port 8080 --session

# Create session
curl -X POST http://localhost:8080/v1/session/create \
  -H "Content-Type: application/json" \
  -d '{"session_id": "task-42", "max_tokens": 128000}'

# Push entries
curl -X POST http://localhost:8080/v1/session/push \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "task-42",
    "entries": [
      {"role": "tool", "content": "file contents...", "source": "file_read", "importance": 0.8}
    ]
  }'

# Read context window
curl -X POST http://localhost:8080/v1/session/context \
  -H "Content-Type: application/json" \
  -d '{"session_id": "task-42"}'
```

### MCP

Session tools are available when `--session` is enabled:

```bash
distill mcp --session
```

Tools exposed: `create_session`, `push_session`, `session_context`, `delete_session`.

### How Budget Enforcement Works

When a push exceeds the token budget:

1. **Compress** oldest entries (outside the `preserve_recent` window) through levels:
   - Full text → Summary (~20%) → Single sentence (~5%) → Keywords (~1%)
2. **Evict** entries that are already at keyword level
3. Lowest-importance entries are compressed/evicted first

The `preserve_recent` setting (default: 10) keeps the most recent entries at full fidelity.

## CLI Commands

```bash
distill api        # Start standalone API server
distill serve      # Start server with vector DB connection
distill pipeline   # Run full optimisation pipeline (dedup → compress → summarize)
distill mcp        # Start MCP server for AI assistants
distill memory     # Store, recall, and manage persistent context memories
distill session    # Manage token-budgeted context windows for agent sessions
distill analyze    # Analyze a file for duplicates
distill sync       # Upload vectors to Pinecone with dedup
distill query      # Test a query from command line
distill config     # Manage configuration files
distill completion # Generate shell completion scripts (bash/zsh/fish/powershell)
```

### Pipeline command

```bash
# Run full pipeline on a JSON chunk array
echo '[{"id":"1","text":"..."}]' | distill pipeline

# From file, with stats
distill pipeline --input chunks.json --output optimised.json --stats

# Tune individual stages
distill pipeline --dedup-threshold 0.2 --compress-ratio 0.4 --summarize --summarize-max-tokens 2000

# Disable a stage
distill pipeline --no-compress
```

### Shell completions

```bash
# Bash (one-time)
distill completion bash > /etc/bash_completion.d/distill

# Zsh
distill completion zsh > "${fpath[1]}/_distill"

# Fish
distill completion fish > ~/.config/fish/completions/distill.fish

# PowerShell
distill completion powershell | Out-String | Invoke-Expression
```

## API Endpoints

Interactive docs available at `/docs` (Swagger UI). Full OpenAPI 3.1 spec at `/openapi.yaml`.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/dedupe` | Deduplicate chunks |
| POST | `/v1/dedupe/stream` | SSE streaming dedup with per-stage progress |
| POST | `/v1/pipeline` | Full optimisation pipeline (dedup → compress → summarize) |
| POST | `/v1/batch` | Submit async batch job |
| GET | `/v1/batch/{id}` | Poll batch job status and progress |
| GET | `/v1/batch/{id}/results` | Retrieve completed batch results |
| POST | `/v1/retrieve` | Query vector DB with dedup (requires backend) |
| POST | `/v1/memory/store` | Store memories with write-time dedup and sensitivity tagging (requires `--memory`) |
| POST | `/v1/memory/recall` | Recall memories by relevance + recency (requires `--memory`) |
| POST | `/v1/memory/forget` | Remove memories by ID, tag, or age (requires `--memory`) |
| POST | `/v1/memory/expire` | Mark memories as expired without deleting (requires `--memory`) |
| POST | `/v1/memory/supersede` | Replace a memory with a newer version (requires `--memory`) |
| GET | `/v1/memory/stats` | Memory store statistics (requires `--memory`) |
| POST | `/v1/session/create` | Create a session with token budget (requires `--session`) |
| POST | `/v1/session/push` | Push entries with dedup + budget enforcement (requires `--session`) |
| POST | `/v1/session/context` | Read current context window (requires `--session`) |
| POST | `/v1/session/delete` | Delete a session (requires `--session`) |
| GET | `/v1/session/get` | Get session metadata (requires `--session`) |
| GET | `/health` | Health check |
| GET | `/metrics` | Prometheus metrics |

### Pipeline API

```json
POST /v1/pipeline
{
  "chunks": [{"id": "1", "text": "..."}],
  "options": {
    "dedup":     {"enabled": true, "threshold": 0.15},
    "compress":  {"enabled": true, "target_reduction": 0.5},
    "summarize": {"enabled": false, "max_tokens": 4000}
  }
}
```

Response includes per-stage token counts, reduction ratios, and latency.

### Batch API

```bash
# Submit
curl -X POST /v1/batch -d '{"chunks":[...],"options":{...}}'
# → {"job_id":"batch_1234","status":"queued"}

# Poll
curl /v1/batch/batch_1234
# → {"status":"processing","progress":0.45}

# Results (when completed)
curl /v1/batch/batch_1234/results
# → {"chunks":[...],"stats":{...}}
```

## Logging

Distill uses structured `log/slog` logging. Default output is JSON to stderr.

```go
import "github.com/Siddhant-K-code/distill/pkg/logging"

// JSON logger (production default)