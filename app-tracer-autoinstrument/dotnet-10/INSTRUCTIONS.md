# Auto-Instrumentacion OpenTelemetry Para .NET 10

Esta plantilla inyecta OpenTelemetry en una aplicacion .NET 10 sin cambiar codigo fuente ni el Dockerfile de produccion.

## Artefacto Requerido

Descarga el paquete OpenTelemetry .NET Auto-Instrumentation desde los releases oficiales de OpenTelemetry .NET instrumentation y extraelo en el host en:

```text
/opt/otel-agents/dotnet/
```

La carpeta extraida debe contener las rutas usadas por la plantilla Compose, incluyendo:

```text
/opt/otel-agents/dotnet/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll
/opt/otel-agents/dotnet/AdditionalDeps
/opt/otel-agents/dotnet/store
/opt/otel-agents/dotnet/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so
```

Usa la ruta que corresponda a la arquitectura. Para imagenes ARM64, cambia `linux-x64` por `linux-arm64` en `CORECLR_PROFILER_PATH` cuando esa ruta exista en el paquete.

## Como Se Inyecta

La plantilla de Compose monta la carpeta de auto-instrumentacion dentro del contenedor en modo solo lectura:

```yaml
/opt/otel-agents/dotnet:/otel/dotnet:ro
```

El CLR carga el tracer mediante variables de profiler y startup hook:

```yaml
CORECLR_ENABLE_PROFILING: "1"
CORECLR_PROFILER: "{918728DD-259F-4A6A-AC2B-B85E1B658318}"
CORECLR_PROFILER_PATH: /otel/dotnet/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so
DOTNET_STARTUP_HOOKS: /otel/dotnet/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll
DOTNET_ADDITIONAL_DEPS: /otel/dotnet/AdditionalDeps
DOTNET_SHARED_STORE: /otel/dotnet/store
OTEL_DOTNET_AUTO_HOME: /otel/dotnet
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
docker compose -f app-tracer-autoinstrument/dotnet-10/docker-compose.yaml config
podman compose -f app-tracer-autoinstrument/dotnet-10/docker-compose.yaml config
```

Verifica que los archivos de auto-instrumentacion esten montados y que las variables esten presentes:

```bash
docker compose exec dotnet-10-app test -f /otel/dotnet/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll
docker compose exec dotnet-10-app env | grep -E 'CORECLR|DOTNET_|OTEL'
```

La app esta disponible en `http://localhost:5001` (el contenedor escucha en 8080, el host expone 5001).

Genera trafico contra la app y luego revisa logs del Collector y Jaeger:

```bash
docker compose logs -f otel-collector
```

Abre `http://localhost:16686` y busca `dotnet-10-app` o el `OTEL_SERVICE_NAME` configurado.

## Notas

- La ruta del profiler nativo debe coincidir con la arquitectura del contenedor de la aplicacion.
- Si la imagen ya define `DOTNET_STARTUP_HOOKS` u otras variables `DOTNET_*`, combina los valores cuidadosamente en vez de sobrescribirlos sin revisar.
