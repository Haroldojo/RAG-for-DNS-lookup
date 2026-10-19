# Agentic DNS & Hosting Support Copilot

A production-minded portfolio project for technical-support teams handling DNS, SSL,
domains, hosting, email-DNS, and website-access issues.

The copilot will combine support-document retrieval with safe, read-only diagnostics.
It will provide evidence-linked guidance and escalate cases when the evidence is
insufficient. It does **not** change DNS, hosting, or domain settings.

## Current milestone

**Phase 8 — SSL, HTTP, and evidence fusion:** the copilot now combines read-only DNS, TLS, HTTP,
and retrieved support evidence. It never changes infrastructure and does not scan private networks.

## Retrieval flow

```text
support document → clean/normalize → overlapping chunks → local embeddings → Qdrant
user query → same embedding model → cosine search → top_k document chunks
```

## DNS diagnostics

`POST /api/v1/tools/dns-check` queries public DNS for `A`, `AAAA`, `CNAME`, `MX`, `NS`, and
`TXT` records. It validates the supplied hostname first and returns a result for every record
type, even where no record exists. The tool has no DNS-write capability.

| Status | Meaning |
| --- | --- |
| `ok` | At least one record was returned and no resolver failure occurred. |
| `no_records` | Domain resolved but none of the requested record types exist. |
| `nxdomain` | DNS says the domain does not exist. |
| `partial_failure` | Some record queries worked, but another timed out or hit a resolver failure. |
| `resolver_failure` | The resolver could not provide usable results. |

Per-record statuses distinguish `no_record`, `nxdomain`, `timeout`, and `resolver_failure`.
A timeout means no timely answer; it does not prove the record is absent. A missing record means
the domain is known but that specific record type has no answer.

`A` maps to IPv4, `AAAA` maps to IPv6, `CNAME` aliases one hostname to another, `MX` identifies
mail servers, `NS` identifies authoritative nameservers, and `TXT` stores text such as domain
verification or SPF policy. A later LLM may explain the returned values but must never invent DNS
records, domain ownership, successful propagation, or a resolver outcome.

## Why these defaults?

- **Embedding model — `BAAI/bge-small-en-v1.5`:** a compact semantic-retrieval model that runs
  locally via FastEmbed/ONNX after its one-time download. It is much smaller and cheaper to run
  than a large language model, while being more semantic than keyword search.
- **800-character chunks with 120-character overlap:** support articles are short and procedural.
  This size usually keeps one troubleshooting step together; the overlap retains context when a
  useful sentence sits at a chunk boundary. These are starting values to evaluate, not claims of
  optimal quality.
- **Cosine similarity:** Qdrant compares the angle between normalized query and document vectors.
  Scores closer to 1 mean the vectors point in a more similar direction; the score is useful for
  ranking, not a guarantee that the text is correct.
- **`top_k`:** the number of highest-scoring chunks returned. `top_k=5` favors recall; a smaller
  value reduces noise. Retrieval quality determines the appropriate value.

Qdrant stores each chunk vector with a payload containing its text and metadata. At search time,
it embeds the query, finds the nearest vectors under cosine distance, and returns their payloads.

Retrieval can fail when a document is missing or stale, chunk boundaries separate the needed
facts, a query uses unfamiliar wording, metadata is wrong, the embedding model lacks domain
coverage, or `top_k` is too low/high. This phase includes ten evaluation questions so those
failures can be measured rather than guessed.

## Run locally

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\\Scripts\\activate
pip install -e ".[dev]"
cp .env.example .env
# Start Qdrant first (local development):
docker run --rm -p 6333:6333 qdrant/qdrant
# In a second terminal:
python -m app.rag.bootstrap
python -m uvicorn app.main:app --reload
```

Open `http://127.0.0.1:8000/docs`. Run tests with `python -m pytest -q`.

### Exact Phase 2 commands

```bash
# Ingest the five sample documents into the support_docs collection
python -m app.rag.bootstrap

# Start the API
python -m uvicorn app.main:app --reload

# Run the automated test suite (uses in-memory Qdrant; Docker is not required for tests)
python -m pytest -q

# Run the offline DNS-tool evaluation (uses mocked resolver outcomes)
python -m app.evaluation.run_dns_tool_evaluation

# Run the deterministic routing evaluation (uses fake retrieval/DNS tools)
python -m app.evaluation.run_routing_evaluation

# Run the deterministic grounded-generation baseline (uses a fake LLM provider)
python -m app.evaluation.run_grounding_evaluation
```

