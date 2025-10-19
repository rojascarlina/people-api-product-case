# Contract-first: Conventions and Guidelines (People API)

This document defines **how we design and evolve** the People API in **contract-first** mode using **OpenAPI**.

---

## 1) Principles
- **Contract-first**: the contract (OpenAPI) is agreed upon before implementation.
- **Predictable**: clear resources, consistent naming, standard errors.
- **Backward compatibility**: “additive” changes in v1; “breaking” → v2.
- **DX**: TTFHW < 30’; ready-to-use examples, Postman, Swagger one-click.

---

## 2) Resource Modeling
- **Plural** and **noun-based** names: `/v1/employees`, `/v1/absences`.
- **Hierarchy when applicable**: `/v1/employees/{employeeId}/absences`.
- **Opaque identifiers**: `emp_123`, `abs_987`.
- **Country-specific extensions**: avoid forks; use optional fields or an `extensions` block.


**Example (OpenAPI fragmento)**
```yaml
paths:
  /v1/employees/{employeeId}/absences:
    get:
      summary: List absences by employee
      parameters:
        - in: path
          name: employeeId
          required: true
          schema: { type: string }
        - in: query
          name: from
          required: true
          schema: { type: string, format: date }
        - in: query
          name: to
          required: true
          schema: { type: string, format: date }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items: { $ref: '#/components/schemas/Absence' }
```
## 3) Paths & Methods (CRUD)

### Operations Summary
| Action               | Verb    | Path                                          |
|----------------------|---------|-----------------------------------------------|
| Create request       | `POST`  | `/v1/absences/requests`                       |
| List by employee     | `GET`   | `/v1/employees/{employeeId}/absences`         |
| Get details          | `GET`   | `/v1/absences/{absenceId}`                    |
| Partial update       | `PATCH` | `/v1/absences/{absenceId}`                    |
| Approve request      | `POST`  | `/v1/absences/{absenceId}/approve`            |

**Pagination/Filtering**
- Pagination based on **page/limit** or **cursor** (do not mix within the same operation).
- Explicit filters (avoid free-text search in v1).
- Sorting with `?sort=createdAt&order=desc`.

---

### OpenAPI — Endpoints (fragmentos listos)

```yaml
paths:
  /v1/absences/requests:
    post:
      summary: Create absence request
      operationId: createAbsenceRequest
      tags: [Absences]
      parameters:
        - $ref: '#/components/headers/Idempotency-Key'
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/AbsenceRequest' }
            examples:
              example:
                value:
                  employeeId: emp_123
                  type: VACATION
                  startDate: 2025-10-06
                  endDate: 2025-10-10
                  country: ES
                  reason: Family trip
      responses:
        "201":
          description: Created
          headers:
            Idempotency-Key: { $ref: '#/components/headers/Idempotency-Key' }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Absence' }
        "400":
          description: Bad Request
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
        "409":
          description: Conflict (duplicate idempotency key or business conflict)
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }

  /v1/employees/{employeeId}/absences:
    get:
      summary: List absences by employee
      operationId: listEmployeeAbsences
      tags: [Absences]
      parameters:
        - in: path
          name: employeeId
          required: true
          schema: { type: string }
        - in: query
          name: from
          required: true
          schema: { type: string, format: date }
        - in: query
          name: to
          required: true
          schema: { type: string, format: date }
        - in: query
          name: page
          schema: { type: integer, minimum: 1 }
        - in: query
          name: limit
          schema: { type: integer, minimum: 1, maximum: 200 }
        - in: query
          name: cursor
          schema: { type: string }
        - in: query
          name: sort
          schema: { type: string, enum: [createdAt, startDate] }
        - in: query
          name: order
          schema: { type: string, enum: [asc, desc], default: desc }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items: { $ref: '#/components/schemas/Absence' }
                  pageInfo:
                    $ref: '#/components/schemas/PageInfo'
        "400":
          description: Bad Request
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }

  /v1/absences/{absenceId}:
    get:
      summary: Get absence by id
      operationId: getAbsence
      tags: [Absences]
      parameters:
        - in: path
          name: absenceId
          required: true
          schema: { type: string }
      responses:
        "200":
          description: OK
          headers:
            ETag:
              description: Weak ETag for concurrency control
              schema: { type: string, example: 'W/"abs_987-v3"' }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Absence' }
        "404":
          description: Not Found
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }

    patch:
      summary: Patch absence (status transitions, metadata)
      operationId: patchAbsence
      tags: [Absences]
      parameters:
        - in: path
          name: absenceId
          required: true
          schema: { type: string }
        - in: header
          name: If-Match
          required: true
          schema: { type: string, example: 'W/"abs_987-v3"' }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              additionalProperties: false
              properties:
                status:
                  $ref: '#/components/schemas/AbsenceStatus'
                reason:
                  type: string
                  maxLength: 500
      responses:
        "200":
          description: Updated
          headers:
            ETag:
              description: New ETag after update
              schema: { type: string, example: 'W/"abs_987-v4"' }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Absence' }
        "400":
          description: Bad Request
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
        "412":
          description: Precondition Failed (ETag mismatch)
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }

  /v1/absences/{absenceId}/approve:
    post:
      summary: Approve absence
      operationId: approveAbsence
      tags: [Absences, Actions]
      parameters:
        - in: path
          name: absenceId
          required: true
          schema: { type: string }
      responses:
        "200":
          description: Approved
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Absence' }
        "409":
          description: Conflict (invalid state transition)
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
        "404":
          description: Not Found
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
```

## 4) Idempotencia y concurrencia

