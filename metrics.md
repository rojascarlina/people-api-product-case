# API Health & Performance Metrics

- **%4xx por endpoint** = peticiones cliente inválidas (contrato, auth, validación, rate limit). Objetivo ≤ 1–2% semanal.
- **%5xx por endpoint** = fallos servidor (timeouts, bugs). Objetivo ≤ 0.1–0.5%.
- **Latencia p95** = el 95% de respuestas es ≤ p95. Objetivos típicos: GET ≤ 250–400 ms; POST ≤ 500–800 ms.
- **TTFHW (Time To First Hello World)** del integrador: objetivo < 30 min.
- **Adopción por versión** y eventos de **deprecation** (Sunset).
- **Onboarding** de nuevo país/cliente (días).

> Instrumentar por **endpoint, versión y client_id**. Alertas si %4xx > 2% (1h), %5xx > 0.1% (15m) o p95 supera objetivo.