### Phase 6 database commands

Configure `DATABASE_URL` for PostgreSQL (the `.env.example` URL is a development default), then
apply the schema through Alembic. Production schema changes use migrations, never application
startup table creation.

```bash
python -m alembic upgrade head
python -m alembic downgrade base  # development only: removes persistence tables
python -m uvicorn app.main:app --reload
python -m pytest -q
```

### Phase 7 Redis commands

```bash
# Start a local Redis instance for cache/rate-limit behavior
docker run --rm -p 6379:6379 redis:7-alpine

# Configure REDIS_URL=redis://localhost:6379/0 in .env, then run the API
python -m uvicorn app.main:app --reload

# Deterministic local cache-aside benchmark (does not require a Redis server)
python -m app.evaluation.run_cache_benchmark
```

## Redis cache and rate limiting

Redis is used only for disposable, repeatable work. DNS uses cache-aside under
`dns:v1:{domain}:all`, which stores the six-record snapshot together for `DNS_CACHE_TTL_SECONDS`
(default 60 seconds). A request first checks Redis: a **hit** returns the snapshot; a **miss** calls
the resolver, then stores the structured result. `cache_status`, cache latency, and actual DNS-tool
latency are returned with the DNS response. Cached values are never proof of current propagation;
TTL bounds, but does not eliminate, stale-data risk.

Deterministic `NXDOMAIN` and no-record observations are briefly cached to reduce repeat load.
Resolver failures and partial failures are not cached, because caching an outage would prolong an
uncertain state. Malformed cache values, Redis errors, timeouts, and connection resets are logged
and bypassed; the DNS resolver/Qdrant remains authoritative. Cache reads and writes therefore
**fail open**.

Retrieval caching uses a hashed, versioned key that includes normalized query, collection name,
embedding/chunk settings, `top_k`, and `RAG_CACHE_VERSION`. Changing the version invalidates prior
entries without scanning Redis. Final LLM answers are deliberately not cached: their wording may
depend on current DNS evidence, conversation context, escalation policy, or source updates.

`POST /api/v1/diagnose` uses a fixed-window Redis counter: default **20 requests per 60 seconds per
client IP**. Exceeded requests receive `429`, a `Retry-After` header, and
`{"code":"rate_limit_exceeded"}`. The limiter fails open when Redis is unavailable, deliberately:
availability of support diagnosis wins over an optional abuse-control layer, and the decision is
logged. A future authenticated product should use a stable authenticated principal rather than IP.

TTL is Redis's automatic expiry time. Invalidation is handled by expiry plus versioned retrieval
keys; DNS entries are intentionally short-lived. A cache stampede is many simultaneous misses for
one key; this baseline does not coordinate locks, so a later scale phase can add request coalescing
or single-flight locking if measurements justify it. No credentials, API keys, or confidential
user payloads are cached or logged.

### Search API

```bash
curl -X POST http://127.0.0.1:8000/api/v1/search \
  -H "Content-Type: application/json" \
  -d '{"query":"Where do I change nameservers after moving hosting?","top_k":3}'
```

### DNS API

```bash
curl -X POST http://127.0.0.1:8000/api/v1/tools/dns-check \
  -H "Content-Type: application/json" \
  -d '{"domain":"example.com"}'
```

Response shape:

```json
{
  "domain": "example.com",
  "status": "ok",
  "records": {
    "A": {"record_type": "A", "status": "ok", "records": ["93.184.216.34"], "error": null},
    "AAAA": {"record_type": "AAAA", "status": "no_record", "records": [], "error": null}
  },
  "errors": [],
  "lookup_duration_ms": 18.42,
  "tool_execution_id": "a-unique-uuid"
}
```

The real response always includes all six record types; values and timing vary with the live DNS
answer. Structured logs carry the same `tool_execution_id` for each tool run.

### Diagnosis API

`POST /api/v1/diagnose` returns graph evidence, tool decisions, a grounded answer, and durable IDs.
Omit `conversation_id` to start a new conversation, or pass a returned ID to continue it.

```text
START → validate_input → classify_issue
  ├─ documentation_question → retrieve_docs → build_context → decide_escalation → END
  ├─ dns_issue → run_dns_check → retrieve_docs → build_context → decide_escalation → END
  └─ unknown → retrieve_docs → build_context → decide_escalation → END
```

```bash
curl -X POST http://127.0.0.1:8000/api/v1/diagnose \
  -H "Content-Type: application/json" \
  -d '{"query":"My domain is not resolving after I changed nameservers","domain":"example.com"}'
```

