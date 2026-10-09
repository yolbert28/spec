# 🔌 Integración Cashea ↔ Odoo — Plan v2: Modelo PUSH

> **Versión:** 2.0 — Arquitectura Push (Web empresa → Cashea)
>
> **Diferencia clave respecto a v1:** En esta versión **no es Cashea quien consulta los datos** del cliente ni del plan. Es la **página web de la empresa** quien recopila toda la información desde Odoo y la envía completa a Cashea en una sola llamada. Cashea simplemente recibe el paquete y ejecuta sus funciones de financiamiento.
>
> **Importante:** Los planes de internet ya están modelados en Odoo como **`product.product` de tipo `service`**, con planes desde **500 MB hasta 2 GB**.

---

## 📋 Tabla de Contenidos

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [Arquitectura del Modelo Push](#arquitectura-del-modelo-push)
3. [Flujo Completo de la Integración](#flujo-completo-de-la-integración)
4. [El Payload: Qué se Envía a Cashea](#el-payload-qué-se-envía-a-cashea)
5. [Respuesta de Cashea](#respuesta-de-cashea)
6. [Webhooks: Cashea notifica a Odoo](#webhooks-cashea-notifica-a-odoo)
7. [Webhooks: Odoo notifica a Cashea](#webhooks-odoo-notifica-a-cashea)
8. [Información a Rastrear en Odoo](#información-a-rastrear-en-odoo)
9. [Estados y Ciclo de Vida del Pago](#estados-y-ciclo-de-vida-del-pago)
10. [Seguridad y Autenticación](#seguridad-y-autenticación)
11. [Consideraciones Especiales (Servicio vs Producto)](#consideraciones-especiales-servicio-vs-producto)
12. [Modelo de Datos en Odoo](#modelo-de-datos-en-odoo)
13. [Checklist de Implementación](#checklist-de-implementación)

---

## 🎯 Resumen Ejecutivo

El flujo es directo y simple:

```
Cliente (web empresa) → selecciona plan + modalidad de pago
        ↓
Odoo recopila toda la información
        ↓
Web empresa envía TODO a Cashea en una sola llamada POST
        ↓
Cashea procesa y aprueba/rechaza el financiamiento
        ↓
Cashea notifica a Odoo mediante Webhooks (pagos, mora, finalización)
```

**Ventajas del modelo Push:**
- Cashea no necesita implementar lógica para consultar datos a Odoo
- Menor latencia (un solo round-trip para iniciar el financiamiento)
- Control total del formato y la experiencia desde la web de la empresa
- Cashea recibe exactamente lo que necesita, sin ambigüedades

---

## 🏗️ Arquitectura del Modelo Push

```
┌────────────────────────────────────────────────────────────────┐
│                     EMPRESA (WEB + ODOO)                       │
│                                                                │
│   ┌──────────────┐    ┌──────────────┐    ┌────────────────┐  │
│   │  Página Web  │───▶│    Odoo      │───▶│  Payload Push  │  │
│   │  (cliente    │    │  (recopila   │    │  (JSON con     │  │
│   │   elige plan │    │   datos del  │    │   TODO listo)  │  │
│   │   y cuotas)  │    │   contrato,  │    └───────┬────────┘  │
│   └──────────────┘    │   cliente y  │            │           │
│                       │   producto)  │            │           │
│                       └──────────────┘            │           │
│                                                   │ POST      │
│   ┌──────────────────────────────────────┐        │           │
│   │  Webhook Handler                     │        │           │
│   │  (recibe eventos de Cashea:          │        │           │
│   │   pagos, mora, finalización)         │        │           │
│   └──────────────────────────────────────┘        │           │
└───────────────────────────────────────────────────┼───────────┘
                                                    │
                                                    ▼
                                         ┌──────────────────┐
                                         │     CASHEA       │
                                         │                  │
                                         │  Recibe el       │
                                         │  paquete         │
                                         │  completo y      │
                                         │  ejecuta el      │
                                         │  financiamiento  │
                                         └──────────────────┘
```

> ✅ **Cashea NO necesita endpoints para consultar datos.** Solo recibe un POST con todo y responde con la decisión de financiamiento.

---

## 🔄 Flujo Completo de la Integración

```
CLIENTE (WEB)         ODOO (Backend)               CASHEA
      │                     │                          │
      │─ Inicia sesión ────▶│                          │
      │                     │                          │
      │◀─ Muestra sus ──────│                          │
      │   contratos activos │                          │
      │   (desde Odoo)      │                          │
      │                     │                          │
      │─ Selecciona:        │                          │
      │   • Contrato X      │                          │
      │   • Plan anual      │                          │
      │   (clic en Cashea)─▶│                          │
      │                     │─ Construye el Payload ──▶│
      │                     │  (cliente + contrato +   │
      │                     │   producto + cuotas)     │
      │                     │                          │
      │                     │◀─ Respuesta: approved ───│
      │                     │   + cashea_order_id      │
      │                     │   + schedule de cuotas   │
      │                     │                          │
      │                     │─ Crea Plan en Odoo       │
      │                     │  (cashea.payment.plan)   │
      │                     │                          │
      │◀─ Confirmación ─────│                          │
      │   "Financiamiento   │                          │
      │    aprobado ✅"     │                          │
      │                     │                          │
      │         ← cada mes →│                          │
      │                     │◀─ Webhook: cuota pagada ─│
      │                     │◀─ Webhook: cuota en mora─│
      │                     │◀─ Webhook: completado ───│
```

---

## 📤 El Payload: Qué se Envía a Cashea

Este es el JSON que Odoo/Web de la empresa construye y envía a Cashea en **una sola llamada POST**.

```http
POST https://api.cashea.app/v1/financing-orders
Content-Type: application/json
Authorization: Bearer <oauth_token>
X-Idempotency-Key: <uuid_v4>
```

```json
{
  "order_reference": "ORD-ODOO-2024-00456",
  "callback_url": "https://tuempresa.com/api/cashea/v1/webhooks/events",
  "redirect_url": "https://tuempresa.com/cliente/pago-confirmado",
  "cancel_url": "https://tuempresa.com/cliente/pago-cancelado",

  "customer": {
    "external_id": "RES-PARTNER-456",
    "name": "Juan Pérez",
    "id_type": "cedula",
    "id_number": "V-12345678",
    "email": "juan.perez@email.com",
    "phone": "+58 412-1234567",
    "address": {
      "street": "Av. Principal, Casa 10",
      "city": "Caracas",
      "state": "Distrito Capital",
      "country": "VE",
      "zip": "1010"
    }
  },

  "product": {
    "sku": "INT-1GB",
    "odoo_product_id": 17,
    "name": "Plan Internet 1GB",
    "description": "Servicio de internet con velocidad de 1GB. Tecnología fibra óptica (FTTH). Soporte 24/7. Cada cuota equivale a 1 mes de servicio activo. Sin envío físico.",
    "category": "INTERNET_SERVICE",
    "type": "SERVICE",
    "requires_shipping": false,
    "fulfillment_type": "DIGITAL_SERVICE",
    "return_policy": "No aplica reembolso sobre meses de servicio ya prestados.",
    "image_url": "https://tuempresa.com/web/image/product.product/17/image_1920",
    "brand": "Tu Empresa Internet",
    "unit": "SUBSCRIPTION",
    "is_recurring": true,
    "service_details": {
      "contract_id": "CONT-2024-001",
      "speed": "1GB",
      "technology": "FTTH",
      "service_address": "Av. Principal, Casa 10",
      "service_city": "Caracas",
      "service_start_date": "2024-01-01"
    }
  },

  "financing": {
    "plan_type": "annual",
    "total_amount": 240.00,
    "currency": "USD",
    "installments_count": 12,
    "amount_per_installment": 20.00,
    "first_installment_date": "2024-02-01",
    "installment_schedule": [
      { "number": 1,  "due_date": "2024-02-01", "amount": 20.00 },
      { "number": 2,  "due_date": "2024-03-01", "amount": 20.00 },
      { "number": 3,  "due_date": "2024-04-01", "amount": 20.00 },
      { "number": 4,  "due_date": "2024-05-01", "amount": 20.00 },
      { "number": 5,  "due_date": "2024-06-01", "amount": 20.00 },
      { "number": 6,  "due_date": "2024-07-01", "amount": 20.00 },
      { "number": 7,  "due_date": "2024-08-01", "amount": 20.00 },
      { "number": 8,  "due_date": "2024-09-01", "amount": 20.00 },
      { "number": 9,  "due_date": "2024-10-01", "amount": 20.00 },
      { "number": 10, "due_date": "2024-11-01", "amount": 20.00 },
      { "number": 11, "due_date": "2024-12-01", "amount": 20.00 },
      { "number": 12, "due_date": "2025-01-01", "amount": 20.00 }
    ]
  },

  "metadata": {
    "odoo_contract_id": "CONT-2024-001",
    "odoo_partner_id": 456,
    "odoo_product_id": 17,
    "source": "company_website",
    "initiated_by": "customer_self_service"
  }
}
```

### 📌 Campos Clave Explicados

| Campo | Por qué es importante |
|---|---|
| `order_reference` | ID interno de Odoo para cruzar con la respuesta de Cashea |
| `callback_url` | URL donde Cashea enviará los webhooks de eventos |
| `redirect_url` | A dónde redirigir al cliente tras aprobación en Cashea |
| `customer.external_id` | ID del `res.partner` en Odoo para relacionar respuestas |
| `product.type = "SERVICE"` | Evita que Cashea aplique lógica de envío/inventario |
| `product.requires_shipping = false` | El "delivery" es la activación del servicio |
| `product.is_recurring = true` | Cashea sabe que es un servicio mensual, no un artículo |
| `financing.installment_schedule` | Se envía el calendario completo calculado desde Odoo |
| `metadata.odoo_contract_id` | Permite que Cashea lo reenvíe en webhooks para identificar el contrato |

---

## 📥 Respuesta de Cashea

### Aprobado

```json
{
  "status": "approved",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "customer_cashea_id": "CASH-CLI-999",
  "credit_limit_used": 240.00,
  "credit_limit_remaining": 60.00,
  "approved_at": "2024-01-15T10:30:00Z",
  "cashea_redirect_url": "https://app.cashea.com/confirm/CASHEA-ORD-789",
  "financing_confirmed": {
    "total_amount": 240.00,
    "installments_count": 12,
    "amount_per_installment": 20.00,
    "first_installment_date": "2024-02-01",
    "last_installment_date": "2025-01-01"
  }
}
```

> Odoo guarda `cashea_order_id` para relacionar todos los webhooks futuros.

### Rechazado

```json
{
  "status": "rejected",
  "order_reference": "ORD-ODOO-2024-00456",
  "reason": "credit_limit_exceeded",
  "reason_display": "El monto solicitado supera el límite de crédito disponible.",
  "credit_limit_available": 100.00,
  "rejected_at": "2024-01-15T10:31:00Z"
}
```

### Pendiente de confirmación del cliente en la app Cashea

```json
{
  "status": "pending_customer_confirmation",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "cashea_redirect_url": "https://app.cashea.com/confirm/CASHEA-ORD-789",
  "expires_at": "2024-01-15T11:00:00Z"
}
```

---

## 📬 Webhooks: Cashea notifica a Odoo

Cashea llama este endpoint cada vez que ocurre un evento importante.

```http
POST https://tuempresa.com/api/cashea/v1/webhooks/events
```

### Pago de Cuota Exitoso
```json
{
  "event_type": "installment.paid",
  "event_id": "EVT-001",
  "timestamp": "2024-02-01T14:23:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "data": {
    "installment_number": 1,
    "paid_amount": 20.00,
    "paid_date": "2024-02-01",
    "transaction_id": "TXN-12345",
    "payment_method": "card"
  }
}
```

### Cuota Vencida (Cliente en Mora)
```json
{
  "event_type": "installment.overdue",
  "event_id": "EVT-002",
  "timestamp": "2024-02-02T00:00:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "data": {
    "installment_number": 1,
    "due_date": "2024-02-01",
    "amount_due": 20.00,
    "days_overdue": 1,
    "late_fee": 1.00
  }
}
```

### Financiamiento Completado
```json
{
  "event_type": "financing.completed",
  "event_id": "EVT-003",
  "timestamp": "2025-01-01T16:00:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "data": {
    "total_paid": 240.00,
    "completed_date": "2025-01-01",
    "all_installments_count": 12
  }
}
```

### Financiamiento Cancelado
```json
{
  "event_type": "financing.cancelled",
  "event_id": "EVT-004",
  "timestamp": "2024-01-15T10:00:00Z",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "data": {
    "reason": "customer_cancelled",
    "reason_display": "Cancelado por el cliente",
    "cancelled_at": "2024-01-15T10:00:00Z"
  }
}
```

---

## 📤 Webhooks: Odoo notifica a Cashea

Cuando el estado del servicio cambia, Odoo notifica a Cashea.

### Servicio Suspendido
```json
{
  "event_type": "service.suspended",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "contract_id": "CONT-2024-001",
  "reason": "non_payment",
  "suspended_at": "2024-03-05T08:00:00Z"
}
```

### Servicio Reactivado
```json
{
  "event_type": "service.reactivated",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "contract_id": "CONT-2024-001",
  "reactivated_at": "2024-03-10T10:00:00Z"
}
```

### Contrato Cancelado
```json
{
  "event_type": "service.cancelled",
  "cashea_order_id": "CASHEA-ORD-789",
  "order_reference": "ORD-ODOO-2024-00456",
  "contract_id": "CONT-2024-001",
  "reason": "customer_request",
  "cancelled_at": "2024-03-15T12:00:00Z",
  "amount_pending": 120.00
}
```

---

## 📦 Información a Rastrear en Odoo

### Modelo: `cashea.payment.plan`

| Campo | Tipo | Descripción |
|---|---|---|
| `name` | Char | Referencia interna (ej: `CASHEA-2024-00123`) |
| `partner_id` | Many2one → `res.partner` | Cliente que paga |
| `contract_id` | Many2one → Contrato de Internet | Contrato asociado |
| `product_id` | Many2one → `product.product` | Plan de internet elegido (500MB…2GB) |
| `cashea_order_id` | Char | ID del financiamiento en Cashea |
| `order_reference` | Char | ID interno de Odoo enviado en el payload |
| `plan_type` | Selection | `monthly` / `quarterly` / `semiannual` / `annual` |
| `total_amount` | Monetary | Monto total financiado |
| `installment_count` | Integer | Número de cuotas totales |
| `installment_amount` | Monetary | Monto de cada cuota |
| `first_installment_date` | Date | Fecha del primer pago |
| `last_installment_date` | Date | Fecha del último pago |
| `state` | Selection | Estado del plan (ver sección de Estados) |
| `credit_limit_used` | Monetary | Crédito Cashea consumido por este plan |
| `credit_limit_remaining` | Monetary | Crédito disponible restante (de la respuesta de Cashea) |
| `approved_at` | Datetime | Fecha de aprobación por Cashea |
| `completed_at` | Datetime | Fecha de finalización de todas las cuotas |
| `cancelled_at` | Datetime | Fecha de cancelación (si aplica) |
| `push_payload` | Text | JSON enviado a Cashea (para auditoría y depuración) |
| `cashea_response` | Text | JSON de respuesta de Cashea (para auditoría) |
| `notes` | Text | Notas internas / log de eventos |

---

### Modelo: `cashea.installment` (Cuota Individual)

| Campo | Tipo | Descripción |
|---|---|---|
| `plan_id` | Many2one → `cashea.payment.plan` | Plan al que pertenece |
| `installment_number` | Integer | Número de cuota (1, 2, 3…) |
| `due_date` | Date | Fecha de vencimiento |
| `amount` | Monetary | Monto de esta cuota |
| `state` | Selection | `pending` / `paid` / `overdue` / `partially_paid` |
| `paid_date` | Date | Fecha en que se efectuó el pago |
| `paid_amount` | Monetary | Monto efectivamente pagado |
| `cashea_transaction_id` | Char | ID de transacción en Cashea |
| `late_fee` | Monetary | Cargo por mora (si aplica) |
| `invoice_id` | Many2one → `account.move` | Factura Odoo relacionada (opcional) |

---

### Extensión del Contrato de Internet

| Campo | Tipo | Descripción |
|---|---|---|
| `cashea_plan_ids` | One2many → `cashea.payment.plan` | Todos los planes Cashea asociados |
| `active_cashea_plan_id` | Many2one (computed) | Plan Cashea activo actualmente |
| `has_cashea_financing` | Boolean (computed) | Si tiene financiamiento activo |

---

## 🚦 Estados y Ciclo de Vida del Pago

### Estados del Plan (`cashea.payment.plan`)

```
pending_approval  ──▶  active  ──▶  completed
       │                  │
       ▼                  ▼
   rejected           overdue ──▶  cancelled
```

| Estado | Descripción | Disparado por |
|---|---|---|
| `pending_approval` | POST enviado a Cashea, esperando respuesta | Envío del payload |
| `pending_customer` | Cashea espera confirmación del cliente en su app | Respuesta `pending_customer_confirmation` |
| `active` | Financiamiento aprobado, cuotas en curso y al día | Respuesta HTTP `approved` |
| `overdue` | Cliente tiene una o más cuotas vencidas | Webhook `installment.overdue` |
| `completed` | Todas las cuotas pagadas exitosamente | Webhook `financing.completed` |
| `rejected` | Cashea rechazó el financiamiento | Respuesta HTTP `rejected` |
| `cancelled` | Cancelado por cliente o empresa | Webhook `financing.cancelled` |

### Estados de la Cuota (`cashea.installment`)

| Estado | Descripción |
|---|---|
| `pending` | Cuota pendiente (no vencida aún) |
| `paid` | Cuota pagada completamente |
| `overdue` | Cuota vencida sin pagar |
| `partially_paid` | Cuota pagada parcialmente |

---

## 🔐 Seguridad y Autenticación

### Llamada Push (Odoo → Cashea)

- **OAuth 2.0 Client Credentials Flow** para obtener el token
- Token con expiración de 1 hora, renovación automática
- `X-Idempotency-Key` en cada POST para evitar doble procesamiento si hay reintentos

```http
POST https://auth.cashea.app/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=<tu_client_id>
&client_secret=<tu_client_secret>
&scope=financing:write
```

### Webhook entrante (Cashea → Odoo)

- Validar firma **HMAC-SHA256** en cada evento recibido

```python
import hmac
import hashlib

def verify_cashea_webhook(payload_bytes: bytes, signature_header: str, secret: str) -> bool:
    """
    Valida que el webhook realmente viene de Cashea.
    signature_header tiene el formato: "sha256=<hex_digest>"
    """
    expected = hmac.new(
        secret.encode('utf-8'),
        payload_bytes,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature_header)
```

- **IP Whitelist:** solo aceptar requests desde los IPs declarados por Cashea
- **Rate Limiting:** 200 webhooks/minuto máximo
- **Registro de auditoría:** guardar cada webhook recibido con timestamp e IP

---

## ⚠️ Consideraciones Especiales: Servicio vs Producto

### Los Productos Ya Existen en Odoo ✅

Cada plan de internet es un `product.product` con `type = 'service'` en Odoo. La web simplemente lee esos datos y los incluye en el payload push.

### Mapeo Odoo → Payload Cashea

| Campo Odoo (`product.product`) | Campo en el Payload | Notas |
|---|---|---|
| `default_code` | `product.sku` | Código interno del producto |
| `name` | `product.name` | Nombre del plan |
| `description_sale` | `product.description` | Descripción comercial |
| `list_price` | `financing.amount_per_installment` | Precio mensual (fuente de verdad) |
| `type = 'service'` | `product.type = 'SERVICE'` | Ya correcto en Odoo |
| `image_1920` | `product.image_url` | URL de imagen del producto |

### Catálogo de Planes (Ejemplos)

| SKU (`default_code`) | Nombre | Cuota mensual |
|---|---|---|
| `INT-500MB` | Plan Internet 500MB | según tarifa |
| `INT-1GB` | Plan Internet 1GB | según tarifa |
| `INT-1-5GB` | Plan Internet 1.5GB | según tarifa |
| `INT-2GB` | Plan Internet 2GB | según tarifa |

### Financiamiento de Múltiples Contratos

Un cliente puede pagar más de un contrato con Cashea. Cada contrato genera un **payload push independiente** y un `cashea.payment.plan` separado en Odoo. El crédito de Cashea se consume acumulativamente; la respuesta incluye `credit_limit_remaining` para saber cuánto le queda disponible.

```
Cliente con crédito Cashea: $400
  ├── Plan 1GB anual  → $240 → credit_limit_remaining: $160
  └── Plan 500MB anual → $120 → credit_limit_remaining: $40
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
            │       └── default_code           ← Usado como SKU en el payload
            │       └── list_price             ← Precio mensual (fuente de verdad)
            │
            ├── contract_id → internet.contract (Contrato del cliente)
            ├── cashea_order_id (ID del financiamiento en Cashea)
            ├── order_reference (ID Odoo enviado en el payload)
            ├── plan_type (monthly/quarterly/semiannual/annual)
            ├── state (pending_approval/active/overdue/completed/...)
            ├── total_amount
            ├── installment_count
            ├── installment_amount
            ├── first_installment_date
            ├── last_installment_date
            ├── approved_at / completed_at / cancelled_at
            ├── push_payload (JSON enviado - auditoría)
            ├── cashea_response (JSON recibido - auditoría)
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
- [ ] Agregar campo `push_payload` y `cashea_response` para auditoría
- [ ] Crear vistas Odoo (formulario, lista, kanban con estados)
- [ ] Configurar acciones y menús
- [ ] Crear grupos de seguridad (admin, usuario de cobros)

### Fase 2: Construcción del Payload (Odoo → Cashea)
- [ ] Función que lee datos del `res.partner` seleccionado
- [ ] Función que lee datos del `product.product` (plan de internet)
- [ ] Función que lee datos del contrato
- [ ] Función que calcula el `installment_schedule` según el plan elegido
- [ ] Función que genera el `order_reference` único
- [ ] Función que ensambla el JSON completo del payload
- [ ] Implementar envío HTTP con OAuth 2.0 a Cashea
- [ ] Manejo de respuesta: crear `cashea.payment.plan` con el `cashea_order_id`

### Fase 3: Webhooks Entrantes (Cashea → Odoo)
- [ ] Implementar receptor de webhooks en Odoo (`/api/cashea/v1/webhooks/events`)
- [ ] Validación de firma HMAC-SHA256 en todos los webhooks
- [ ] Handler para `installment.paid` → marcar cuota pagada
- [ ] Handler para `installment.overdue` → cambiar estado a mora
- [ ] Handler para `financing.completed` → cerrar plan
- [ ] Handler para `financing.cancelled` → cancelar plan
- [ ] Sistema de log de todos los webhooks recibidos (para auditoría)
- [ ] Responder `200 OK` de inmediato y procesar en background

### Fase 4: Webhooks Salientes (Odoo → Cashea)
- [ ] Notificar a Cashea cuando un servicio se suspende
- [ ] Notificar a Cashea cuando un servicio se reactiva
- [ ] Notificar a Cashea cuando un contrato se cancela (con monto pendiente)

### Fase 5: Lógica de Negocio
- [ ] Cron diario: detectar cuotas vencidas y actualizar estado
- [ ] Cron diario: detectar planes completados
- [ ] Regla: validar `credit_limit_remaining` antes de permitir un segundo plan Cashea
- [ ] Alertas al equipo de cobros cuando un cliente entra en mora
- [ ] Dashboard: planes activos, en mora, completados, rechazados
- [ ] Reporte: ingresos esperados por Cashea por mes

### Fase 6: Integración con la Página Web
- [ ] Endpoint que dispara el payload push al hacer clic en "Pagar con Cashea"
- [ ] Mostrar los contratos activos del cliente para que seleccione cuál financiar
- [ ] Mostrar opciones de plan (mensual, trimestral, semestral, anual)
- [ ] Mostrar desglose de cuotas antes de confirmar
- [ ] Manejar `redirect_url` tras aprobación
- [ ] Manejar `cancel_url` si el cliente cancela en Cashea

### Fase 7: Pruebas y Documentación
- [ ] Entorno sandbox con Cashea para pruebas
- [ ] Test del payload push completo (aprobación, rechazo, pending)
- [ ] Test de todos los webhooks entrantes
- [ ] Test de reintentos y manejo de errores
- [ ] Documentación interna del flujo para el equipo técnico
- [ ] Manual de usuario para el equipo de cobros

---

## 📌 Información Adicional Recomendada a Registrar

- **Historial de cambios de estado** del plan (chatter de Odoo + timestamps)
- **IP y User-Agent** de cada webhook recibido (auditoría de seguridad)
- **Tiempo de respuesta** de Cashea al payload push (monitoreo de performance)
- **Número de reintentos** del push si hay error de red
- **Tasa de cambio** si el sistema maneja multidivisa (USD/VES) al momento del push
- **Nombre del agente** que inició el financiamiento (si fue un operador, no el cliente)

---

*Documento generado: 2026-10-08 | Versión: 2.0 — Modelo Push*
*Archivo original (v1 con endpoints bidireccionales): `cashea-odoo-integration.md`*