- **Idempotencia** en creación: header **`Idempotency-Key` obligatorio** para `POST` que crean recursos (permite reintentos seguros sin duplicar).
- **Clave única** por operación de negocio (p. ej., `POST /v1/absences/requests`): UUID v4 o hash estable del payload+actor.
- **TTL** recomendado para la clave: 24–72 h (persistir estado de idempotencia en servidor).
- **Respuestas**:
  - Primer intento → `201 Created` + recurso.
  - Reintento con misma clave → `201 Created` (mismo `id`) o `409 Conflict` si cambia el payload.
- **Concurrencia** optimista en actualizaciones: **`ETag` + `If-Match`** en `PATCH/PUT` para evitar overwrites.
- **Reintentos de cliente**: usa backoff exponencial; respeta `Retry-After` si existe.

### OpenAPI — header y uso en POST
```yaml
components:
  headers:
    Idempotency-Key:
      description: Required for safely retrying POST without creating duplicates.
      schema: { type: string, format: uuid }
      required: true

paths:
  /v1/absences/requests:
    post:
      summary: Create absence request
      parameters:
        - $ref: '#/components/headers/Idempotency-Key'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AbsenceRequest'
      responses:
        "201":
          description: Created
          headers:
            Idempotency-Key:
              $ref: '#/components/headers/Idempotency-Key'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Absence'
        "409":
          description: Conflict (same Idempotency-Key with different payload)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
```

## 5) Esquemas (nombres y formatos)

- **Nomenclatura JSON:** `camelCase` para propiedades; `PascalCase` para `schemas`.
- **Fechas/horas:** `date` = `YYYY-MM-DD`; `date-time` = ISO-8601 (UTC o con offset).
- **Identificadores:** cadenas opacas (`emp_123`, `abs_987`), no exponer claves internas.
- **Moneda:** ISO-4217 (`EUR`, `USD`); cantidades como **decimal string** (evita flotantes).
- **Enums:** valores **MAYÚSCULAS_CON_GUIONES** o **UPPER_SNAKE_CASE** documentados.
- **Localización/país:** ISO-3166-1 alpha-2 (`ES`, `PT`, `MX`).
- **Extensiones locales:** bloque `extensions` (objeto abierto) para diferencias por país/cliente.
- **Paginación:** `page`, `limit` o `cursor`; respuestas con `items` + `pageInfo`.
- **Metadatos comunes:** `createdAt`, `updatedAt` (`date-time`).

### Componentes reutilizables (OpenAPI)
```yaml
components:
  schemas:
    CountryCode:
      type: string
      minLength: 2
      maxLength: 2
      description: ISO-3166-1 alpha-2 country code (e.g., ES, PT, MX)

    CurrencyCode:
      type: string
      minLength: 3
      maxLength: 3
      description: ISO-4217 currency code (e.g., EUR, USD, MXN)

    Money:
      type: object
      required: [amount, currency]
      properties:
        amount:
          type: string
          pattern: "^-?\\d{1,12}(\\.\\d{1,2})?$"
          description: Decimal string with up to 2 fraction digits (avoid floats)
          example: "1234.56"
        currency:
          $ref: '#/components/schemas/CurrencyCode'

    PageInfo:
      type: object
      properties:
        page:
          type: integer
          minimum: 1
          example: 1
        limit:
          type: integer
          minimum: 1
          maximum: 200
          example: 50
        total:
          type: integer
          minimum: 0
          example: 137
        nextCursor:
          type: string
          nullable: true
          description: Opaque cursor for next page (if cursor-based pagination)
          example: "eyJwYWdlIjoyfQ=="

    Problem:
      type: object
      required: [type, title, status]
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer, minimum: 100, maximum: 599 }
        detail: { type: string }
        instance: { type: string }
components:
  schemas:
    AbsenceType:
      type: string
      description: Business absence category
      enum: [VACATION, SICKNESS, PARENTAL, OTHER]
      example: VACATION

    AbsenceStatus:
      type: string
      enum: [REQUESTED, APPROVED, REJECTED, CANCELLED]
      example: REQUESTED

    AbsenceRequest:
      type: object
      required:
        - employeeId
        - type
        - startDate
        - endDate
        - country
      properties:
        employeeId:
          type: string
          example: "emp_123"
        type:
          $ref: '#/components/schemas/AbsenceType'
        startDate:
          type: string
          format: date
          example: "2025-10-06"
        endDate:
          type: string
          format: date
          example: "2025-10-10"
        reason:
          type: string
          maxLength: 500
          example: "Family trip"
        country:
          $ref: '#/components/schemas/CountryCode'
        # Optional, to illustrate payroll implications if needed
        estimatedCost:
          $ref: '#/components/schemas/Money'
        extensions:
          type: object
          additionalProperties: true
          description: |
            Optional local/customer-specific fields (e.g., ES: medicalCertificateId)
          example:
            medicalCertificateId: "doc_555"

    Absence:
      type: object
      required:
        - id
        - employeeId
        - type
        - status
        - startDate
        - endDate
        - country
        - createdAt
        - updatedAt
      properties:
        id:
          type: string
          example: "abs_987"
        employeeId:
          type: string
          example: "emp_123"
        type:
          $ref: '#/components/schemas/AbsenceType'
        status:
          $ref: '#/components/schemas/AbsenceStatus'
        startDate:
          type: string
          format: date
        endDate:
          type: string
          format: date
        country:
          $ref: '#/components/schemas/CountryCode'
        approvedBy:
          type: string
          nullable: true
          description: Manager ID who approved the request
          example: "mgr_55"
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time
        extensions:
          type: object
          additionalProperties: true
          description: Optional local/customer-specific fields on stored entity
```