```json
{
  "conversation_id": "5cda0c12-...",
  "diagnosis_run_id": "5a5a6d9c-...",
  "intent": "dns_issue",
  "route_taken": "dns_and_rag",
  "tools_called": ["dns_check", "search_support_docs"],
  "retrieved_chunks": [],
  "dns_result": {"domain": "example.com", "status": "ok"},
  "answer_context": {"note": "Evidence only; no LLM answer was generated."},
  "confidence": 0.85,
  "needs_escalation": false,
  "errors": []
}
```

The typed state includes query, domain, intent, retrieved chunks, DNS result, context, confidence,
escalation, errors, route, and tools called. Nodes update state; ordinary edges establish sequence;
the conditional edge after classification chooses the tool path. Routing is deterministic and
keyword-based in this phase, not delegated to an LLM.

### Conversation history API

```bash
curl -X POST http://127.0.0.1:8000/api/v1/conversations
curl http://127.0.0.1:8000/api/v1/conversations/CONVERSATION_ID
curl http://127.0.0.1:8000/api/v1/conversations/CONVERSATION_ID/messages
curl http://127.0.0.1:8000/api/v1/diagnosis/DIAGNOSIS_RUN_ID
```

Persistence is deliberately fail-closed for durable IDs. If the graph completes but PostgreSQL is
unavailable or the transaction cannot commit, `/diagnose` returns `503` with
`{"code":"persistence_failed","request_id":"..."}`; it does not claim that a conversation or
run was saved. An unknown `conversation_id` returns `404` with `conversation_not_found` before
the diagnostic graph runs.

## Durable-history design

PostgreSQL is the source of truth for relational history: a conversation owns messages and
diagnosis runs; a run owns compact tool runs, citations, and optional escalations. Foreign keys
prevent orphan audit records, and indexes support the conversation timeline, request lookup, and
run-detail lookups. This normalization avoids repeating conversation fields on every tool or
citation row.

One database transaction writes the user message, diagnosis run, assistant answer, tool summaries,
citations, and escalation. `commit` makes the entire set durable; any database exception triggers
`rollback`, so a half-written run cannot appear in later support review. The LangGraph workflow is
kept independent: it returns a structured state first, then the API/service layer persists it.

Qdrant remains the vector database for semantic chunk lookup; PostgreSQL is the relational
database for durable, queryable relationships and audit history. Redis is intentionally not used:
it is useful later for short-lived caching, not a replacement for durable conversation records.

Persisted data is limited to user/assistant messages, routing/run attributes, compact tool
summaries (record-type names and counts rather than raw DNS payloads), citations, and escalation
status. API keys, passwords, database credentials, and raw provider credentials are never stored
or logged. Database-operation logs contain request/conversation/run IDs, operation, duration, and
success/failure only.

## Grounded answer generation

After retrieval/DNS tools finish, the `generate_answer` graph node sends the query, route, retrieved
chunks, DNS JSON, and tool errors to the provider abstraction. It cannot call tools or create
evidence. The dedicated prompt requires observed facts, recommendations, uncertainty, and concise
troubleshooting. It forbids invented DNS records, propagation, ownership, SSL/server status,
configuration changes, and citations.

`GroqProvider` is the first implementation of the `LLMProvider` interface. Configuration is loaded
from `GROQ_API_KEY`, `GROQ_MODEL`, `GROQ_TIMEOUT_SECONDS`, `GROQ_TEMPERATURE`, and
`GROQ_MAX_TOKENS`. The default temperature is `0.1` for stable technical responses. A missing key,
timeout, rate limit, malformed/empty output, or unavailable provider returns a safe fallback while
preserving evidence and setting escalation.
If the DNS tool itself reports a resolver failure, generation is skipped and the fallback states
that no usable DNS records were observed.

Citations are never accepted from the model as canonical data. Application code derives them from
retrieved `document_id`, `source`, `topic`, and `version`, then replaces the model's Sources section
with that controlled list. Generation logs include request ID, model, latency, input-context size,
available token usage, status, and error type—never the API key.

Response shape:

```json
{
  "query": "Where do I change nameservers after moving hosting?",
  "retrieved_chunks": [
    {
      "text": "When a domain does not resolve after a migration...",
      "similarity_score": 0.7969588456513065,
      "document_id": "nameserver-troubleshooting-v1",
      "source": "data/support_docs/nameserver-troubleshooting.md",
      "category": "dns",
      "topic": "nameserver delegation",
      "version": "1.0",
      "chunk_index": 0
    }
  ]
}
```

