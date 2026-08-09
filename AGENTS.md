# AGENTS.md

## Scope

- This repo is a Docker Compose observability config for lower environments, not an application repo.
- The main stack is intentionally self-contained in `docker-compose.yaml`; do not reintroduce a separate Collector config file unless requested.
- Production application images use Datadog, so do not modify application Dockerfiles to enable Jaeger.

## Runtime

- Start with `docker compose up -d` or `podman compose up -d`.
- Validate config with `docker compose config` or `podman compose config`.
- Jaeger UI is `http://localhost:16686`.
- Apps send OTLP to the Collector on `localhost:4317` or `localhost:4318` from the host, and `otel-collector:4317` or `http://otel-collector:4318` from Compose networks.

## Architecture

- Apps must send telemetry to `otel-collector`, not directly to `jaeger`.
- `jaeger` uses in-memory storage and is intended for local/lower environments only.
- The root Compose file creates the external-facing network named `observability`; app templates under `app-tracer-autoinstrument/*/docker-compose.yaml` expect that network to already exist.

## Application Template

- `app-tracer-autoinstrument/` contains lower-environment compose templates split by runtime: `dotnet-10/`, `python-313-fastapi/`, and `java-21-quarkus/`.
- Each runtime folder must keep its own `INSTRUCTIONS.md` aligned with that folder's `docker-compose.yaml`.
- Every application template must keep `cpus: "0.5"`, `mem_limit: 700m`, and `memswap_limit: 700m` unless the user changes the hardware limit.
- Java injection uses `/opt/otel-agents/java/opentelemetry-javaagent.jar` mounted to `/otel/opentelemetry-javaagent.jar` and loaded with `JAVA_TOOL_OPTIONS`.
- .NET injection uses `/opt/otel-agents/dotnet` mounted to `/otel/dotnet` and loaded with `CORECLR_*` and `DOTNET_*` variables; keep architecture paths explicit.
- Python injection installs OTel packages at startup and runs `opentelemetry-bootstrap -a install` before `opentelemetry-instrument`; do not add Dockerfiles here to install it.
- Keep the root `INSTRUCTIONS.md` as a short index; put detailed tracer setup in the runtime-specific instruction files.
- Keep Datadog/prod behavior out of this repo unless the user explicitly asks for a production migration.
