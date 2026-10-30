# DevOps Runbook & Incident Assistant

A personal learning project: a RAG assistant built with **Haystack 2.x**, the official **IBM Db2 vector-store integration**, and **IBM Granite** models running locally through **Ollama**.

It uses the same RAG recipe as the IBM demo [Haystack_IBM_Db2_Practical_AI_Applications](https://github.com/GeetikaChughIBM/Haystack_IBM_Db2_Practical_AI_Applications), but adds three things that demo lacks:

1. **An ingestion pipeline** – Markdown runbooks with YAML front-matter → split → embed → Db2.
2. **Metadata-filtered retrieval** – answers are scoped by environment (and optionally service) using Db2 metadata filters, so a prod question never gets a dev answer.
3. **Document lifecycle management** – deprecate, restore and purge runbooks with Db2 metadata operations, without re-embedding.

## Architecture

```
 data/runbooks/*.md
        |  (front-matter + body)
        v
  ingest.py:  DocumentSplitter -> OllamaDocumentEmbedder -> DocumentWriter
                (220 words)        (granite-embedding,       (OVERWRITE)
                                    NO prefix)                    |
                                                                  v
                                                      +--------------------+
                                                      |  Db2 12.1.2+       |
                                                      |  table RUNBOOKS    |
                                                      |  VECTOR(768)+meta  |
                                                      +--------------------+
                                                         ^            ^
   ask.py:  OllamaTextEmbedder -> IBMDb2EmbeddingRetriever            |
            (query prefix)        filters: status=active              |
                 |                 AND env [AND service]              |
                 v                                                    |
            PromptBuilder -> OllamaGenerator (granite3.3:8b)          |
                                                                      |
   manage.py:  stats / list / deprecate / restore / purge -------------+
               (metadata API, no re-embedding)
```

## Project layout

```
.
├── .env.example        local practice settings
├── .gitignore
├── README.md
├── ask.py              ask a question (env-scoped RAG)
├── config.py           shared settings + get_document_store()
├── docker-compose.yml  local Db2 12.1.2
├── ingest.py           ingestion pipeline
├── manage.py           lifecycle commands
├── requirements.txt
└── data/runbooks/      RB-001 ... RB-008 sample runbooks
```

## Setup

Prerequisites: Python 3.10+, Docker Desktop, [Ollama](https://ollama.com), VS Code.

### Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

### macOS / Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

### Then (both platforms)

1. In VS Code: `Ctrl+Shift+P` → **Python: Select Interpreter** → choose `.venv`.
2. Start Db2: `docker compose up -d`. The first start takes several minutes. Run `docker compose logs -f db2` and wait for **"Setup has completed"**.
3. Pull the models:
   ```
   ollama pull granite-embedding:278m
   ollama pull granite3.3:8b
   ```
4. Load the runbooks: `python ingest.py --reset`

## Demo

**Same question, different environment, different answer** (RB-001 vs RB-002):

```
python ask.py "Deployment fails on S3 sync step with AccessDenied" --env prod
python ask.py "Deployment fails on S3 sync step with AccessDenied" --env dev
```

Prod answers with the careful path (change request, IAM policy simulation, never widen the bucket policy). Dev answers with retry, refresh credentials, and widening the dev bucket policy as acceptable.

**Narrow by service and inspect what was retrieved:**

```
python ask.py "Service is unhealthy after deploy" --env prod --service payments-api --show-docs
```

**A runbook that exists only in another environment** (RB-004 is uat-only):

```
python ask.py "Pod is in CrashLoopBackOff" --env prod --service payments-api
```

Expected: `No matching runbook for this environment. Escalate to the platform on-call.`

The deprecated RB-008 (legacy FTP deploy) is never returned for any query, because retrieval always filters on `status == active`.

## Lifecycle commands

```
python manage.py stats                 # counts, services, environments, active chunks per env
python manage.py list --env prod       # one line per active runbook
python manage.py list --all            # include deprecated
python manage.py deprecate RB-001      # hide from retrieval (no re-embedding)
python manage.py restore RB-001
python manage.py purge                 # permanently delete deprecated chunks
```

## Viewing the table in VS Code

Install the **IBM Db2** extension, add a connection (host `localhost`, port `50000`, database `RUNBOOKS`, user `db2inst1`). Browse schema **DB2INST1**, table **RUNBOOKS**. The `EMBEDDING` column is `VECTOR(768, FLOAT32)`; metadata is stored alongside it.

## Adding a new runbook

1. Create `data/runbooks/RB-009-your-topic.md`.
2. Start with a front-matter block between `---` lines containing `runbook_id`, `title`, `service`, `environment`, `severity`, `status`, `last_verified`.
3. Add `## Symptom`, `## Steps` (numbered), `## Escalation`.
4. Run `python ingest.py` (no `--reset` needed; OVERWRITE updates existing chunks).

## Which script uses which Db2 integration feature

| Script | Db2 integration feature |
|---|---|
| `config.py` | `IBMDb2DocumentStore` (table, `embedding_dim=768`, COSINE distance, `Secret` credentials) |
| `ingest.py` | `DocumentWriter` with `DuplicatePolicy.OVERWRITE`; `recreate_table` |
| `ask.py` | `IBMDb2EmbeddingRetriever`, metadata filters, `FilterPolicy.MERGE` |
| `manage.py` | `count_documents`, `count_documents_by_filter`, `get_metadata_field_unique_values`, `filter_documents`, `update_by_filter`, `delete_by_filter` |

## Troubleshooting

- **Connection refused / SQL1032N**: Db2 is still starting. Check `docker compose logs db2` for "Setup has completed".
- **Embedding dimension mismatch**: the table was created with a different dimension or you changed `EMBED_MODEL`. Use `granite-embedding:278m` (768) and re-run `python ingest.py --reset`.
- **Poor or odd retrieval**: the query prefix must be only on the query embedder (`ask.py`), never on the document embedder (`ingest.py`).
- **Ollama errors / connection refused**: make sure Ollama is running (`ollama list`) and `OLLAMA_URL` in `.env` is correct; models must be pulled.

## Credits

- Official Haystack Db2 integration documentation (Haystack integrations, IBM Db2).
- Inspired by GeetikaChughIBM's [Haystack_IBM_Db2_Practical_AI_Applications](https://github.com/GeetikaChughIBM/Haystack_IBM_Db2_Practical_AI_Applications).
