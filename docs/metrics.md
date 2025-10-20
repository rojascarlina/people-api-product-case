# API Health, Performance & DX Metrics

This document defines the **minimum metrics** to manage a **People API** as a product: health, performance, and developer experience (DX). It includes **objectives**, **how to instrument**, and **recommended alerts**.

---

## 1) Core Metrics (per endpoint / version / client)

### %4xx — Client Errors
- **What it is:** percentage of responses with **4xx** status codes (400, 401, 403, 404, 409, 422, 429…).
- **What it indicates:** misunderstood contract, validation issues, auth errors, rate limits, incorrect routes.
- **Target guideline:** **≤ 1–2%** weekly (can be >2% for login/permissions if failed attempts are expected).
- **Formula:**
```
%4xx = (nº responses 4xx / nº total responses) × 100
```

### %5xx — Server Errors
- **What it is:** percentage of responses with **5xx** status codes (500, 502, 503, 504…).
- **What it indicates:** server-side issues — timeouts with dependencies, bugs, overload.
- **Target guideline:** **≤ 0.1–0.5%** weekly (aim for **≈ 0%**).
- **Formula:**
```
%5xx = (nº responses 5xx / nº total responses) × 100
```

### p95 (and p99) Latency — Performance
- **What it is:** 95th (or 99th) percentile of response time per operation.
- **Why p95:** reflects real experience without distortion from outliers.
- **Target guidelines:**
  - **GET** (read): **p95 ≤ 250–400 ms**
  - **POST/PUT** (write): **p95 ≤ 500–800 ms**
  - Complex People operations (multi-country rules/calculations) may tolerate up to ~1 s, but **monitor trends**.
- **Note:** track **p99** to detect queues/incidents.

### Availability (SLA/SLO)
- **Quick definition:** % of time endpoints meet error/latency targets.
- **Example SLO:** `Monthly availability ≥ 99.9%` for critical operations (create absence request, check balances).

---

## 2) DX Metrics (Developer Experience)

### TTFHW — *Time To First Hello World*
- **What it is:** time from when an integrator opens the Quickstart to their **first successful API call** (200/201).
- **Target guideline:** **< 30 min** (if complex auth/compliance: **< 60 min**).
- **How to reduce it:** OpenAPI + Swagger/Redoc, Postman collection with variables (`base_url`, `token`, `Idempotency-Key`), 5-step Quickstart, clear RFC-7807 errors, sandbox.

### Client/Country Onboarding (days)
- **What it is:** time from kick-off to first production flow.
- **Target guideline:** downward trend **↓**; report by country to identify friction points.

### Version Adoption
- **What it is:** relative usage of **v1** vs **v2** (and deprecation events).
- **Target guideline:** planned migrations with **zero surprises**.

---

## 3) How to Instrument (Practical)

### Recommended Dimensions
- `endpoint`/`route`, `method`, `version` (v1, v2), `client_id` (integrator), `country`, `status_code`, `request_id`.
- **Correlation:** store `request_id`/`trace_id` in logs to jump from metric → trace → log.

### Prometheus Examples (PromQL)

**%5xx per endpoint (last 5 min)**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) by (route)
/ 
sum(rate(http_requests_total[5m])) by (route)
```

**%4xx por endpoint (last 5 min)**
```promql
sum(rate(http_requests_total{status=~"4.."}[5m])) by (route)
/ 
sum(rate(http_requests_total[5m])) by (route)
```

**Latency p95**
```promql
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)
)
```

**By version and client (example %5xx)**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) by (route, version, client_id)
/ 
sum(rate(http_requests_total[5m])) by (route, version, client_id)
```

### Alerts (Simple and Effective Rules)

**Errors 5xx**
```txt
ALERT api_5xx_high IF %5xx > 0.1% DURING 15m
```

**Errors 4xx (contrato/DX)**
```txt
ALERT api_4xx_high IF %4xx > 2% DURING 60m
```

**Latency p95**
```txt
ALERT api_latency_p95_high IF p95 > objetivo DURING 15m
```

**Availability (SLO)**
```txt
Alert for monthly breach or “error budget” exhaustion..
```

### Logging & errors (DX)

**RFC-7807** (`application/problem+json`) for consistent error responses:
```json
{
  "type": "https://api.example.com/problems/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "endDate must be after startDate",
  "instance": "/v1/absences/requests"
}
```

Registra `request_id`, `client_id`, `version`, `country` y el **motivo** (`reason`) para analizar picos de 4xx.

---

## 4) Suggested thresholds/targets (table)

| Metric                  | Guiding Objetive                         | Comment                                        |
|-------------------------|---------------------------------------|---------------------------------------------------|
| %5xx por endpoint       | ≤ **0.1–0.5%** semanal                | Search ≈0%; review dependencies/timeouts         |
| %4xx por endpoint       | ≤ **1–2%** semanal                    | >2% → review contract, auth, validations        |
| p95 GET                 | ≤ **250–400 ms**                      | Monitor p99 for outliers                       |
| p95 POST/PUT            | ≤ **500–800 ms**                      | Complex People ops up to ~1 s (watch trend)      |
| TTFHW                   | **< 30 min** (*< 60 min* si complejo) | OpenAPI + Postman + Quickstart                    |
| Onboarding client/country | Trend **↓**                       | Track by country/partner                            |
| Availability (SLO)    | **≥ 99.9%** operaciones críticas      | Define by endpoint                              |

---

## 5) Using Metrics for Product Decisions

- **High %4xx** → Contract confusion? Add examples, validate server-side with useful messages, review auth/rate limits, make Quickstart clearer.  
- **High %5xx/p95** → Technical issue: dependencies, cache, indexes, queues, scaling, idempotency, and backoff.  
- **High TTFHW** → Missing Quickstart, Postman without variables, confusing auth, poor error messages.  
- **Low v2 adoption** → Adjust migration plan, use Sunset header, add changelog and coexistence period.

---

## 6) Example Targets by Phase

**MVP (0–90 days)**
- Base instrumentation, dashboards by endpoint/version/client.  
- **%5xx ≤ 0.5%**, **%4xx ≤ 2%**, **p95** within target for 2 pilot countries.  
- **TTFHW < 30 min**, Postman + Quickstart.

**Scale (90–180 days)**
- Active alerts, reports by country/client/version.  
- Deprecation policy with `Sunset`, monitored adoption.  
- Onboarding per country **↓ 30–50%** vs baseline.

---

## 7) Quick Glossary

- **p95/p99:** 95th/99th percentile of response time.  
- **TTFHW:** *Time To First Hello World* (integrator’s first success).  
- **SLO/SLA:** service level objective / service level agreement.  
- **RFC-7807:** standard error format (Problem Details).  
- **Sunset:** HTTP header used to announce upcoming deprecation.
