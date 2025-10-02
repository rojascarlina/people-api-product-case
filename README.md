# People APIs — Product Case (People)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**Author:** Carlina — Technical Product Manager (Producto API)  
**Context:** Product approach to build a **People API** from scratch (discovery → contract-first (OpenAPI) → governance → multi‑client → metrics).

> This repository is a concise portfolio artifact to share with recruiters/tech customer teams.  
> It contains: a minimal **OpenAPI** spec, a **30‑60‑90** plan, and a one‑pager.

## Índice
- 📄 [One-pager](docs/one-pager.md)
- 🗓️ [30-60-90](docs/30-60-90.md)
- 📊 [Métricas API](docs/metrics.md)
- 🔐 [OpenAPI v1 (YAML)](openapi/people-absences-api.yaml)
- 🔧 [Postman collection](openapi/postman_collection.json)


## Quick Preview
```mermaid
flowchart TD
  A[Discovery multi-país] --> B[Modelo de dominio]
  B --> C[Contrato OpenAPI v1]
  C --> D[Portal dev + sandbox + webhooks]
  D --> E[Pilotos 2 países]
  E --> F[Observabilidad: %4xx/%5xx, p95]
  F --> G[Governance & deprecations]
```

## How to use

- Click **Open in Swagger Editor** (badge above) to explore the API contract.  
  Direct link: https://editor.swagger.io/?url=https://raw.githubusercontent.com/rojascarlina/people-inditex-apis-product-case/main/openapi/people-absences-api.yaml
- Review the **one-pager** and the **30-60-90** in `/docs`:
  - One-pager: [docs/one-pager.md](docs/one-pager.md)
  - 30-60-90: [docs/30-60-90.md](docs/30-60-90.md)
- (Optional) Import the **Postman collection** to try the endpoints:
  https://raw.githubusercontent.com/rojascarlina/people-inditex-apis-product-case/main/openapi/postman_collection.json


## Why this matters
- **Contract‑first** enables speed + quality across teams and countries.  
- **Retrocompatibility** protects existing clients while we evolve.  
- **Observability & KPIs** keep the product honest and measurable.

---

*Last updated:* 2025-10-02

## How to visualize the OpenAPI
You can explore the contract in **Swagger UI** or **ReDoc** in seconds:

**Option A — Online (no install):**
### Demo rápida

👉 **Abrir OpenAPI en Swagger (1 clic):**  
https://editor.swagger.io/?url=https://raw.githubusercontent.com/rojascarlina/people-inditex-apis-product-case/main/people-absences-api.yaml

**Postman (RAW):** https://raw.githubusercontent.com/rojascarlina/people-inditex-apis-product-case/main/postman_collection.json


**Option B — ReDoc (online viewer):**
- Use any ReDoc online viewer and paste the *raw* URL of the YAML once the repo is public.

**Option C — Local (Docker Swagger UI):**
```bash
docker run -p 8080:8080 -e SWAGGER_JSON=/foo/people-absences-api.yaml       -v $(pwd)/openapi:/foo swaggerapi/swagger-ui
# Open http://localhost:8080
```

## Postman
- Import `openapi/postman_collection.json` into Postman.
- Set `{{base_url}}`, `{{access_token}}`, and variables as needed.
---

## NOTICE
Este repositorio es un **caso de producto personal** para evaluación.
No está afiliado ni patrocinado por **Inditex**. Todas las marcas mencionadas
pertenecen a sus respectivos propietarios.
