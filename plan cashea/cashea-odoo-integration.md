# 🔌 Integración Cashea ↔ Odoo — Plan de Integración

> **Contexto:** Sistema Odoo para empresa de venta de internet, integrado con Cashea (sistema de cobro en cuotas) para que los clientes paguen sus planes de conectividad mediante financiamiento.
>
> **Importante:** Los planes de internet ya están modelados en Odoo como **`product.product` de tipo `service`**, con planes desde **500 MB hasta 2 GB**. La integración debe aprovechar estos productos existentes y no duplicar información.

---

## 📋 Tabla de Contenidos

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [Información a Rastrear en Odoo](#información-a-rastrear-en-odoo)
3. [Arquitectura de la API](#arquitectura-de-la-api)
4. [Protocolo de Comunicación](#protocolo-de-comunicación)
5. [Endpoints REST a Exponer](#endpoints-rest-a-exponer)
6. [Webhooks desde Cashea hacia Odoo](#webhooks-desde-cashea-hacia-odoo)
7. [Datos a Enviar a Cashea](#datos-a-enviar-a-cashea)
8. [Flujo Completo de la Integración](#flujo-completo-de-la-integración)
9. [Estados y Ciclo de Vida del Pago](#estados-y-ciclo-de-vida-del-pago)
10. [Seguridad y Autenticación](#seguridad-y-autenticación)
11. [Consideraciones Especiales (Servicio vs Producto)](#consideraciones-especiales-servicio-vs-producto)
12. [Modelo de Datos en Odoo](#modelo-de-datos-en-odoo)
13. [Checklist de Implementación](#checklist-de-implementación)

---

## 🎯 Resumen Ejecutivo

La integración tiene dos flujos principales:

- **Odoo → Cashea:** Registrar un "pedido de financiamiento" para un plan de internet, pasando los datos del cliente, el monto total y el plan de cuotas seleccionado.
- **Cashea → Odoo (Webhook):** Notificar sobre pagos de cuotas realizados, rechazos, retrasos o cancelaciones para que Odoo actualice el estado del contrato en tiempo real.

La estrategia recomendada es una **API REST + Webhooks bidireccionales**, que es el estándar de la industria para integraciones financieras de este tipo.

---

## 📦 Información a Rastrear en Odoo

### Nuevo Modelo: `cashea.payment.plan` (Plan de Pago Cashea)

| Campo | Tipo | Descripción |
|---|---|---|
| `name` | Char | Referencia interna (ej: `CASHEA-2024-00123`) |
| `partner_id` | Many2one → `res.partner` | Cliente que paga |
| `contract_id` | Many2one → Contrato de Internet | Contrato asociado al plan de internet |
| `cashea_order_id` | Char | ID del pedido en el sistema de Cashea |
| `cashea_plan_type` | Selection | `monthly` / `quarterly` / `semiannual` / `annual` |
| `total_amount` | Monetary | Monto total a financiar |
| `installment_count` | Integer | Número de cuotas totales |
| `installment_amount` | Monetary | Monto de cada cuota |
| `first_installment_date` | Date | Fecha del primer pago |
| `last_installment_date` | Date | Fecha del último pago (fin del financiamiento) |
| `state` | Selection | Estado del plan (ver sección de Estados) |
| `cashea_status` | Char | Estado raw devuelto por Cashea |
| `credit_limit_used` | Monetary | Cuánto crédito Cashea consumió este plan |
| `approved_at` | Datetime | Fecha en que Cashea aprobó el financiamiento |
| `completed_at` | Datetime | Fecha en que se completaron todas las cuotas |
| `cancelled_at` | Datetime | Fecha de cancelación (si aplica) |
| `notes` | Text | Notas internas / log de eventos |

---

### Nuevo Modelo: `cashea.installment` (Cuota Individual)

| Campo | Tipo | Descripción |
|---|---|---|
| `plan_id` | Many2one → `cashea.payment.plan` | Plan al que pertenece |
| `installment_number` | Integer | Número de cuota (1, 2, 3…) |
| `due_date` | Date | Fecha de vencimiento de la cuota |
| `amount` | Monetary | Monto de esta cuota |
| `state` | Selection | `pending` / `paid` / `overdue` / `partially_paid` |
| `paid_date` | Date | Fecha en que se efectuó el pago |
| `paid_amount` | Monetary | Monto efectivamente pagado (puede variar con cargos de mora) |
| `cashea_transaction_id` | Char | ID de transacción en Cashea |
| `late_fee` | Monetary | Cargo por mora (si aplica) |
| `invoice_id` | Many2one → `account.move` | Factura Odoo relacionada (si se factura cada cuota) |

---

### Extensión del Contrato de Internet

Agregar en el modelo de contrato:

| Campo | Tipo | Descripción |
|---|---|---|
| `cashea_plan_ids` | One2many → `cashea.payment.plan` | Todos los planes Cashea asociados |
| `active_cashea_plan_id` | Many2one (computed) | Plan Cashea activo actualmente |
| `has_cashea_financing` | Boolean (computed) | Si tiene financiamiento activo |

### Referencia al Producto Odoo Existente

El modelo `cashea.payment.plan` debe incluir también:

| Campo | Tipo | Descripción |
|---|---|---|
| `product_id` | Many2one → `product.product` | El plan de internet (producto tipo servicio en Odoo) |
| `product_name` | Char (related) | Nombre del plan ej: `"Internet 500MB"` |
| `product_speed` | Char (related/stored) | Velocidad del plan ej: `"500MB"`, `"1GB"`, `"2GB"` |
| `list_price` | Float (related) | Precio de lista mensual del producto |

> 💡 **No es necesario crear un nuevo modelo de producto.** El `product.product` de tipo servicio ya existente en Odoo es la fuente de verdad. Cashea recibirá los datos derivados de ese registro.

---

## 🏗️ Arquitectura de la API

```
┌──────────────────────────────────────────────────────┐
│                      ODOO                            │
│                                                      │
│  ┌─────────────────┐     ┌──────────────────────┐   │
│  │  Módulo Cashea  │────▶│  API REST Controller │   │
│  │  (addon Odoo)   │     │  /api/cashea/v1/...  │   │
│  └────────┬────────┘     └──────────┬───────────┘   │
│           │                         │                │
│           ▼                         ▼                │
│  ┌─────────────────┐     ┌──────────────────────┐   │
│  │  Webhook Handler│◀────│  Outbound HTTP Client│   │
│  │  (recibe eventos│     │  (llama a Cashea API) │   │
│  │   de Cashea)    │     └──────────────────────┘   │
│  └─────────────────┘                                 │
└──────────────────────────────────────────────────────┘
          ▲  │
  Webhook │  │ REST Calls
          │  ▼
┌──────────────────┐
│     CASHEA       │
│  (App de pagos   │
│    en cuotas)    │
└──────────────────┘
```

### Estrategia Recomendada: **REST API + Webhooks**

| Técnica | Dirección | Uso |
|---|---|---|
| **REST API** (Odoo expone) | Cashea → Odoo | Consultar datos del cliente, contrato, plan |
| **REST API** (Cashea expone) | Odoo → Cashea | Crear orden de financiamiento, consultar estado |
| **Webhook** (Cashea → Odoo) | Cashea → Odoo | Notificar pagos, rechazos, vencimientos |
| **Webhook** (Odoo → Cashea) | Odoo → Cashea | Notificar cambios en el servicio (suspensión, cancelación) |

---

## 📡 Protocolo de Comunicación

### Especificaciones Técnicas

```yaml
Protocolo:     HTTPS (TLS 1.2+, obligatorio)
Formato:       JSON (application/json)
Autenticación: OAuth 2.0 (Client Credentials) o API Key + HMAC Signature
Versionado:    URI versioning (/api/cashea/v1/)
Idempotencia:  Usar Idempotency-Key header en POST para evitar doble procesamiento
Timeout:       30 segundos máximo por request
Retry Policy:  3 reintentos con exponential backoff (para webhooks)
```

### Cabeceras HTTP Estándar

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>
X-Cashea-Signature: <hmac_sha256_signature>
X-Idempotency-Key: <uuid_v4>
X-Request-ID: <uuid_v4>
```

---

## 🔌 Endpoints REST a Exponer (Odoo → Cashea consume)

Estos endpoints los expone **Odoo** para que **Cashea** los llame y obtenga información.

### 1. Obtener Información del Cliente

```http
GET /api/cashea/v1/customers/{customer_id}
```

**Response:**
```json
{
  "customer_id": "RES-PARTNER-456",
  "external_id": "CLI-00123",
  "name": "Juan Pérez",
  "id_number": "V-12345678",
  "email": "juan.perez@email.com",
  "phone": "+58 412-1234567",
  "address": {
    "street": "Av. Principal, Casa 10",
    "city": "Caracas",
    "state": "Distrito Capital",
    "country": "VE",
    "zip": "1010"
  },
  "active_contracts_count": 2,
  "credit_score_internal": "good"
}
```

---

### 2. Obtener Contratos del Cliente

```http
GET /api/cashea/v1/customers/{customer_id}/contracts
```

**Response:**
```json
{
  "customer_id": "RES-PARTNER-456",
  "contracts": [
    {
      "contract_id": "CONT-2024-001",
      "plan_name": "Internet Hogar 50Mbps",
      "plan_code": "IH-50",
      "monthly_fee": 25.00,
      "currency": "USD",
      "status": "active",
      "start_date": "2024-01-01",
      "service_address": "Av. Principal, Casa 10",
      "has_active_cashea_plan": false
    },
    {
      "contract_id": "CONT-2024-002",
      "plan_name": "Internet Empresarial 100Mbps",
      "plan_code": "IE-100",
      "monthly_fee": 60.00,
      "currency": "USD",
      "status": "active",
      "start_date": "2024-03-15",
      "service_address": "Calle Comercio, Local 5",
      "has_active_cashea_plan": false
    }
  ]
}
```

---

### 3. Obtener Detalle de un Contrato / "Producto" para Cashea

> ⚠️ Este es el endpoint más crítico. Cashea está orientada a productos; se expone el **`product.product` de tipo servicio que ya existe en Odoo** mapeado al formato que Cashea espera. No se crea un producto nuevo.

```http
GET /api/cashea/v1/contracts/{contract_id}/product-info
```

**Response:** *(datos leídos directamente del `product.product` y el contrato en Odoo)*
```json
{
  "product": {
    "sku": "[product.default_code de Odoo, ej: INT-500MB]",
    "odoo_product_id": 17,
    "name": "Plan Internet 500MB",
    "description": "Servicio de internet con velocidad de 500MB. Incluye soporte técnico 24/7. Tecnología de fibra óptica.",
    "category": "INTERNET_SERVICE",
    "category_display": "Servicios de Internet",
    "type": "SERVICE",
    "requires_shipping": false,
    "fulfillment_type": "DIGITAL_SERVICE",
    "images": [
      "https://tuempresa.com/web/image/product.product/17/image_1920"
    ],
    "brand": "Tu Empresa Internet",
    "unit": "SUBSCRIPTION"
  },
  "pricing": {
    "monthly_price": 20.00,
    "currency": "USD",
    "tax_rate": 0.16,
    "tax_included": false
  },
  "service_details": {
    "contract_id": "CONT-2024-001",
    "speed": "500MB",
    "technology": "FTTH",
    "service_address": "Av. Principal, Casa 10",
    "service_city": "Caracas",
    "recurring": true,
    "service_start_date": "2024-01-01"
  },
  "available_financing_plans": [
    {
      "plan_type": "monthly",
      "installments": 1,
      "amount_per_installment": 20.00,
      "total_amount": 20.00,
      "description": "Pago mensual"
    },
    {
      "plan_type": "quarterly",
      "installments": 3,
      "amount_per_installment": 20.00,
      "total_amount": 60.00,
      "description": "Pago trimestral (3 cuotas)"
    },
    {
      "plan_type": "semiannual",
      "installments": 6,
      "amount_per_installment": 20.00,
      "total_amount": 120.00,
      "description": "Pago semestral (6 cuotas)"
    },
    {
      "plan_type": "annual",
      "installments": 12,
      "amount_per_installment": 20.00,
      "total_amount": 240.00,
      "description": "Pago anual (12 cuotas)"
    }
  ]
}
```

> 📝 **Catálogo de planes disponibles (ejemplos reales):**
>
> | SKU (default_code) | Nombre | Precio mensual |
> |---|---|---|
> | `INT-500MB` | Plan Internet 500MB | según tarifa |
> | `INT-1GB` | Plan Internet 1GB | según tarifa |
> | `INT-1-5GB` | Plan Internet 1.5GB | según tarifa |
> | `INT-2GB` | Plan Internet 2GB | según tarifa |
>
> El precio real se lee siempre desde `product.product.list_price` en Odoo (fuente de verdad).

---

### 4. Confirmar/Registrar Orden de Financiamiento (Cashea notifica a Odoo)

```http
POST /api/cashea/v1/financing-orders
```

**Payload enviado por Cashea:**
```json
{
  "cashea_order_id": "CASHEA-ORD-789",
  "customer_id": "RES-PARTNER-456",
  "contract_id": "CONT-2024-001",
  "plan_type": "annual",
  "total_amount": 300.00,
  "currency": "USD",
  "installments": [
    {
      "number": 1,
      "due_date": "2024-02-01",
      "amount": 25.00
    },
    {
      "number": 2,
      "due_date": "2024-03-01",
      "amount": 25.00
    }
  ],
  "approved_at": "2024-01-15T10:30:00Z",
  "credit_limit_used": 300.00,
  "cashea_metadata": {
    "customer_cashea_id": "CASH-CLI-999",
    "financing_product_code": "SERV-INT"
  }
}
```

---

### 5. Consultar Estado de un Plan de Pago

```http
GET /api/cashea/v1/financing-orders/{cashea_order_id}/status
```

**Response:**
```json
{
  "cashea_order_id": "CASHEA-ORD-789",
  "odoo_plan_id": 42,
  "state": "in_progress",
  "installments_paid": 3,
  "installments_total": 12,
  "amount_paid": 75.00,
  "amount_pending": 225.00,
  "next_installment": {
    "number": 4,
    "due_date": "2024-05-01",
    "amount": 25.00
  },
  "is_overdue": false,
  "days_overdue": 0
}
```

---

## 📬 Webhooks desde Cashea hacia Odoo

Cashea debe llamar estos endpoints cuando ocurran eventos importantes.

### Endpoint Receptor de Webhooks en Odoo

```http
POST /api/cashea/v1/webhooks/events
```

### Tipos de Eventos

#### Pago de Cuota Exitoso
```json
{
  "event_type": "installment.paid",
  "event_id": "EVT-001",
  "timestamp": "2024-02-01T14:23:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "data": {
    "installment_number": 1,
    "paid_amount": 25.00,
    "paid_date": "2024-02-01",
    "transaction_id": "TXN-12345",
    "payment_method": "card"
  }
}
```

#### Cuota Vencida (Cliente en Mora)
```json
{
  "event_type": "installment.overdue",
  "event_id": "EVT-002",
  "timestamp": "2024-02-02T00:00:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "data": {
    "installment_number": 1,
    "due_date": "2024-02-01",
    "amount_due": 25.00,
    "days_overdue": 1,
    "late_fee": 1.25
  }
}
```

#### Financiamiento Completado
```json
{
  "event_type": "financing.completed",
  "event_id": "EVT-003",
  "timestamp": "2024-01-15T16:00:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "data": {
    "total_paid": 300.00,
    "completed_date": "2025-01-01",
    "all_installments_count": 12
  }
}
```

#### Financiamiento Cancelado / Rechazado
```json
{
  "event_type": "financing.cancelled",
  "event_id": "EVT-004",
  "timestamp": "2024-01-15T10:00:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "data": {
    "reason": "credit_limit_exceeded",
    "reason_display": "Límite de crédito insuficiente",
    "cancelled_at": "2024-01-15T10:00:00Z"
  }
}
```

---

## 📤 Webhooks desde Odoo hacia Cashea

Odoo debe notificar a Cashea cuando el servicio cambia de estado.

### Evento: Servicio Suspendido

```json
{
  "event_type": "service.suspended",
  "cashea_order_id": "CASHEA-ORD-789",
  "contract_id": "CONT-2024-001",
  "reason": "non_payment",
  "suspended_at": "2024-03-05T08:00:00Z"
}
```

### Evento: Servicio Reactivado

```json
{
  "event_type": "service.reactivated",
  "cashea_order_id": "CASHEA-ORD-789",
  "contract_id": "CONT-2024-001",
  "reactivated_at": "2024-03-10T10:00:00Z"
}
```

### Evento: Contrato Cancelado

```json
{
  "event_type": "service.cancelled",
  "cashea_order_id": "CASHEA-ORD-789",
  "contract_id": "CONT-2024-001",
  "reason": "customer_request",
  "cancelled_at": "2024-03-15T12:00:00Z",
  "amount_pending": 200.00
}
```

---

## 🔄 Flujo Completo de la Integración

```
CLIENTE         ODOO                    CASHEA APP
   │               │                        │
   │─ Elige pago ─▶│                        │
   │  con Cashea   │                        │
   │               │─ GET contract info ───▶│
   │               │◀─ Retorna datos ───────│
   │               │                        │
   │               │─ Redirect / Deep-link ─────────────────▶│
   │               │  (con token de sesión)                  │
   │               │                                         │
   │               │           Cliente selecciona plan       │
   │               │           y autoriza en Cashea App      │
   │               │                                         │
   │               │◀── Webhook: financing.approved ─────────│
   │               │    (con schedule de cuotas)             │
   │               │                                         │
   │  Crea Plan    │                                         │
   │  de Pago en   │                                         │
   │  Odoo ────────│                                         │
   │               │                                         │
   │  (cada cuota) │◀── Webhook: installment.paid ───────────│
   │               │                                         │
   │               │  Marca cuota como                       │
   │               │  pagada en Odoo                         │
   │               │                                         │
   │  (si mora)    │◀── Webhook: installment.overdue ────────│
   │               │                                         │
   │               │  Actualiza estado                       │
   │               │  del cliente a "mora"                   │
   │               │                                         │
   │  (al final)   │◀── Webhook: financing.completed ────────│
   │               │                                         │
   │               │  Cierra plan, marca                     │
   │               │  contrato como pagado                   │
```

---

## 🚦 Estados y Ciclo de Vida del Pago

### Estados del Plan (`cashea.payment.plan`)

```
pending_approval  ──▶  active  ──▶  completed
       │                  │
       ▼                  ▼
   rejected           overdue ──▶  cancelled
```

| Estado | Descripción |
|---|---|
| `pending_approval` | Orden enviada a Cashea, esperando aprobación |
| `active` | Financiamiento aprobado, cuotas en curso y al día |
| `overdue` | Cliente tiene una o más cuotas vencidas (en mora) |
| `completed` | Todas las cuotas pagadas exitosamente |
| `rejected` | Cashea rechazó el financiamiento |
| `cancelled` | Cancelado por el cliente o la empresa |

### Estados de la Cuota (`cashea.installment`)

| Estado | Descripción |
|---|---|
| `pending` | Cuota pendiente (no vencida aún) |
| `paid` | Cuota pagada completamente |
| `overdue` | Cuota vencida sin pagar |
| `partially_paid` | Cuota pagada parcialmente |

---

## 🔐 Seguridad y Autenticación

### Para la API REST

1. **API Key + HMAC:** Cashea firma cada request con HMAC-SHA256 usando un secreto compartido. Odoo valida la firma antes de procesar.
2. **IP Whitelist:** Bloquear todos los requests que no vengan de los IPs de Cashea.
3. **Rate Limiting:** Máximo 100 requests/minuto por cliente.

### Para Webhooks

```python
# Validación de firma del webhook en Odoo
import hmac
import hashlib

def verify_cashea_webhook(payload_bytes, signature_header, secret):
    expected = hmac.new(
        secret.encode(), payload_bytes, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature_header)
```

### Para llamadas de Odoo a Cashea

- OAuth 2.0 Client Credentials Flow
- Token con expiración de 1 hora
- Renovación automática con refresh token

---

## ⚠️ Consideraciones Especiales: Servicio vs Producto

Cashea está orientada a productos físicos. La ventaja aquí es que **Odoo ya tiene los planes de internet modelados como `product.product` de tipo `service`** (desde 500MB hasta 2GB), lo que reduce el trabajo de adaptación.

### 1. Los Productos Ya Existen en Odoo ✅
- Cada plan de internet es un `product.product` con `type = 'service'` en Odoo
- Ya tienen nombre, descripción, precio (`list_price`) y código interno (`default_code`)
- La API simplemente **lee y serializa** esos productos al formato que Cashea espera
- **Usar `default_code` como SKU** para Cashea (ej: `INT-500MB`, `INT-1GB`, `INT-2GB`)
- Asegurarse de que cada producto tenga una **imagen** (`image_1920`) cargada en Odoo

### 2. Mapeo Odoo → Cashea

| Campo Odoo (`product.product`) | Campo Cashea | Notas |
|---|---|---|
| `default_code` | `sku` | Código interno del producto |
| `name` | `name` | Nombre del plan |
| `description_sale` | `description` | Descripción comercial |
| `list_price` | `pricing.monthly_price` | Precio mensual (fuente de verdad) |
| `type = 'service'` | `type = 'SERVICE'` | Ya correcto, no requiere cambio |
| `image_1920` | `images[0]` | URL de imagen del producto |
| `categ_id.name` | `category` | Categoría del producto |

### 3. Sin Entrega Física
- Marcar `"requires_shipping": false` en la respuesta de la API
- El "delivery" es la activación del servicio → reportar `"fulfilled_at"` con la fecha de activación del contrato

### 4. Servicio Recurrente
- Aclarar que el financiamiento es por **N meses de servicio**, no por un artículo único
- Cada cuota de Cashea = 1 mes de internet prepagado
- Si el cliente cancela el contrato a mitad, coordinar con Cashea el manejo del saldo pendiente

### 5. Sin Devoluciones Tradicionales
- Definir política de cancelación clara (ej: no hay reembolso si ya se usó el mes)
- Enviar esta política en el campo `"return_policy"` del producto

### 6. Descripción Sugerida por Plan (a cargar en `description_sale` de Odoo)

```
Plan Internet 500MB — Pague su servicio en cómodas cuotas con Cashea.
Velocidad garantizada: 500MB. Tecnología fibra óptica (FTTH).
Cada cuota equivale a 1 mes de servicio activo. Sin envío físico.
```

```
Plan Internet 2GB — Pague su servicio en cómodas cuotas con Cashea.
Velocidad garantizada: 2GB. Tecnología fibra óptica (FTTH).
Cada cuota equivale a 1 mes de servicio activo. Sin envío físico.
```

---

## 🗄️ Modelo de Datos en Odoo (Resumen)

```
res.partner (Cliente)
    │
    └── cashea.payment.plan (Plan de Pago)  [One2many]
            │
            ├── product_id → product.product  ← Plan de internet (500MB/1GB/2GB...)
            │       └── type = 'service'       ← Ya definido en Odoo ✅
            │       └── default_code           ← Usado como SKU para Cashea
            │       └── list_price             ← Precio mensual (fuente de verdad)
            │
            ├── contract_id → internet.contract (Contrato del cliente)
            ├── cashea_order_id (ID del financiamiento en Cashea)
            ├── plan_type (monthly/quarterly/semiannual/annual)
            ├── state (pending_approval/active/overdue/completed/...)
            ├── total_amount
            ├── installment_count
            ├── installment_amount
            ├── first_installment_date
            ├── last_installment_date
            ├── approved_at
            ├── completed_at
            │
            └── cashea.installment (Cuotas)  [One2many]
                    ├── installment_number
                    ├── due_date
                    ├── amount
                    ├── state (pending/paid/overdue/partially_paid)
                    ├── paid_date
                    ├── paid_amount
                    ├── late_fee
                    └── cashea_transaction_id
```

---

## ✅ Checklist de Implementación

### Fase 1: Módulo Odoo Base
- [ ] Crear modelos `cashea.payment.plan` y `cashea.installment`
- [ ] Extender el modelo de contrato de internet
- [ ] Crear vistas Odoo (formulario, lista, kanban)
- [ ] Configurar acciones y menús
- [ ] Crear grupos de seguridad (admin, usuario de cobros)

### Fase 2: API REST (Odoo expone)
- [ ] Implementar controlador HTTP en Odoo (`/api/cashea/v1/`)
- [ ] Endpoint: GET cliente
- [ ] Endpoint: GET contratos del cliente
- [ ] Endpoint: GET info de contrato como "producto"
- [ ] Endpoint: POST registrar orden de financiamiento
- [ ] Endpoint: GET estado del plan
- [ ] Agregar autenticación por API Key + HMAC

### Fase 3: Webhooks (Cashea → Odoo)
- [ ] Implementar receptor de webhooks en Odoo
- [ ] Handler para `installment.paid`
- [ ] Handler para `installment.overdue`
- [ ] Handler para `financing.completed`
- [ ] Handler para `financing.cancelled`
- [ ] Validación de firma HMAC en todos los webhooks
- [ ] Sistema de reintentos y log de eventos

### Fase 4: Webhooks (Odoo → Cashea)
- [ ] Implementar cliente HTTP en Odoo para llamar a Cashea
- [ ] Notificar suspensión de servicio
- [ ] Notificar reactivación de servicio
- [ ] Notificar cancelación de contrato

### Fase 5: Lógica de Negocio
- [ ] Automatización: detectar cuotas vencidas (cron diario)
- [ ] Automatización: cambiar estado del contrato si hay mora
- [ ] Regla: un cliente puede tener N planes Cashea activos según crédito
- [ ] Dashboard / Reporte de planes Cashea activos, en mora y completados
- [ ] Alertas automáticas al equipo de cobros si un cliente entra en mora

### Fase 6: Pruebas y Documentación
- [ ] Entorno sandbox/pruebas con Cashea
- [ ] Tests de todos los endpoints REST
- [ ] Tests de todos los webhooks
- [ ] Documentación de la API (Swagger/OpenAPI)
- [ ] Manual de usuario para el equipo de cobros

---

## 📌 Información Adicional Recomendada a Registrar

- **Historial de cambios de estado** del plan (log con timestamps)
- **Comunicaciones enviadas al cliente** (SMS/email recordatorio de cuota)
- **IP y User-Agent** de cada webhook recibido de Cashea (auditoría)
- **Token de sesión Cashea** usado para el redirect (con expiración)
- **Número de reintentos** de webhooks fallidos
- **Monto de mora acumulada** por plan
- **Tasa de cambio** si el sistema maneja multidivisa (USD/VES)

---

*Documento generado: 2026-10-08 | Versión: 1.0*