## Architecture decisions

- **FastAPI + Pydantic:** typed API contracts and automatic API documentation.
- **Qdrant + cosine vectors:** dedicated vector storage and similarity search, with source metadata
  returned alongside every result.
- **dnspython:** a mature DNS protocol client wrapped behind a narrow read-only service interface,
  which keeps unit tests independent of the live internet.
- **LangGraph:** explicit graph state and conditional edges keep routing observable and each tool
  failure isolated, while allowing new tools to be added later without rewriting control flow.
- **Provider abstraction:** Groq is isolated behind a small interface, allowing a model/provider
  replacement without changing graph nodes or API routes.
- **Explicit settings:** configuration comes from environment variables; no secrets in code.
- **Modular boundaries:** API, agents, tools, RAG, cache, database, evaluation and
  observability remain separate as capabilities are introduced.
- **Read-only diagnostics first:** safer support guidance and a clear audit boundary.
- **SQLAlchemy + Alembic:** SQLAlchemy 2.x models encode relationships; Alembic owns explicit,
  reviewable production migrations. The repository/service layer owns transactions, keeping graph
  nodes independent of database I/O.
- **Redis cache-aside:** Redis accelerates read-heavy DNS and retrieval work and enforces a small
  fixed-window rate limit, but every cache failure safely bypasses to the authoritative component.

### Run the included retrieval evaluation

This uses Qdrant's in-memory mode and the actual embedding model, so it requires neither Docker
nor a Qdrant server:

```bash
python -m app.evaluation.run_retrieval_evaluation
```

## Acceptance criteria for this milestone

- Five support documents can be ingested into a `support_docs` Qdrant collection.
- `POST /api/v1/search` returns matching chunks with scores and complete source metadata.
- Invalid input and an empty collection are handled safely.
- DNS validation, NXDOMAIN, timeouts, missing records, and resolver failures have distinct
  structured representations.
- The diagnosis endpoint returns selected route, tools called, evidence, confidence, errors, and
  escalation state.
- Generated answers retain tool evidence and application-controlled citations; provider failures
  return safe uncertainty rather than crashing the workflow.
- Conversations and diagnosis runs are durable, queryable, and transactionally consistent; a
  failed persistence transaction returns a structured error rather than a fabricated saved state.
- DNS/retrieval cache hits, misses, TTL expiry, malformed entries, Redis failure bypasses, and
  fixed-window rate limiting are covered by isolated tests.
- `GET /health` returns service status and environment.
- `GET /ready` distinguishes an application-ready service from future dependency checks.
- The endpoints have automated tests.

## Phase 8: SSL, HTTP, and evidence fusion

`check_ssl(domain)` opens a normal TLS connection and validates the presented certificate using the
system trust store. A valid connection reports issuer, subject, dates, hostname match, and days to
expiry. **Expiry** means the validity end date passed; **hostname mismatch** means the certificate
does not cover the requested domain. Timeout, refused connection, DNS failure, malformed
certificate, not-yet-valid certificate, and handshake failures remain distinct structured errors.

`check_http(url_or_domain)` performs a bounded `GET` with `Range: bytes=0-0`, no body storage, a
configurable five-second timeout, and at most three redirects. It reports final URL, status,
redirects, content type, HTTPS use, latency, and error state. HTTP families are: 2xx success, 3xx
redirect, 4xx client/access issue, and 5xx server/upstream issue. 500 is application failure; 502
is bad gateway; 503 means service unavailable; 504 is gateway timeout.

SSRF protection runs before every initial and redirected HTTP target: only public `http`/`https`
targets are allowed. Localhost, loopback, `0.0.0.0`, private/reserved IPv4, private IPv6,
link-local addresses, and metadata targets are blocked. SSRF is when user input tricks a server
into reaching internal resources; these rules prevent the tool becoming an internal-network scanner.

The graph chooses only relevant tools: DNS+RAG for DNS, DNS→SSL→RAG for SSL, DNS→HTTP→RAG for
website-down, HTTP+RAG for hosting, RAG-only for documentation, and conservative RAG for unknown
queries. DNS failure explicitly skips dependent SSL/HTTP calls. Evidence fusion preserves facts
side-by-side, notices conflicts such as DNS OK + HTTP 503, reduces confidence, and escalates instead
of forcing a diagnosis. The LLM explains these structured facts; it does not independently invent
infrastructure state.

