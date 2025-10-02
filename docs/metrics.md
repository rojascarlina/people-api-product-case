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
> Ajusta etiquetas a tu stack.

**%5xx por endpoint (últimos 5 min):**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) by (route)
/
sum(rate(http_requests_total[5m])) by (route)
%4xx por endpoint (últimos 5 min):
sum(rate(http_requests_total{status=~"4.."}[5m])) by (route)
/
sum(rate(http_requests_total[5m])) by (route)
Latencia p95:
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)
)
Por versión y cliente (ejemplo %5xx):
sum(rate(http_requests_total{status=~"5.."}[5m])) by (route, version, client_id)
/
sum(rate(http_requests_total[5m])) by (route, version, client_id)
Alertas (reglas simples y efectivas)
Errores 5xx:
ALERT api_5xx_high IF %5xx > 0.1% DURING 15m
Errores 4xx (contrato/DX):
ALERT api_4xx_high IF %4xx > 2% DURING 60m
Latencia p95:
ALERT api_latency_p95_high IF p95 > objetivo DURING 15m
Disponibilidad (SLO):
alerta por incumplimiento mensual o presupuestos de error.
Logging & errores (DX)
RFC-7807 (application/problem+json) para respuestas de error consistentes:
{
  "type": "https://api.example.com/problems/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "endDate must be after startDate",
  "instance": "/v1/absences/requests"
}
Registra request_id, client_id, version, country, y motivo (reason) para analizar picos de 4xx.
4) Umbrales/objetivos sugeridos (tabla)
| Métrica                 | Objetivo guía                         | Comentario                                   |
| ----------------------- | ------------------------------------- | -------------------------------------------- |
| %5xx por endpoint       | ≤ **0.1–0.5%** semanal                | Buscar ≈0%; revisar dependencias/timeouts    |
| %4xx por endpoint       | ≤ **1–2%** semanal                    | >2% → revisar contrato, auth, validaciones   |
| p95 GET                 | ≤ **250–400 ms**                      | Observar p99 para outliers                   |
| p95 POST/PUT            | ≤ **500–800 ms**                      | People complejo puede subir a ~1 s (vigilar) |
| TTFHW                   | **< 30 min** (*< 60 min* si complejo) | OpenAPI + Postman + Quickstart               |
| Onboarding cliente/país | Tendencia **↓**                       | Track por país/partner                       |
| Disponibilidad (SLO)    | **≥ 99.9%** operaciones críticas      | Definir por endpoint                         |

5) Uso de métricas para decisiones de producto
%4xx alto → ¿Contrato confuso? Añade ejemplos, valida server-side con mensajes útiles, revisa auth/rate limit, Quickstart más claro.
%5xx/p95 alto → Incidencia técnica: dependencias, caché, índices, colas, escalado, idempotencia y backoff.
TTFHW alto → Falta de Quickstart, Postman sin variables, auth confusa, errores poco explicativos.
Adopción v2 baja → Ajustar plan de migración, Sunset header, guía de cambios y período de convivencia.
6) Ejemplo de objetivos por fase
MVP (0–90 días)
Instrumentación base, dashboards por endpoint/versión/cliente.
%5xx ≤ 0.5%, %4xx ≤ 2%, p95 dentro de target en 2 países piloto.
TTFHW < 30 min, Postman + Quickstart.
Escala (90–180 días)
Alertas activas, reportes por país/cliente y versión.
Deprecation policy con Sunset, adopción monitorizada.
Onboarding por país ↓ 30–50% respecto a baseline.
7) Glosario rápido
p95/p99: percentil 95/99 del tiempo de respuesta.
TTFHW: Time To First Hello World (primer éxito del integrador).
SLO/SLA: objetivo de servicio / acuerdo de nivel de servicio.
RFC-7807: formato estándar de errores (Problem Details).
Sunset: cabecera HTTP para comunicar deprecación futura.
