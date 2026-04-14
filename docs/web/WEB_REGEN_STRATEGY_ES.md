# Estrategia de regeneración de SUMO hacia una plataforma web moderna (Mapbox + OpenStreetMap)

## 1) Lectura del repositorio y diagnóstico técnico

### Qué existe hoy y cómo encaja
- **Núcleo de simulación maduro y de alto rendimiento** en C++ (SUMO y herramientas relacionadas), con construcción vía CMake y una base muy extensa de módulos y pruebas.
- **Pipeline geoespacial ya funcional para OSM** en Python, reutilizable para backend web:
  - descarga OSM/Overpass (`osmGet.py`),
  - conversión/red de simulación (`osmBuild.py` desde `osmWebWizard.py`),
  - generación de demanda (`randomTrips.py`) y PT (`ptlines2flows.py`).
- **UI web existente (OSM Web Wizard)** basada en HTML/JS + Leaflet + jQuery, con capa de servidor HTTP/WebSocket en Python.

### Fortalezas heredables
1. Motor de simulación y utilidades consolidadas.
2. Flujo OSM->red SUMO->escenario ya integrado extremo a extremo.
3. Capacidades de control remoto/progreso ya contempladas (HTTP `/build`, `/progress` y mensajes de estado).

### Limitaciones actuales frente a una web moderna
1. Frontend legacy (sin framework SPA moderno, sin tipado fuerte, sin arquitectura de componentes).
2. Acoplamiento fuerte de orquestación a scripts y procesos locales.
3. Escalabilidad operativa limitada (colas, multiusuario, observabilidad y tenancy no aparecen como diseño de primera clase).
4. Experiencia de mapa centrada en Leaflet raster; no aprovecha completamente estilos vectoriales modernos ni un modelo desacoplado de proveedores.

## 2) Objetivo de plataforma (target)

Construir una **plataforma web multiusuario** para diseñar, ejecutar y analizar simulaciones SUMO desde navegador, con:

- **Mapa moderno vectorial** (estilo Mapbox GL) con compatibilidad OpenStreetMap.
- **Ejecución asíncrona y escalable** de escenarios (jobs).
- **Observabilidad y trazabilidad** (estado, logs, métricas, artefactos).
- **API estable** para integraciones externas.
- **Arquitectura cloud-ready** (contenedores, workers, storage, CI/CD).

## 3) Stack recomendado (mejores herramientas actuales)

### Frontend
- **React + TypeScript + Next.js** (App Router) para DX moderna, SSR/ISR cuando sea útil, y estructura empresarial.
- **MapLibre GL JS** como motor base de mapas (compatibilidad con Mapbox Style Spec, evitando lock-in);
  - consumir estilos de **Mapbox** cuando se disponga de token,
  - fallback/alternativa con estilos y tiles OSM abiertos.
- **TanStack Query** para estados remotos, reintentos y caché de jobs.
- **Zod** para validación de contratos en cliente.
- **Playwright** para E2E (flujos de mapa + lanzamiento de simulación).

### Backend
- **FastAPI (Python)** para API de orquestación, ideal por cercanía natural al ecosistema actual (scripts SUMO en Python).
- **Pydantic v2** para esquemas estrictos de request/response.
- **Celery o Dramatiq + Redis** para cola de trabajos de simulación.
- **PostgreSQL + PostGIS** para persistencia geoespacial y metadatos de escenarios.
- **Object storage** (S3 compatible/MinIO) para artefactos (redes, rutas, outputs, ZIPs).
- **WebSocket/SSE** para progreso en vivo de jobs.

### Runtime / DevOps
- **Docker + Kubernetes (opcional por fase)** para aislar workers SUMO.
- **GitHub Actions** para CI (lint, test, build, security scan, imágenes).
- **OpenTelemetry + Prometheus + Grafana + Loki** para observabilidad integral.

## 4) Arquitectura propuesta

1. **Web App**: editor de área, configuración de demanda, catálogo de escenarios, monitor de ejecución y visor de resultados.
2. **API Gateway / Backend**:
   - AuthN/AuthZ (OIDC/JWT),
   - endpoints de escenarios/jobs,
   - firma y entrega de URLs de descarga.
3. **Job Orchestrator**:
   - encola ejecución,
   - lanza workers con timeout/cuotas,
   - persiste estados (`queued/running/succeeded/failed`).