SSL results use a 300-second Redis TTL; HTTP-status results use 30 seconds. Transient DNS,
timeout, and connection failures are not cached. PostgreSQL stores compact summaries only—never
page bodies, cookies, authorization headers, or secrets.

```bash
python -m app.evaluation.run_diagnostic_scenarios
```

## Phase 9: human-in-the-loop review

Human-in-the-loop (HITL) means that the copilot can collect and explain evidence, but a person
reviews cases that deterministic policy marks as uncertain or high-risk. This is not a model-only
decision: review is required for low confidence, tool failure, conflicting evidence, missing RAG
evidence, DNS dependency blocks, LLM failure, unknown intent, or ambiguous root causes. A normal
high-confidence, evidence-backed diagnosis is marked `not_required`.

When review is required, the graph enters `pending`, preserves evidence, and appends a clear notice
that the case must not be treated as resolved. It does not silently pass the checkpoint. Future
write actions—DNS changes, hosting changes, ticket creation, service restarts, or SSL changes—must
remain behind this explicit approval boundary; none are implemented now.

Review state is durable in PostgreSQL: `pending`, `approved`, `rejected`, or `resolved`, with reason,
reviewed time, decision, and notes. Approval means a human accepts the proposed recommendation;
rejection means it was not accepted; pending explicitly means unresolved.

```bash
curl http://127.0.0.1:8000/api/v1/escalations
curl http://127.0.0.1:8000/api/v1/escalations/ESCALATION_ID
curl -X POST http://127.0.0.1:8000/api/v1/escalations/ESCALATION_ID/review \
  -H "Content-Type: application/json" \
  -d '{"decision":"approved","notes":"Evidence supports a hosting-side outage."}'
python -m app.evaluation.run_escalation_evaluation
```

HITL reduces unsafe automation because it makes uncertainty visible and reviewable rather than
allowing an agent to make consequential changes from incomplete evidence. Escalation is a controlled
state, not an application failure: the diagnostic record, citations, and tool observations remain
available for the reviewer.

## Phase 10: observability and operational diagnostics

Each `/api/v1/diagnose` response now contains a `request_id`, `trace_id`, and safe per-node timing
map. The same correlation IDs flow through graph nodes, retrieval, tools, LLM generation, persistence
logs, and review handling. Logs are JSON-style events with component, operation, status, latency, and
normalized error type. They deliberately redact API keys, passwords, authorization/cookie fields,
prompts, queries, and raw content; chain-of-thought is never captured.

`/health` is a **liveness** endpoint: it says the process is running and never checks dependencies.
`/ready` is dependency-aware: it reports `ready` or `degraded` and compact PostgreSQL, Redis, Qdrant,
and configured-LLM status when those dependencies are configured. A dependency outage never makes the
liveness endpoint fail. `/metrics` exposes small in-process counters and latency summaries (average,
P50, P95) for diagnosis, graph nodes, tools, retrieval, Redis cache hit/miss, rate-limit rejections,
LLM failures, and token totals.

Logs answer *what happened*; metrics answer *how often and how slow*; traces connect one request's
steps. Token and latency metadata makes model cost/slowdown visible without storing full prompts.
Optional external tracing is intentionally not required: telemetry failure must fail open so a diagnosis
can still complete. Observability helps distinguish a retrieval miss, bad tool routing, provider failure,
or unsupported claim without treating model text as ground truth.

```bash
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/ready
curl http://127.0.0.1:8000/metrics
```

## Phase 11: evaluation and regression gates

The offline corpus contains 100 realistic DNS, TLS, website, hosting, documentation, ambiguous, and
out-of-scope scenarios. Every record carries the expected intent, tools, skipped tools, support source,
escalation outcome, optional error type, and root-cause category. It includes paraphrases, incomplete
reports, conflicting symptoms, and deliberately unsupported requests.

Run the reproducible local evaluation with:

```bash
python -m app.evaluation.run_full_evaluation
```

It produces `data/evaluation/reports/phase11_summary.json`, `phase11_results.csv`, and
`phase11_report.md`. The command returns a non-zero exit code when quality gates fail. Measurements
are labelled carefully: orchestration/grounding/reliability use mocked tools and provider; retrieval uses
an in-memory local Qdrant dense index over the five support documents. They are not live-production
measurements.

Precision asks “when the system says yes, how often is it right?” Recall asks “of the real positives,
how many did it find?” Precision@k measures whether relevant evidence appears in the first *k* results;
MRR rewards putting the first relevant result early. A confusion matrix exposes where one intent is
mistaken for another. Deterministic checks are the primary grounding signal; any future LLM judge would
be advisory rather than ground truth. Benchmark datasets can become biased, so the corpus must evolve
and must not be used as the sole optimization target. Business outcomes—resolution time, reviewer load,
and customer satisfaction—are different from model metrics.

