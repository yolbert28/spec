# 🛠️ Integración Cashea ↔ Odoo — Stack Tecnológico & Portal Externo

> **Documento complementario** a los planes v1 y v2.
> Cubre las tecnologías recomendadas para implementar el módulo en Odoo y plantea el escenario donde el portal web del cliente es **externo** a Odoo (no el portal nativo de Odoo).

---

## 📋 Tabla de Contenidos

1. [Stack Tecnológico en Odoo](#stack-tecnológico-en-odoo)
2. [Estructura del Módulo Odoo](#estructura-del-módulo-odoo)
3. [Escenario A: Portal Nativo de Odoo](#escenario-a-portal-nativo-de-odoo)
4. [Escenario B: Portal Web Externo](#escenario-b-portal-web-externo)
5. [Comparativa: Portal Odoo vs Portal Externo](#comparativa-portal-odoo-vs-portal-externo)
6. [API REST de Odoo para el Portal Externo](#api-rest-de-odoo-para-el-portal-externo)
7. [Autenticación entre Portal Externo y Odoo](#autenticación-entre-portal-externo-y-odoo)
8. [Diagrama de Arquitectura Completa](#diagrama-de-arquitectura-completa)
9. [Tecnologías Recomendadas por Capa](#tecnologías-recomendadas-por-capa)
10. [Consideraciones de Despliegue](#consideraciones-de-despliegue)

---

## 🧰 Stack Tecnológico en Odoo

### Lenguaje Base

| Componente | Tecnología | Versión recomendada |
|---|---|---|
| Backend / Modelos | **Python** | 3.10+ |
| ORM | **Odoo ORM** (nativo) | Odoo 16 / 17 |
| Templates web (portal nativo) | **QWeb** (XML) | Nativo Odoo |
| Vistas backend | **XML** (Odoo views) | Nativo Odoo |
| Lógica en cliente (backend) | **JavaScript / OWL** | Odoo Web Library |

---

### Librería HTTP (llamadas salientes a Cashea)

Para que Odoo haga el **push** del payload a Cashea:

```python
# Opción recomendada: requests (ya disponible en Odoo)
import requests
import json

# NO usar urllib directamente; requests es más limpio y robusto
response = requests.post(
    url="https://api.cashea.app/v1/financing-orders",
    json=payload,
    headers={
        "Authorization": f"Bearer {token}",
        "X-Idempotency-Key": str(uuid.uuid4()),
        "Content-Type": "application/json",
    },
    timeout=30
)
```

| Librería | Uso | Disponible en Odoo |
|---|---|---|
| `requests` | Llamadas HTTP salientes a Cashea | ✅ Sí |
| `json` | Serialización del payload | ✅ Sí (stdlib) |
| `uuid` | Generación de `order_reference` e `Idempotency-Key` | ✅ Sí (stdlib) |
| `hmac` + `hashlib` | Validación de firma de webhooks | ✅ Sí (stdlib) |
| `datetime` | Cálculo del calendario de cuotas | ✅ Sí (stdlib) |
| `logging` | Log interno de errores y eventos | ✅ Sí (stdlib) |

---

### Controlador HTTP (Webhooks entrantes desde Cashea)

Odoo usa su propio sistema de routing HTTP basado en **Werkzeug**:

```python
from odoo import http
from odoo.http import request
import json
import hmac
import hashlib

class CasheaWebhookController(http.Controller):

    @http.route(
        '/api/cashea/v1/webhooks/events',
        type='json',
        auth='none',           # Sin autenticación de sesión Odoo
        methods=['POST'],
        csrf=False             # CSRF desactivado para webhooks externos
    )
    def receive_event(self, **kwargs):
        # 1. Leer el payload raw para validar la firma HMAC
        payload_bytes = request.httprequest.get_data()
        signature = request.httprequest.headers.get('X-Cashea-Signature', '')

        # 2. Validar firma
        secret = request.env['ir.config_parameter'].sudo().get_param('cashea.webhook_secret')
        if not self._verify_signature(payload_bytes, signature, secret):
            return {'error': 'Invalid signature'}, 401

        # 3. Procesar el evento de forma asíncrona
        data = json.loads(payload_bytes)
        request.env['cashea.webhook.log'].sudo().create_and_dispatch(data)

        # 4. Responder 200 INMEDIATAMENTE (Cashea no debe esperar)
        return {'status': 'received'}

    def _verify_signature(self, payload_bytes, signature_header, secret):
        expected = hmac.new(
            secret.encode('utf-8'), payload_bytes, hashlib.sha256
        ).hexdigest()
        return hmac.compare_digest(f"sha256={expected}", signature_header)
```

---

### Tareas Programadas (Crons)

Para detección de mora y planes completados, usar el **cron nativo de Odoo**:

```xml
<!-- En el archivo de datos del módulo -->
<record id="ir_cron_cashea_check_overdue" model="ir.cron">
    <field name="name">Cashea: Verificar cuotas vencidas</field>
    <field name="model_id" ref="model_cashea_installment"/>
    <field name="state">code</field>
    <field name="code">model.check_overdue_installments()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">days</field>
    <field name="numbercall">-1</field>
    <field name="active">True</field>
</record>
```

---

### Procesamiento Asíncrono de Webhooks

Los webhooks deben procesarse fuera del request HTTP para responder rápido a Cashea. Opciones:

| Opción | Descripción | Recomendación |
|---|---|---|
| **Odoo Queue Job** (`queue_job`) | Módulo OCA que agrega una cola de jobs asíncrona a Odoo | ✅ Recomendado |
| **`ir.cron` diferido** | Crear un cron de un solo disparo para procesar el evento | ⚠️ Funciona, menos elegante |
| **Thread en Python** | Lanzar un thread desde el controlador | ❌ No recomendado en Odoo |
| **Celery externo** | Cola de tareas independiente | ⚠️ Solo si ya lo usan |

```bash
# Instalar queue_job (módulo OCA)
pip install odoo14-addon-queue-job  # ajustar versión según tu Odoo
```

---

## 📁 Estructura del Módulo Odoo

```
addons/
└── cashea_integration/
    ├── __manifest__.py
    ├── __init__.py
    │
    ├── models/
    │   ├── __init__.py
    │   ├── cashea_payment_plan.py      # Modelo principal del plan
    │   ├── cashea_installment.py       # Cuotas individuales
    │   ├── cashea_webhook_log.py       # Log de webhooks recibidos
    │   ├── res_partner.py              # Extensión del cliente
    │   └── internet_contract.py        # Extensión del contrato
    │
    ├── controllers/
    │   ├── __init__.py
    │   ├── webhook.py                  # Receptor de webhooks de Cashea
    │   └── portal.py                  # Vistas portal (si se usa portal Odoo)
    │
    ├── services/
    │   ├── __init__.py
    │   ├── cashea_client.py            # Cliente HTTP para llamar a Cashea
    │   ├── payload_builder.py          # Constructor del JSON push
    │   └── schedule_calculator.py      # Calcula el calendario de cuotas
    │
    ├── wizard/
    │   ├── __init__.py
    │   └── cashea_initiate_wizard.py   # Wizard para iniciar financiamiento desde Odoo
    │
    ├── views/
    │   ├── cashea_payment_plan_views.xml
    │   ├── cashea_installment_views.xml
    │   ├── cashea_webhook_log_views.xml
    │   ├── res_partner_views.xml
    │   ├── menu.xml
    │   └── portal_templates.xml        # Templates QWeb (portal Odoo)
    │
    ├── security/
    │   ├── ir.model.access.csv
    │   └── cashea_security.xml
    │
    ├── data/
    │   ├── cron.xml                    # Tareas programadas
    │   └── ir_config_parameter.xml     # Parámetros de configuración
    │
    ├── static/
    │   └── src/
    │       ├── js/
    │       │   └── cashea_widget.js    # Widget OWL (si se necesita)
    │       └── css/
    │           └── cashea.css
    │
    └── tests/
        ├── __init__.py
        ├── test_payload_builder.py
        ├── test_webhook_handler.py
        └── test_cashea_client.py
```

---

## 🖥️ Escenario A: Portal Nativo de Odoo

En este escenario, el cliente accede al portal integrado de Odoo (`/web/portal`).

### Cómo funciona

1. El cliente inicia sesión en `https://tuempresa.com/web/login`
2. Ve sus contratos en una vista portal (QWeb + `http.Controller`)
3. Selecciona el contrato y el plan de cuotas desde la misma interfaz
4. Odoo construye el payload y hace el push a Cashea sin salir del sistema

### Tecnologías usadas

| Capa | Tecnología |
|---|---|
| Templates HTML | **QWeb** (XML Odoo) |
| Routing | `http.Controller` + `@http.route` |
| Autenticación del cliente | Sesión Odoo nativa (`auth='user'`) |
| Estilos | Bootstrap (incluido en Odoo) + CSS propio |
| JavaScript (UI dinámica) | **OWL** (Odoo Web Library) o vanilla JS |

### Ejemplo de Controlador Portal

```python
from odoo import http
from odoo.http import request

class CasheaPortalController(http.Controller):

    @http.route('/cashea/mis-contratos', type='http', auth='user', website=True)
    def mis_contratos(self, **kwargs):
        """Muestra los contratos activos del cliente y sus planes Cashea"""
        partner = request.env.user.partner_id
        contratos = request.env['internet.contract'].search([
            ('partner_id', '=', partner.id),
            ('state', '=', 'active')
        ])
        return request.render('cashea_integration.portal_contratos', {
            'contratos': contratos,
            'page_name': 'Pagar con Cashea',
        })

    @http.route('/cashea/iniciar-pago', type='json', auth='user', methods=['POST'])
    def iniciar_pago(self, contract_id, plan_type, **kwargs):
        """Construye el payload y hace el push a Cashea"""
        contract = request.env['internet.contract'].browse(contract_id)
        # Verificar que el contrato pertenece al cliente
        if contract.partner_id != request.env.user.partner_id:
            return {'error': 'Acceso denegado'}

        plan = request.env['cashea.payment.plan'].sudo().create_and_push(
            contract=contract,
            plan_type=plan_type
        )
        return {'status': plan.state, 'cashea_order_id': plan.cashea_order_id}
```

---

## 🌐 Escenario B: Portal Web Externo

En este escenario, la empresa tiene su propio sitio web (React, Vue, Next.js, Laravel, WordPress, etc.) completamente separado de Odoo. El portal externo necesita **comunicarse con Odoo** para obtener datos y disparar el flujo de financiamiento.

### Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│               PORTAL WEB EXTERNO                            │
│   (React / Vue / Next.js / Laravel / cualquier stack)       │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  1. Autenticar al cliente (JWT propio del portal)   │  │
│   │  2. GET /odoo-api/contratos  →  muestra contratos   │  │
│   │  3. Cliente selecciona plan y cuotas                │  │
│   │  4. POST /odoo-api/cashea/iniciar  →  dispara push  │  │
│   └─────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────┘
                             │  HTTPS + API Key / JWT
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    ODOO (Backend)                           │
│                                                             │
│  Expone una REST API dedicada para el portal externo:       │
│  GET  /external-api/v1/contratos/{partner_id}               │
│  POST /external-api/v1/cashea/iniciar                       │
│  GET  /external-api/v1/cashea/estado/{plan_id}              │
│                                                             │
│  Internamente: construye el payload y hace push a Cashea    │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
                     ┌──────────────┐
                     │    CASHEA    │
                     └──────────────┘
                             │
                             │ Webhooks
                             ▼
             ┌───────────────────────────┐
             │  Odoo Webhook Handler     │
             │  /api/cashea/v1/webhooks  │
             └───────────────────────────┘
```

---

## 🔌 API REST de Odoo para el Portal Externo

Odoo necesita exponer endpoints que el portal externo consumirá. Estos son **distintos** a los webhooks de Cashea.

### Endpoint 1: Obtener Contratos del Cliente

```http
GET /external-api/v1/contratos
Authorization: Bearer <portal_jwt_token>
```

```json
{
  "partner_id": 456,
  "partner_name": "Juan Pérez",
  "contratos": [
    {
      "id": 12,
      "ref": "CONT-2024-001",
      "plan_nombre": "Plan Internet 1GB",
      "plan_sku": "INT-1GB",
      "cuota_mensual": 20.00,
      "moneda": "USD",
      "estado": "active",
      "tiene_cashea_activo": false,
      "cashea_planes": []
    },
    {
      "id": 15,
      "ref": "CONT-2024-002",
      "plan_nombre": "Plan Internet 500MB",
      "plan_sku": "INT-500MB",
      "cuota_mensual": 12.00,
      "moneda": "USD",
      "estado": "active",
      "tiene_cashea_activo": true,
      "cashea_planes": [
        {
          "cashea_order_id": "CASHEA-ORD-100",
          "plan_type": "annual",
          "state": "active",
          "cuotas_pagadas": 4,
          "cuotas_total": 12,
          "proxima_cuota": "2024-06-01"
        }
      ]
    }
  ]
}
```

---

### Endpoint 2: Iniciar Financiamiento Cashea

El portal externo llama a este endpoint cuando el cliente confirma el plan. Odoo se encarga de todo lo demás.

```http
POST /external-api/v1/cashea/iniciar
Authorization: Bearer <portal_jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "contrato_id": 12,
  "plan_type": "annual"
}
```

**Response (éxito):**
```json
{
  "status": "approved",
  "cashea_order_id": "CASHEA-ORD-789",
  "odoo_plan_id": 42,
  "mensaje": "Financiamiento aprobado exitosamente.",
  "resumen": {
    "cuotas": 12,
    "monto_por_cuota": 20.00,
    "primer_pago": "2024-02-01",
    "ultimo_pago": "2025-01-01"
  }
}
```

**Response (rechazo):**
```json
{
  "status": "rejected",
  "mensaje": "El monto solicitado supera tu límite de crédito disponible en Cashea.",
  "credito_disponible": 100.00
}
```

**Response (pendiente confirmación en app Cashea):**
```json
{
  "status": "pending_customer_confirmation",
  "cashea_order_id": "CASHEA-ORD-789",
  "cashea_url": "https://app.cashea.com/confirm/CASHEA-ORD-789",
  "mensaje": "Confirma el pago desde la app de Cashea.",
  "expira_en": "2024-01-15T11:00:00Z"
}
```

---

### Endpoint 3: Consultar Estado de un Plan

```http
GET /external-api/v1/cashea/estado/{odoo_plan_id}
Authorization: Bearer <portal_jwt_token>
```

```json
{
  "plan_id": 42,
  "cashea_order_id": "CASHEA-ORD-789",
  "state": "active",
  "state_display": "Al día",
  "cuotas_pagadas": 3,
  "cuotas_total": 12,
  "monto_pagado": 60.00,
  "monto_pendiente": 180.00,
  "proxima_cuota": {
    "numero": 4,
    "fecha_vencimiento": "2024-05-01",
    "monto": 20.00
  },
  "en_mora": false,
  "dias_mora": 0,
  "cuotas": [
    { "numero": 1, "fecha": "2024-02-01", "monto": 20.00, "estado": "paid",    "fecha_pago": "2024-02-01" },
    { "numero": 2, "fecha": "2024-03-01", "monto": 20.00, "estado": "paid",    "fecha_pago": "2024-03-01" },
    { "numero": 3, "fecha": "2024-04-01", "monto": 20.00, "estado": "paid",    "fecha_pago": "2024-04-02" },
    { "numero": 4, "fecha": "2024-05-01", "monto": 20.00, "estado": "pending", "fecha_pago": null }
  ]
}
```

---

## 🔐 Autenticación entre Portal Externo y Odoo

El portal externo no usa sesiones de Odoo. Se necesita un mecanismo propio de autenticación.

### Opción 1: API Key por cliente (recomendada para implementaciones simples)

```
Portal Externo                          Odoo
     │                                    │
     │── Solicita contratos del cliente ──▶│
     │   Header: X-Portal-Api-Key: <key>   │
     │   Header: X-Partner-Id: 456         │
     │                                    │
     │   Odoo valida la key contra         │
     │   ir.config_parameter o un modelo  │
     │   propio de claves                  │
     │◀── Retorna contratos ───────────────│
```

```python
# En Odoo: validación de la API Key del portal
class ExternalApiController(http.Controller):

    def _validate_portal_key(self):
        api_key = request.httprequest.headers.get('X-Portal-Api-Key')
        expected = request.env['ir.config_parameter'].sudo().get_param(
            'cashea.portal_api_key'
        )
        if not api_key or not hmac.compare_digest(api_key, expected):
            return False
        return True
```

### Opción 2: JWT (recomendada para portales con autenticación de usuario)

El portal externo emite un **JWT** cuando el cliente inicia sesión. Odoo valida ese JWT en cada petición.

```
Portal Externo                          Odoo
     │                                    │
     │── POST /auth/login ────────────────▶│  (o login en el portal externo)
     │◀── JWT token ───────────────────────│
     │                                    │
     │── GET /external-api/v1/contratos ──▶│
     │   Authorization: Bearer <jwt>       │
     │                                    │
     │   Odoo decodifica el JWT y          │
     │   verifica el partner_id            │
     │◀── Contratos del cliente ───────────│
```

```python
# Librería recomendada para validar JWT en Odoo
import jwt  # pip install PyJWT

def _validate_jwt(self, token: str) -> dict:
    secret = request.env['ir.config_parameter'].sudo().get_param(
        'cashea.jwt_secret'
    )
    try:
        payload = jwt.decode(token, secret, algorithms=["HS256"])
        return payload  # contiene partner_id, exp, etc.
    except jwt.ExpiredSignatureError:
        raise Exception("Token expirado")
    except jwt.InvalidTokenError:
        raise Exception("Token inválido")
```

### Opción 3: OAuth 2.0 con Odoo como Authorization Server

Solo recomendada si ya usan Odoo como IdP (Identity Provider). Requiere configuración adicional.

---

## ⚖️ Comparativa: Portal Odoo vs Portal Externo

| Criterio | Portal Nativo Odoo | Portal Externo |
|---|---|---|
| **Velocidad de desarrollo** | ✅ Más rápido (todo integrado) | ⚠️ Más lento (hay que crear la API) |
| **Mantenimiento** | ✅ Un solo sistema | ⚠️ Dos sistemas sincronizados |
| **Flexibilidad de diseño** | ⚠️ Limitado al look de Odoo | ✅ Total libertad de diseño |
| **Autenticación** | ✅ La nativa de Odoo | ⚠️ Requiere diseñar auth propia |
| **Escalabilidad frontend** | ⚠️ Limitada | ✅ Alta (SPA, CDN, etc.) |
| **Separación de responsabilidades** | ⚠️ Todo mezclado en Odoo | ✅ Frontend y backend separados |
| **Integración con otras APIs** | ⚠️ Más difícil desde Odoo | ✅ Más sencillo desde el portal |
| **Experiencia de usuario (UX)** | ⚠️ UX estándar Odoo | ✅ UX personalizada y moderna |
| **Complejidad de la integración** | ✅ Baja | ⚠️ Media-Alta |
| **Costo de implementación** | ✅ Menor | ⚠️ Mayor |

> 💡 **Recomendación:** Si el portal externo ya existe o está planificado, invertir en la API de Odoo vale la pena a largo plazo porque le da independencia total al frontend. Si solo necesitan que el cliente pueda pagar con Cashea y no hay portal externo, el portal nativo de Odoo es suficiente.

---

## 🏗️ Diagrama de Arquitectura Completa (Portal Externo)

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CLIENTE FINAL                                │
│                    (Navegador / App móvil)                           │
└─────────────────────────────┬────────────────────────────────────────┘
                              │ HTTPS
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    PORTAL WEB EXTERNO                                │
│              (React / Vue / Next.js / Laravel / etc.)                │
│                                                                      │
│  • Autenticación propia del cliente (email + pass, OAuth social)     │
│  • Muestra contratos (datos de la API de Odoo)                       │
│  • Muestra planes Cashea disponibles y estado de cuotas              │
│  • Envía solicitud de financiamiento a Odoo                          │
│  • Muestra confirmación o redirección a app Cashea                   │
└─────────────────────────────┬────────────────────────────────────────┘
                              │ HTTPS + JWT / API Key
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         ODOO (Backend)                               │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  External API Layer (/external-api/v1/...)                  │    │
│  │  • Autenticación JWT / API Key                              │    │
│  │  • GET contratos del cliente                                │    │
│  │  • POST iniciar financiamiento → construye payload          │    │
│  │  • GET estado del plan                                      │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                        │
│  ┌──────────────────────────▼──────────────────────────────────┐    │
│  │  Cashea Service Layer                                       │    │
│  │  • payload_builder.py → ensambla el JSON                    │    │
│  │  • cashea_client.py → OAuth2 + POST a Cashea                │    │
│  │  • schedule_calculator.py → fechas de cuotas                │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                        │
│  ┌──────────────────────────▼──────────────────────────────────┐    │
│  │  Odoo ORM (Modelos)                                         │    │
│  │  • cashea.payment.plan                                      │    │
│  │  • cashea.installment                                       │    │
│  │  • cashea.webhook.log                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Webhook Handler (/api/cashea/v1/webhooks/events)           │    │
│  │  • Recibe eventos de Cashea                                 │    │
│  │  • Valida HMAC-SHA256                                       │    │
│  │  • Actualiza cuotas y estados en Odoo                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Cron Jobs                                                  │    │
│  │  • Detectar cuotas vencidas (diario)                        │    │
│  │  • Alertas al equipo de cobros                              │    │
│  │  • Notificar suspensiones a Cashea                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────┬────────────────────────────────────────┘
                              │ HTTPS + OAuth2
                              ▼
                    ┌──────────────────┐
                    │     CASHEA       │
                    │  (Financia y     │
                    │   cobra cuotas)  │
                    └─────────┬────────┘
                              │ Webhooks (HMAC firmados)
                              ▼
              ┌───────────────────────────────┐
              │  Odoo: /api/cashea/v1/webhooks│
              └───────────────────────────────┘
```

---

## 📊 Tecnologías Recomendadas por Capa

### Backend (Odoo)

| Componente | Tecnología | Notas |
|---|---|---|
| **Lenguaje** | Python 3.10+ | Base de Odoo |
| **ORM** | Odoo ORM | Modelos, campos, relaciones |
| **Routing HTTP** | Werkzeug (vía Odoo) | `@http.route` |
| **HTTP saliente** | `requests` | Push a Cashea |
| **Autenticación JWT** | `PyJWT` | Validar tokens del portal externo |
| **Colas asíncronas** | `queue_job` (OCA) | Procesar webhooks sin bloquear |
| **Crons** | `ir.cron` Odoo | Verificación diaria de mora |
| **Logs** | `logging` stdlib + `mail.thread` Odoo | Chatter + logs sistema |
| **Tests** | `unittest` + `odoo.tests.common` | Tests unitarios del módulo |
| **Configuración** | `ir.config_parameter` Odoo | Credenciales Cashea, secrets |

### Frontend (Portal Externo — si aplica)

| Componente | Tecnología | Notas |
|---|---|---|
| **Framework JS** | React / Vue / Next.js | Según el stack actual de la empresa |
| **HTTP Client** | `axios` o `fetch` nativo | Llamadas a la API de Odoo |
| **Autenticación** | JWT + refresh token | Persistencia de sesión |
| **Estilos** | CSS propio / Tailwind | Según preferencia del equipo |
| **Estado global** | Zustand / Pinia / Redux | Según el framework elegido |

### Infraestructura

| Componente | Tecnología | Notas |
|---|---|---|
| **Servidor Odoo** | Nginx + Gunicorn/uWSGI | Proxy reverso estándar |
| **Base de datos** | PostgreSQL 14+ | Requerido por Odoo |
| **SSL/TLS** | Let's Encrypt / Certificado propio | **Obligatorio** para webhooks |
| **Monitoreo** | Datadog / Sentry / Prometheus | Alertas de errores en webhooks |
| **Logs centralizados** | Graylog / ELK Stack / CloudWatch | Auditoría completa |
| **IP Whitelist Cashea** | Configurar en Nginx o firewall | Bloquear IPs no autorizadas |

---

## 🚀 Consideraciones de Despliegue

### SSL/TLS es Obligatorio

Cashea no enviará webhooks a URLs HTTP. El servidor de Odoo **debe** tener certificado válido.

```nginx
# nginx.conf — Configuración básica para Odoo + webhooks Cashea
server {
    listen 443 ssl;
    server_name tuempresa.com;

    ssl_certificate /etc/letsencrypt/live/tuempresa.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tuempresa.com/privkey.pem;

    # Ruta de webhooks de Cashea
    location /api/cashea/ {
        proxy_pass http://localhost:8069;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 30s;

        # Limitar a IPs de Cashea (completar con IPs reales)
        allow 203.0.113.10;  # IP de Cashea (ejemplo)
        deny all;
    }

    # API para el portal externo
    location /external-api/ {
        proxy_pass http://localhost:8069;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Variables de Entorno / Parámetros de Sistema en Odoo

Almacenar todos los secretos en `ir.config_parameter` (nunca en código):

| Parámetro | Descripción |
|---|---|
| `cashea.client_id` | Client ID OAuth2 de Cashea |
| `cashea.client_secret` | Client Secret OAuth2 de Cashea |
| `cashea.api_base_url` | URL base de la API de Cashea |
| `cashea.webhook_secret` | Secreto para validar firmas HMAC |
| `cashea.portal_api_key` | API Key que el portal externo usa para hablar con Odoo |
| `cashea.jwt_secret` | Secreto para firmar/validar JWTs del portal externo |
| `cashea.sandbox_mode` | `True` en desarrollo, `False` en producción |

---

## 📌 Resumen de Decisiones Clave

| Decisión | Opción A | Opción B | Recomendación |
|---|---|---|---|
| Portal web | Nativo Odoo (QWeb) | Externo (React/Vue/etc.) | Externo si ya existe; nativo si se construye desde cero |
| Autenticación portal externo | API Key simple | JWT | JWT si hay usuarios individuales; API Key si es M2M |
| Procesamiento webhooks | Síncrono | Asíncrono (`queue_job`) | **Asíncrono** siempre |
| HTTP a Cashea | `requests` | `httpx` (async) | `requests` (más simple en Odoo) |
| Tests | Manuales | `odoo.tests.common` | **Tests automatizados** obligatorios |

---

*Documento generado: 2026-10-08 | Stack Tecnológico & Portal Externo*
*Documentos relacionados: `cashea-odoo-integration.md` (v1) · `cashea-odoo-integration-v2-push.md` (v2)*
