# Compiler Contracts (OpenAPI & AsyncAPI Stubs)

First-pass service contracts for a microservice compiler/interpreter stack. Minimal, generator-friendly specs for quick client/server scaffolding and contract-first development.

Services & specs:

* `source-service.openapi.yaml`
* `lexer-service.openapi.yaml`
* `parser-service.openapi.yaml`
* `icode-service.openapi.yaml`
* `symtab-service.openapi.yaml`
* `orchestrator-service.openapi.yaml`
* `codegen-service.openapi.yaml`
* `execution-service.openapi.yaml`
* `gateway-service.openapi.yaml`
* `event-bus.asyncapi.yaml` (Kafka/NATS/Rabbit backbone)

**Spec versions:** OpenAPI **3.0.3** (services), AsyncAPI **3.0.0** (event bus).

> Tip: Use `openapi-generator-cli` (multi-lang) or `oapi-codegen` (Go) to generate clients/servers.

---

## Overview

This repo defines the boundaries for a compiler pipeline split into focused services:

* **Source** (ingest & stream lines)
* **Lexer** (tokens & SSE stream)
* **Parser** (async parse + summaries)
* **IR/ICode** (intermediate form store)
* **SymTab** (symbol tables & queries)
* **Orchestrator** (job lifecycle)
* **Codegen** (artifact build from IR+SymTab)
* **Execution** (run artifacts; logs/SSE)
* **Gateway** (CLI/API facade)
* **Event Bus** (typed topics for pipeline telemetry)

---

## Quickstart: Code Generation

### OpenAPI Generator (TypeScript & Go examples)

```bash
# Install once (Java required)
brew install openapi-generator     # macOS
# or: sdk install openapi-generator

# TypeScript (fetch)
openapi-generator generate \
  -i parser-service.openapi.yaml \
  -g typescript-fetch \
  -o clients/ts/parser

# Go client
openapi-generator generate \
  -i execution-service.openapi.yaml \
  -g go \
  -o clients/go/execution
```

### oapi-codegen (Go)

```bash
# go install github.com/deepmap/oapi-codegen/cmd/oapi-codegen@latest
oapi-codegen -generate types,client \
  -o clients/go/gateway/gateway.gen.go \
  -package gateway \
  gateway-service.openapi.yaml
```

---

## Service Contracts at a Glance

| Service          | Base URL                | Key Endpoints / Notes                                                                                            |
| ---------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Source**       | `http://localhost:7001` | `POST /sources` (text or multipart) → `id`; `GET /sources/{id}`; `GET /sources/{id}/stream` (SSE `SOURCE_LINE`). |
| **Lexer**        | `http://localhost:7002` | `POST /lex?stream=true` (SSE `TOKEN`) or JSON tokens.                                                            |
| **Parser**       | `http://localhost:7003` | `POST /parses` → `parseId`; `GET /parses/{id}`; `GET /parses/{id}/summary` (`ParserSummary`).                    |
| **ICode**        | `http://localhost:7004` | `POST /icode` (store IR) → `ICodeRef`; `GET /icode/{id}`.                                                        |
| **SymTab**       | `http://localhost:7005` | `POST /symtabs` → `SymTabRef`; `GET /symtabs/{id}`; `GET /symtabs/{id}/symbols?name=...`.                        |
| **Orchestrator** | `http://localhost:7006` | `POST /jobs` (compile/execute); `GET /jobs/{id}`; `POST /jobs/{id}/cancel`.                                      |
| **Codegen**      | `http://localhost:7007` | `POST /compile` → `artifactId`; `GET /artifacts/{id}`; `GET /artifacts/{id}/download`.                           |
| **Execution**    | `http://localhost:7008` | `POST /execute` → `Run`; `GET /runs/{id}`; `GET /runs/{id}/logs` (SSE summaries/errors/stdout).                  |
| **Gateway**      | `http://localhost:7000` | `POST /v1/compile` (file or text); `POST /v1/execute` (file or text + `stdin`).                                  |

### Event Bus Channels (AsyncAPI 3.0.0)

* `source.line` → `SourceLine`
* `lexer.token` → `Token`
* `parser.summary` → `ParserSummary`
* `compiler.summary` → `CompilerSummary`
* `execution.interpreter_summary` → `InterpreterSummary`
* `execution.runtime_error` → `RuntimeError`

---

## Usage Examples

### A) Compile & Execute via Gateway (simplest)