4. **SUMO Worker**:
   - reutiliza `osmGet.py`, `osmBuild` y generación de demanda,
   - ejecuta sumo/sumo-gui headless según tipo de job,
   - publica progreso y sube outputs.
5. **Geo/Data Layer**:
   - PostGIS para geometrías y bounding boxes,
   - storage de objetos para archivos pesados.

## 5) Estrategia de migración por fases

### Fase 0 — Descubrimiento y contratos (2-3 semanas)
- Inventariar comandos/opciones hoy usadas por `osmWebWizard.py`.
- Definir **contratos API versionados** (OpenAPI) para: crear escenario, lanzar simulación, consultar progreso, descargar artefactos.
- Acordar modelo de permisos y límites por usuario/proyecto.

### Fase 1 — Backend de compatibilidad (3-5 semanas)
- Crear FastAPI con endpoints equivalentes a `/build` y `/progress`, pero versionados (`/api/v1/jobs`).
- Encapsular el pipeline actual en un **runner** idempotente.
- Guardar estados y logs en DB + storage.

### Fase 2 — Frontend moderno mínimo (4-6 semanas)
- Nueva SPA con mapa MapLibre.
- Selector de área (bbox/polígono), parámetros de demanda/PT, envío de job.
- Vista de progreso en tiempo real y descarga ZIP.

### Fase 3 — Integración Mapbox + OSM avanzada (3-4 semanas)
- Motor de estilos configurable por tenant/proyecto:
  - estilo Mapbox (token privado),
  - estilo OSM/MapLibre público.
- Capas temáticas: red importada, PT, calor de demanda, resultados de simulación.

### Fase 4 — Operación productiva (4-8 semanas)
- CI/CD completo + quality gates.
- Observabilidad end-to-end.
- Hardening de seguridad (rate limits, aislamiento de jobs, controles de egress).

### Fase 5 — Capacidades diferenciales
- Reproducción temporal de simulación en navegador.
- Comparación entre escenarios (A/B).
- API pública para terceros y notebooks.

## 6) Decisiones técnicas clave (recomendadas)

1. **MapLibre primero, Mapbox como proveedor de estilos**: compatibilidad alta y menor riesgo de lock-in.
2. **Python backend** para maximizar reutilización de tooling existente del repo.
3. **Ejecución asíncrona obligatoria** (nunca bloquear request HTTP con simulación).
4. **Artefactos inmutables por job** para reproducibilidad.
5. **Contratos API estrictos** para desacoplar frontend/backend desde el día 1.

## 7) Riesgos y mitigaciones

- **Licenciamiento/costos de mapas (Mapbox)** -> feature flag y fallback OSM.
- **Carga computacional alta** -> colas, cuotas, límites de tamaño de bbox y duración.
- **Complejidad geoespacial** -> PostGIS + validadores de geometría + tests E2E con fixtures reales.
- **Divergencia con scripts legacy** -> capa adaptadora y pruebas de regresión contra salidas conocidas.

## 8) Plan de ejecución inmediato (primeros 30 días)

1. Levantar ADRs (arquitectura, mapas, colas, storage).
2. Crear `api/` FastAPI con endpoint `POST /api/v1/jobs/import-osm` y `GET /api/v1/jobs/{id}`.
3. Implementar runner mínimo que invoque flujo actual y emita progreso.
4. Crear `web/` Next.js con:
   - mapa MapLibre,
   - formulario básico,
   - pantalla de estado de job.
5. Añadir pruebas:
   - unitarias (validaciones de payload),
   - integración (runner sobre fixture pequeño),
   - E2E (flujo usuario completo).

## 9) Criterios de éxito

- Tiempo medio de creación de escenario <= 3 min en bbox pequeña.
- Job reproducible (mismo input -> mismos artefactos relevantes).
- Frontend desacoplado del runner legacy (solo API).
- Observabilidad suficiente para diagnosticar fallos sin acceso al host.

---

## Evidencia principal del repositorio que soporta esta estrategia

- Existencia de un **wizard web legacy** basado en Leaflet/jQuery/HTML estático.
- Integración de capas OSM y geocodificación Nominatim en el cliente.
- Backend Python con endpoints de construcción/progreso y modos HTTP/WebSocket.
- Pipeline de importación OSM robusto con Overpass, retries y filtros.
