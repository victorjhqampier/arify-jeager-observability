# Inyeccion OpenTelemetry Para Python 3.13 FastAPI

Esta plantilla inyecta OpenTelemetry en una aplicacion Python 3.13 FastAPI sin cambiar codigo fuente ni el Dockerfile de produccion.

## Artefacto Requerido

Por defecto no se requiere ningun artefacto manual.

La plantilla de Compose instala los paquetes OpenTelemetry al iniciar el contenedor usando el venv de uv:

```bash
/axapp/.venv/bin/python -m pip install --no-cache-dir opentelemetry-distro opentelemetry-exporter-otlp
/axapp/.venv/bin/python -m opentelemetry.instrumentation.bootstrap -a install
/axapp/.venv/bin/python -m opentelemetry.instrumentation.auto_instrumentation /axapp/.venv/bin/uvicorn Presentation.app:app --app-dir src --host 0.0.0.0 --port 80
```

## Requisitos

La imagen de la aplicacion debe tener:

- Runtime compatible con Python 3.13.
- Virtual environment de `uv` en `/axapp/.venv/`.
- Acceso a internet para instalar paquetes.
- Permisos de escritura en el venv para instalar paquetes.
- Un entrypoint ASGI que coincida con `Presentation.app:app`, o ajustar el `command` del Compose.

## Como Se Inyecta

La plantilla de Compose instala OpenTelemetry en el venv de uv, ejecuta bootstrap discovery y arranca FastAPI mediante el modulo de auto-instrumentacion:

```yaml
command:
  - sh
  - -c
  - >
    /axapp/.venv/bin/python -m pip install --no-cache-dir
    opentelemetry-distro
    opentelemetry-exporter-otlp &&
    /axapp/.venv/bin/python -m opentelemetry.instrumentation.bootstrap -a install &&
    /axapp/.venv/bin/python -m opentelemetry.instrumentation.auto_instrumentation
    /axapp/.venv/bin/uvicorn Presentation.app:app --app-dir src --host 0.0.0.0 --port 80 --workers 1 --loop uvloop --http httptools
```

**Nota:** Se usa `python -m opentelemetry.instrumentation.auto_instrumentation` en lugar del wrapper `opentelemetry-instrument` para evitar errores de `sys.executable=None` en entornos con venv de uv.

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
docker compose -f app-tracer-autoinstrument/python-313-fastapi/docker-compose.yaml config
podman compose -f app-tracer-autoinstrument/python-313-fastapi/docker-compose.yaml config
```

Verifica que los paquetes existan despues de que el contenedor arranque:

```bash
docker compose exec business-api /axapp/.venv/bin/python -m pip list | grep opentelemetry
```

La app esta disponible en `http://localhost:8000` (el contenedor escucha en 80, el host expone 8000).

Genera trafico contra la app y luego revisa logs del Collector y Jaeger:

```bash
docker compose logs -f otel-collector
```

Abre `http://localhost:16686` y busca `business-api` o el `OTEL_SERVICE_NAME` configurado.

## Notas

- Instalar paquetes al inicio es practico para ambientes bajos, pero es mas lento y depende de acceso a internet.
- Para ambientes controlados, considera agregar las dependencias de OpenTelemetry directamente en el `pyproject.toml` del proyecto.
- Si tu imagen usa pip estandar en lugar de uv, cambia `/axapp/.venv/bin/python` por `python` y ajusta los paths.
- Cambia `Presentation.app:app` y `--app-dir src` si la app FastAPI real usa otra ruta de modulo.
- Se usa `python -m opentelemetry.instrumentation.auto_instrumentation` en lugar del script `opentelemetry-instrument` para evitar el error `TypeError: execv: path should be string, bytes or os.PathLike, not NoneType` que ocurre cuando `sys.executable` es `None` en entornos con venv de uv.