```bash
# Compile text
curl -s -X POST 'http://localhost:7000/v1/compile' \
  -F 'language=pascal' \
  -F 'text=program Hello; begin writeln(''hi''); end.' \
| jq

# Execute text with stdin
curl -s -X POST 'http://localhost:7000/v1/execute' \
  -F 'language=pascal' \
  -F 'text=program Echo; var s:string; begin readln(s); writeln(s); end.' \
  -F 'stdin=hello world' \
| jq
```

### B) Low-level Flow (per service)

```bash
# 1) Create source (text/plain)
SRC_ID=$(curl -s -X POST 'http://localhost:7001/sources' \
  -H 'Content-Type: text/plain' \
  --data-binary $'program P; begin writeln(\'ok\'); end.' \
| jq -r .id)

# 2) Stream tokens (SSE)
curl -N -X POST "http://localhost:7002/lex?stream=true" \
  -H 'Accept: text/event-stream' -H 'Content-Type: application/json' \
  -d "{\"sourceId\":\"$SRC_ID\"}"

# 3) Parse
PARSE_ID=$(curl -s -X POST 'http://localhost:7003/parses' \
  -H 'Content-Type: application/json' \
  -d "{\"sourceId\":\"$SRC_ID\"}" \
| jq -r .parseId)

# 4) Summary → get IR & SymTab IDs
curl -s "http://localhost:7003/parses/$PARSE_ID/summary" | jq

# Assume you now have ICODE_ID and SYMTAB_ID:

# 5) Codegen
ARTIFACT_ID=$(curl -s -X POST 'http://localhost:7007/compile' \
  -H 'Content-Type: application/json' \
  -d "{\"icodeId\":\"$ICODE_ID\",\"symTabId\":\"$SYMTAB_ID\"}" \
| jq -r .artifactId)

# 6) Execute and stream logs
RUN_ID=$(curl -s -X POST 'http://localhost:7008/execute' \
  -H 'Content-Type: application/json' \
  -d "{\"icodeId\":\"$ICODE_ID\",\"symTabId\":\"$SYMTAB_ID\"}" \
| jq -r .id)

curl -N "http://localhost:7008/runs/$RUN_ID/logs" \
  -H 'Accept: text/event-stream'
```

---

## Installation (tools you’ll want)

* **OpenAPI Generator**: `brew install openapi-generator` (or see releases)
* **oapi-codegen (Go)**: `go install github.com/deepmap/oapi-codegen/cmd/oapi-codegen@latest`
* **jq** and `curl` for examples
* Optional validators:

  * `openapi-generator validate -i <file>`
  * `npm i -g @redocly/cli` → `redocly lint <file>`

---

## Local Dev Tips

* Keep specs single-source-of-truth; generate clients/servers into separate folders (`clients/`, `servers/`) that are **not** committed if you prefer clean repos.
* Prefer SSE (`text/event-stream`) for live token/line/log streams; pass `-N` to `curl` to disable buffering.
* Shared error envelope across services:

  ```json
  { "code": "SOME_CODE", "message": "Human readable", "details": { /* any */ } }
  ```

---

## API Reference (selected schemas)

* **ParserSummary**: `{ parseId, statementCount, syntaxErrors, elapsedMs }`
* **Run** (Execution): `{ id, state, exitCode?, stdout?, stderr?, startedAt?, finishedAt?, error? }`
* **CompileResult**: `{ artifactId, instructionCount, elapsedMs }`
* **Token**: `{ type, text?, value, line, position, sequence }`
* **SourceCreated**: `{ id, bytes, language, createdAt }`

Consult each `*.openapi.yaml` / `*.asyncapi.yaml` for the complete, authoritative definitions.

---

## Deployment

This repo is **contracts only**—no runtime services here. Typical patterns:

* Publish artifacts to your internal registry (e.g., package the YAML under `npm`, `pip`, or a Git tag).
* Gate changes via CI: lint/validate specs + regenerate sample clients to catch breaking changes.

---

## Environment Variables

*Not required* for the contracts themselves. Downstream services/clients may use:

* `GATEWAY_URL` (default `http://localhost:7000`)
* `BROKER_URL` / `KAFKA_BOOTSTRAP_SERVERS` (for event bus consumers/producers)

---

## Appendix

* Move to OpenAPI **3.1** when ready (nullable changes, JSON Schema vocab alignment).
* Consider JSON Schema `$defs` reuse if you split common envelopes/types.
