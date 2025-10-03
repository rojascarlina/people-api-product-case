# Contract-first: convenciones y guías (People API)

Este documento define **cómo diseñamos y evolucionamos** la People API en modo **contract-first** usando **OpenAPI** como única fuente de verdad.

---

## 1) Principios
- **Contract-first**: el contrato (OpenAPI) se acuerda antes de implementar.
- **Predecible**: recursos claros, nombres consistentes, errores estándar.
- **Retrocompatibilidad**: cambios “additive” en v1; “breaking” → v2.
- **DX**: TTFHW < 30’; ejemplos listos, Postman, Swagger 1-clic.

---

## 2) Modelado de recursos
- **Nombres en plural** y **sustantivos**: `/v1/employees`, `/v1/absences`.
- **Jerarquía cuando aplica**: `/v1/employees/{employeeId}/absences`.
- **Identificadores opacos**: `emp_123`, `abs_987`.
- **Extensiones por país**: evitar forks; usar campos opcionales o un bloque `extensions`.

**Ejemplo (OpenAPI fragmento)**
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
## 3) Paths & Methods (CRUD)

| Acción                | Verbo   | Ejemplo                                         |
|----------------------|---------|--------------------------------------------------|
| Crear                | `POST`  | `/v1/absences/requests`                          |
| Leer (colección)     | `GET`   | `/v1/employees/{id}/absences?from=&to=`          |
| Leer (detalle)       | `GET`   | `/v1/absences/{absenceId}`                       |
| Actualizar parcial   | `PATCH` | `/v1/absences/{absenceId}`                       |
| Aprobación/acciones  | `POST`  | `/v1/absences/{absenceId}/approve`               |

**Paginación/filtrado**
- Paginación: `?page=1&limit=50` (o `cursor` si esperas >100k).
- Ordenación: `?sort=createdAt&order=desc`.
- Filtros: campos explícitos (evitar búsqueda libre en v1).

**Ejemplo (OpenAPI fragmento)**
```yaml
paths:
  /v1/absences/{absenceId}/approve:
    post:
      summary: Approve absence
      parameters:
        - in: path
          name: absenceId
          required: true
          schema: { type: string }
      responses:
        "200": { description: Approved }
        "409":
          description: Conflict (already approved/invalid state)
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
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
