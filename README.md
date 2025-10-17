# OpenAPI / AsyncAPI Stubs

This bundle contains first-pass API stubs for each service in the compiler/interpreter decomposition.
They are intentionally minimal and generator-friendly.

- `source-service.openapi.yaml`
- `lexer-service.openapi.yaml`
- `parser-service.openapi.yaml`
- `icode-service.openapi.yaml`
- `symtab-service.openapi.yaml`
- `orchestrator-service.openapi.yaml`
- `codegen-service.openapi.yaml`
- `execution-service.openapi.yaml`
- `gateway-service.openapi.yaml`
- `event-bus.asyncapi.yaml` (for the Kafka/NATS/Rabbit event backbone)

All OpenAPI files are 3.1.0; the event bus uses AsyncAPI 3.0.0.

> Tip: Use `openapi-generator-cli` or `oapi-codegen` to generate server/clients.
