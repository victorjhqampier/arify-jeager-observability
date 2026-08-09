# Jaeger Observability Stack

Stack **local/de ambientes bajos** para trazas con **OpenTelemetry Collector** y **Jaeger**. **Produccion sigue usando Datadog**; **no modifiques el Dockerfile ni el codigo fuente** real de la aplicacion para este flujo.

Para inyectar tracers en aplicaciones sin tocar codigo ni Dockerfile, sigue el `INSTRUCTIONS.md` de cada runtime bajo `app-tracer-autoinstrument/`.

## Componentes

- `otel-collector`: punto de entrada OTLP para las aplicaciones.
- `jaeger`: almacenamiento en memoria y UI para revisar trazas.

## Endpoints

- Jaeger UI: `http://localhost:16686`
- OTLP gRPC: `localhost:4317`
- OTLP HTTP: `http://localhost:4318`

Las aplicaciones dentro de la red `observability` deben usar `otel-collector:4317` para gRPC o `http://otel-collector:4318` para HTTP/protobuf.

## Docker

Validar la configuracion:

```bash
docker compose config
```

Levantar el stack:

```bash
docker compose up -d
```

Ver logs:

```bash
docker compose logs -f otel-collector jaeger
```

Apagar el stack:

```bash
docker compose down
```

## Podman

Validar la configuracion:

```bash
podman compose config
```

Levantar el stack:

```bash
podman compose up -d
```

Ver logs:

```bash
podman compose logs -f otel-collector jaeger
```

Apagar el stack:

```bash
podman compose down
```

Si tu instalacion usa `podman-compose`, reemplaza `podman compose` por `podman-compose`.

## Variables Para Aplicaciones

Configuracion recomendada para ambientes bajos:

```bash
OTEL_SERVICE_NAME=your-service-name
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=none
OTEL_LOGS_EXPORTER=none
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=lower,service.namespace=local
```

Este repositorio no modifica Dockerfiles de aplicaciones porque produccion esta integrado con Datadog.

## Plantillas De Aplicacion

`app-tracer-autoinstrument/` contiene plantillas de compose/override separadas por lenguaje para ambientes bajos:

- `.NET 10`: `app-tracer-autoinstrument/dotnet-10/docker-compose.yaml`
- `Python 3.13 FastAPI`: `app-tracer-autoinstrument/python-313-fastapi/docker-compose.yaml`
- `Java 21 Quarkus`: `app-tracer-autoinstrument/java-21-quarkus/docker-compose.yaml`

Cada carpeta incluye su propia guia especializada:

- `.NET 10`: `app-tracer-autoinstrument/dotnet-10/INSTRUCTIONS.md`
- `Python 3.13 FastAPI`: `app-tracer-autoinstrument/python-313-fastapi/INSTRUCTIONS.md`
- `Java 21 Quarkus`: `app-tracer-autoinstrument/java-21-quarkus/INSTRUCTIONS.md`

Cada plantilla aplica limites por contenedor:

- CPU: `cpus: "0.5"`
- RAM: `mem_limit: 700m`
- Swap: `memswap_limit: 700m`

Las plantillas cargan tracers de forma externa, similar al enfoque de Datadog:

- `.NET 10`: monta OpenTelemetry .NET Auto-Instrumentation desde `/opt/otel-agents/dotnet`.
- `Python 3.13 FastAPI`: instala `opentelemetry-distro`, `opentelemetry-exporter-otlp`, ejecuta `opentelemetry-bootstrap -a install` y arranca con `opentelemetry-instrument`.
- `Java 21 Quarkus`: monta `opentelemetry-javaagent.jar` desde `/opt/otel-agents/java/opentelemetry-javaagent.jar` y lo carga con `JAVA_TOOL_OPTIONS`.

Ajusta el nombre del servicio, la imagen y el comando de arranque segun la aplicacion real. Mantener estas plantillas fuera de produccion.

Validar una plantilla especifica con Docker:

```bash
docker compose -f app-tracer-autoinstrument/python-313-fastapi/docker-compose.yaml config
```

Validar una plantilla especifica con Podman:

```bash
podman compose -f app-tracer-autoinstrument/java-21-quarkus/docker-compose.yaml config
```
