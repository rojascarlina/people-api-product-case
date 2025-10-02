# API Health, Performance & DX Metrics

Este documento define las **métricas mínimas** para gestionar una **People API** como producto: salud, rendimiento y experiencia del integrador (DX). Incluye **objetivos**, **cómo instrumentar** y **alertas** recomendadas.

## 1) Métricas principales (por endpoint / versión / cliente)

### %4xx — Errores de cliente
- **Qué es:** porcentaje de respuestas con códigos **4xx** (400, 401, 403, 404, 409, 422, 429…).
- **Qué indica:** contrato mal entendido, validaciones, auth, rate limits, rutas incorrectas.
- **Objetivo guía:** **≤ 1–2%** semanal (puede ser >2% en login/permiso si hay intentos fallidos esperables).
- **Fórmula:**  
%4xx = (nº respuestas 4xx / nº total respuestas) × 100

### %5xx — Errores de servidor
- **Qué es:** porcentaje de respuestas **5xx** (500, 502, 503, 504…).
- **Qué indica:** fallos del lado servidor: timeouts con dependencias, bugs, saturación.
- **Objetivo guía:** **≤ 0.1–0.5%** semanal (aspirar a **≈ 0%**).
- **Fórmula:**  
%5xx = (nº respuestas 5xx / nº total respuestas) × 100

### Latencia p95 (y p99) — Rendimiento
- **Qué es:** percentil 95 (o 99) del tiempo de respuesta por operación.
- **Por qué p95:** refleja la experiencia real sin distorsión de outliers.
- **Objetivos guía:**
- **GET** (lectura): **p95 ≤ 250–400 ms**
- **POST/PUT** (escritura): **p95 ≤ 500–800 ms**
- Operaciones People complejas (cálculos/reglas multi-país) pueden tolerar hasta ~1 s, pero **vigilar tendencia**.
- **Nota:** monitorea **p99** para detectar colas/incidentes.

### Disponibilidad (SLA/SLO)
- **Definición rápida:** % de tiempo en que los endpoints cumplen objetivos de error/latencia.
- **Ejemplo SLO:** `Disponibilidad mensual ≥ 99.9%` para operaciones críticas (crear solicitud de ausencia, consultar saldos).

---

## 2) Métricas de DX (Developer Experience)

### TTFHW — *Time To First Hello World*
- **Qué es:** tiempo desde que un integrador abre el Quickstart hasta su **primera llamada exitosa** (200/201).
- **Objetivo guía:** **< 30 min** (si auth/compliance compleja: **< 60 min**).
- **Cómo reducirlo:** OpenAPI + Swagger/Redoc, colección Postman con variables (`base_url`, `token`, `Idempotency-Key`), Quickstart 5 pasos, errores RFC-7807 claros, sandbox.

### Onboarding de cliente/país (días)
- **Qué es:** tiempo desde kick-off hasta primer flujo en producción.
- **Objetivo guía:** tendencialmente **↓**; reportar por país para ver dónde duele.

### Adopción por versión
- **Qué es:** uso relativo de **v1** vs **v2** (y eventos de deprecación).
- **Objetivo guía:** migraciones con plan y **cero sorpresas**.

---

## 3) Cómo instrumentar (práctico)

### Dimensiones recomendadas
- `endpoint`/`route`, `method`, `version` (v1, v2), `client_id` (integrador), `country`, `status_code`, `request_id`.
- **Correlación:** guarda `request_id`/`trace_id` en logs para saltar de métrica → traza → log.

### Ejemplos Prometheus (PromQL)

**%5xx por endpoint (últimos 5 min)**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) by (route)
/
sum(rate(http_requests_total[5m])) by (route)
```promql

