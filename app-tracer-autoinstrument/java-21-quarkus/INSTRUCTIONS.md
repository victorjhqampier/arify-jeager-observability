# Inyeccion OpenTelemetry Para Java 21 Quarkus

Esta plantilla inyecta OpenTelemetry en una aplicacion Quarkus JVM sin cambiar codigo fuente ni el Dockerfile de produccion.

## Artefacto Requerido

Descarga el OpenTelemetry Java agent desde los releases oficiales de OpenTelemetry Java instrumentation y colocalo en el host en:

```text
/opt/otel-agents/java/opentelemetry-javaagent.jar
```

## Como Se Inyecta

La plantilla de Compose monta el agent dentro del contenedor en modo solo lectura:

```yaml
/opt/otel-agents/java/opentelemetry-javaagent.jar:/otel/opentelemetry-javaagent.jar:ro
```

La JVM carga el tracer mediante `JAVA_TOOL_OPTIONS`:

```yaml
JAVA_TOOL_OPTIONS: "-javaagent:/otel/opentelemetry-javaagent.jar"
```

La telemetria se envia al Collector con:

```yaml
OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4318
OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf
OTEL_TRACES_EXPORTER: otlp
```

No apuntes la app directamente a `jaeger`.

## Limites

La plantilla mantiene el limite de hardware para ambientes bajos:

```yaml
cpus: "0.5"
mem_limit: 700m
memswap_limit: 700m
```

## Validacion

Valida el archivo Compose:

```bash
docker compose -f app-tracer-autoinstrument/java-21-quarkus/docker-compose.yaml config
podman compose -f app-tracer-autoinstrument/java-21-quarkus/docker-compose.yaml config
```

Verifica que el agent este montado y que el flag de JVM este presente:

```bash
docker compose exec java-21-quarkus-app test -f /otel/opentelemetry-javaagent.jar
docker compose exec java-21-quarkus-app sh -c 'echo "$JAVA_TOOL_OPTIONS"'
```

La app esta disponible en `http://localhost:8080`.

Genera trafico contra la app y luego revisa logs del Collector y Jaeger:

```bash
docker compose logs -f otel-collector
```

Abre `http://localhost:16686` y busca `java-21-quarkus-app` o el `OTEL_SERVICE_NAME` configurado.

## Notas

- Esto aplica a Quarkus en modo JVM.
- Las imagenes Quarkus native/GraalVM no cargan Java agents de la misma forma.
- Si la imagen ya define `JAVA_TOOL_OPTIONS`, combina el valor existente con `-javaagent:/otel/opentelemetry-javaagent.jar` en vez de sobrescribirlo.