## Phase 12: security hardening and red-team testing

The copilot treats user queries and retrieved documents as **untrusted data**. Deterministic controls
block obvious prompt injection and secret-exfiltration attempts before an LLM call. Hostile instructions
inside retrieved documents are removed from model context; documents remain evidence, never commands.
Generated output is screened for secret-looking content before response or persistence. These are
defense-in-depth guardrails, not a claim that pattern matching catches every future attack.

Support metadata now carries future-compatible `tenant_id`, `document_scope`, and `access_level`.
Retrieval permits public documents plus the requested scope, and scope is part of the Redis retrieval
cache key. No authentication system exists yet, so future callers must only supply scope after an
authenticated authorization boundary. DNS, SSL, HTTP, and RAG are `READ_ONLY`; future write tools require
explicit approval and destructive tools remain blocked.

HTTP validates destinations before use and after each redirect, rejecting local/private/reserved addresses,
IPv6 loopback/mapped loopback, metadata services, and non-HTTP schemes. Sensitive-looking emails, phone
numbers, account identifiers, connection strings, and key patterns are redacted before persistence/logging
where possible. Security controls fail closed; observability may fail open but must not weaken a block.

```bash
python -m app.evaluation.run_security_evaluation
```

The 56-case **offline security evaluation** covers direct/indirect injection, secrets, SSRF, write-tool
abuse, and cross-tenant retrieval. It writes `data/evaluation/reports/phase12_security.json` and returns
non-zero on a security-gate failure. Limitations: this is not production-security certification; it does
not replace authentication, encrypted secret management, a WAF, sandboxing, or independently reviewed
authorization. Ingestion metadata must be assigned correctly.

## Phase 13: deployment and CI/CD preparation

**IMPLEMENTED:** a non-root Python 3.12 Docker image, Docker Compose topology for FastAPI,
PostgreSQL, Redis, Qdrant, and Nginx; internal-only data services; Docker health checks; production
configuration validation; GitHub Actions workflow; deployment smoke script; and an EC2/Nginx runbook.

**VERIFIED locally:** deployment configuration tests, lint, application tests, Phase 11 regression gate,
and Phase 12 security gate. **PREPARED:** AWS EC2 deployment configuration and runbook.
**NOT YET DEPLOYED:** AWS infrastructure and GitHub Actions execution are not claimed until the remote
repository is pushed and the workflow genuinely runs.

```bash
cp .env.example .env                 # edit values; never commit this file
docker compose up -d postgres redis qdrant
docker compose run --rm app alembic upgrade head
docker compose run --rm app python -m app.rag.bootstrap
docker compose up -d app nginx
SMOKE_BASE_URL=http://127.0.0.1 python scripts/smoke_test.py
```

Use `uvicorn` workers without `--reload` in the container; reload watches files and is only suitable for
development. Choose worker count through `WEB_CONCURRENCY` after measuring the deployment. See
[deployment.md](docs/deployment.md) for EC2 security-group rules, migration coordination, restart
behavior, rollback, secret handling, and resource-sizing limits.

## Learning checkpoint

Be ready to explain: embeddings versus LLMs, cosine similarity, vector payloads, chunk overlap,
`top_k`, dependency injection, and why retrieval results should include traceable sources. For
DNS tooling, explain each record type, why NXDOMAIN differs from a missing record and timeout,
why unit tests mock the resolver, and why an AI agent should begin with read-only diagnostics.
For LangGraph, explain typed state, nodes, edges, conditional routing, deterministic versus LLM
routing, and why intermediate tool output must remain inspectable.
For grounded generation, explain why the LLM interprets evidence rather than creates it, why
citations are application-controlled, retrieval versus model confidence, low temperature, and why
provider calls are mocked in tests.
For observability, explain logs versus metrics versus traces, correlation IDs, liveness versus
readiness, P50/P95 latency, dependency degradation, safe LLM token tracking, and why prompt/chain-of-
thought logging is unsafe.
For evaluation, explain precision/recall, Precision@k, MRR, confusion matrices, regression gates,
offline evaluation versus production monitoring, benchmark bias, and overfitting.
For security, explain direct/indirect injection, untrusted RAG content, least privilege, SSRF, tenant/cache
isolation, PII minimization, secret management, AI red teaming, and defense in depth.

