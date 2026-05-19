---
document_type: "Product Requirements Document"
version: "1.0.0"
status: "approved"
created: "2026-05-18"
updated: "2026-05-18"
author: "John (Product Manager)"
project_name: "Sistema de Notificaciones Omnicanal"
phase: "Fase 1 - MVP"
source_documents:
  - "_bmad-output/01-product-brief.md"
  - "_bmad-output/03-technical-spec.md"
approved_by: "ivan"
---

# PRD: Sistema de Notificaciones Omnicanal — Fase 1 (MVP)

## 1. Problema

**¿Por qué estamos construyendo esto?**

Cada microservicio del ecosistema implementa su propia lógica de conexión con proveedores de Email y SMS. El resultado es un desorden estructural:

- **Código duplicado** en N servicios que hacen lo mismo: conectar, enviar, reintentar, loguear
- **Sin control de prioridad**: un OTP de seguridad se encola detrás de una campaña de marketing masiva
- **Cero visibilidad**: nadie sabe cuántas notificaciones fallan, por qué, o cuánto tardan
- **Vendor lock-in implícito**: cambiar de proveedor requiere tocar cada microservicio que envía notificaciones
- **Sin reintentos centralizados**: si un proveedor falla temporalmente, la notificación se pierde sin más

**¿Quién siente este dolor?** Los equipos de desarrollo (pierden tiempo en boilerplate), los arquitectos (no pueden gobernar resiliencia), y los clientes finales (no reciben notificaciones críticas a tiempo).

---

## 2. Propuesta de Valor

Un **servicio centralizado de notificaciones** que los microservicios consumen mediante una API simple o un contrato de eventos. El sistema se encarga de:

- Encolar por prioridad (Alta/Media/Baja) con aislamiento físico de colas
- Abstraer proveedores externos (Email, SMS) detrás de interfaces limpias
- Reintentar con backoff exponencial y dead-letter queue
- Exponer métricas de entrega en tiempo real
- Gestionar plantillas versionadas

**El MVP se limita a Email + SMS.** Push queda para Fase 2.

---

## 3. Objetivos y Criterios de Éxito

### 3.1 KPIs del Product Brief cruzados con Métricas Prometheus

| KPI | Meta | Métrica Prometheus | Cómo se mide |
|---|---|---|---|
| **K1**: Tasa de entrega (prioridad Alta) | > 99.5% | `notification_delivery_total{priority="high", status="sent"}` / `notification_delivery_total{priority="high"}` | Ratio de sent/total para prioridad Alta en ventana de 24h |
| **K2**: Latencia de encolamiento (prioridad Alta) | p99 < 100ms | `notification_enqueue_latency_ms{priority="high"}` | Histogram quantile(0.99) entre aceptación y dispatch |
| **K3**: Reducción de código duplicado | > 80% boilerplate eliminado | No es métrica runtime — se valida por auditoría de código pre/post migración | Contar líneas de código de notificación eliminadas en microservicios migrados |
| **K4**: Tiempo para cambiar proveedor | < 1 día (configuración, sin deploy) | No es métrica runtime — se valida con ejercicio de cambio de proveedor en staging | Medir tiempo desde decisión hasta primer envío exitoso con nuevo proveedor |
| **K5**: Visibilidad de fallos | 100% trazable con correlationId | `notification_events` table: cada correlación debe tener al menos 1 evento | Query: `SELECT COUNT(DISTINCT correlation_id) FROM notifications WHERE correlation_id NOT IN (SELECT correlation_id FROM notification_events)` = 0 |
| **K6**: Adopción interna | > 50% microservicios migrados en 90 días | No es métrica runtime — se mide por registro de servicios consumidores | Contar microservicios que publican al sistema / total de microservicios que enviaban notificaciones |

### 3.2 Métricas de Operación (Prometheus)

| Métrica | Tipo | Labels | Umbral de Alerta |
|---|---|---|---|
| `notification_enqueue_total` | Counter | `priority`, `channel` | — |
| `notification_delivery_total` | Counter | `priority`, `channel`, `status` | failure rate > 5% en 5min |
| `notification_delivery_duration_ms` | Histogram | `priority`, `channel` | p99 > 5s |
| `notification_enqueue_latency_ms` | Histogram | `priority` | p99 > 100ms para high |
| `notification_retry_total` | Counter | `priority`, `channel`, `error_code` | — |
| `notification_dead_letter_total` | Counter | `priority`, `channel` | > 50 en 15min |
| `notification_queue_depth` | Gauge | `priority` | high > 1000 por 2min |
| `notification_template_render_duration_ms` | Histogram | `channel` | — |

---

## 4. Alcance del MVP

### 4.1 In-Scope

| Capacidad | Descripción | Prioridad |
|---|---|---|
| **API REST de publicación** | `POST /api/v1/notifications` con validación JSON Schema | **P0** |
| **Consulta de estado** | `GET /api/v1/notifications/:correlationId` | **P0** |
| **Canal Email** | Integración con SendGrid (1 proveedor) | **P0** |
| **Canal SMS** | Integración con Twilio (1 proveedor) | **P0** |
| **Priorización** | 3 colas físicas separadas: Alta, Media, Baja | **P0** |
| **Reintentos** | Backoff exponencial, máx 5 intentos, DLQ | **P0** |
| **Templates** | Motor Handlebars con plantillas versionadas en DB | **P0** |
| **Idempotencia** | CorrelationId como clave de deduplicación | **P0** |
| **Métricas** | Endpoint Prometheus con 8 métricas | **P0** |
| **Dead-Letter management** | Listar y reintentar desde DLQ | **P1** |
| **Health check** | `GET /api/v1/health` | **P1** |

### 4.2 Out-of-Scope (explícito)

| Elemento | Rationale |
|---|---|
| Canal Push (FCM/APNs) | Fase 2 |
| Fallback a proveedor secundario | Fase 2 |
| Circuit breaker por proveedor | Fase 2 |
| Portal self-service de templates | Fase 3 |
| Preferencias de usuario (opt-in/opt-out) | Fase 3 |
| Canal WhatsApp / OTT | Fase 4 |
| A/B testing de contenido | Fase 4 |
| SDK cliente para lenguajes | Fase 3 |

---

## 5. Historias de Usuario

### Épica E1: Publicación y Encolamiento de Notificaciones

---

#### US-01: Envío Exitoso de OTP con Prioridad Alta

**Como** desarrollador del auth-service
**Quiero** publicar una notificación OTP de prioridad alta por email o SMS
**Para que** el usuario reciba su código de verificación en menos de 100ms desde la solicitud

**Contexto técnico:**
- Canal: `email` o `sms`
- Prioridad: `high`
- Template: `otp.verification`
- El correlationId lo genera el publisher y debe ser UUID v4

**Criterios de Aceptación (Gherkin):**

```gherkin
Scenario: Envío exitoso de OTP por email con prioridad alta
  Given existe una plantilla activa con templateCode "otp.verification" para canal "email"
  And el sistema está conectado correctamente a SendGrid
  When un microservicio envía una petición POST /api/v1/notifications con el siguiente payload:
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "channel": "email",
      "priority": "high",
      "templateCode": "otp.verification",
      "recipient": {
        "type": "email",
        "value": "user@example.com"
      },
      "payload": {
        "code": "847291",
        "expiryMinutes": 10,
        "userName": "Ivan"
      },
      "metadata": {
        "sourceService": "auth-service",
        "tags": ["otp", "security"]
      }
    }
    """
  Then el sistema responde con HTTP 202 Accepted
  And el body de la respuesta contiene:
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "status": "accepted",
      "acceptedAt": "<timestamp ISO 8601>"
    }
    """
  And la notificación se encola en la cola "notifications.queue.high"
  And el email se envía a "user@example.com" a través de SendGrid en menos de 100ms desde la aceptación
  And se registra un evento "enqueued" en la tabla notification_events
  And se registra un evento "dispatched" en la tabla notification_events
  And se registra un evento "sent" en la tabla notification_events con providerMessageId de SendGrid
  And la métrica notification_enqueue_total{priority="high", channel="email"} incrementa en 1
  And la métrica notification_delivery_total{priority="high", channel="email", status="sent"} incrementa en 1
```

```gherkin
Scenario: Envío exitoso de OTP por SMS con prioridad alta
  Given existe una plantilla activa con templateCode "otp.verification" para canal "sms"
  And el sistema está conectado correctamente a Twilio
  When un microservicio envía una petición POST /api/v1/notifications con:
    """
    {
      "correlationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "channel": "sms",
      "priority": "high",
      "templateCode": "otp.verification",
      "recipient": {
        "type": "phone",
        "value": "+34612345678"
      },
      "payload": {
        "code": "847291",
        "expiryMinutes": 10,
        "userName": "Ivan"
      },
      "metadata": {
        "sourceService": "auth-service"
      }
    }
    """
  Then el sistema responde con HTTP 202 Accepted
  And el SMS se envía a "+34612345678" a través de Twilio
  And se registra un evento "sent" con providerMessageId de Twilio (formato SID)
  And la métrica notification_delivery_total{priority="high", channel="sms", status="sent"} incrementa en 1
```

```gherkin
Scenario: Latencia de encolamiento para prioridad alta cumple SLA
  Given el sistema está bajo carga normal (menos de 100 msg/min en cola alta)
  When se publica una notificación con priority "high"
  Then el tiempo entre "accepted" y "dispatched" es menor a 100ms en el percentil 99
  And la métrica notification_enqueue_latency_ms{priority="high"} refleja este valor
```

---

#### US-02: Rechazo por Validación de Schema Inválido

**Como** consumidor de la API
**Quiero** recibir un error claro cuando mi petición no cumple el schema
**Para que** pueda corregir el payload sin que el sistema procese datos inválidos

**Criterios de Aceptación (Gherkin):**

```gherkin
Scenario: Rechazo por canal inválido
  When un microservicio envía POST /api/v1/notifications con:
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "channel": "whatsapp",
      "priority": "high",
      "templateCode": "otp.verification",
      "recipient": { "type": "phone", "value": "+34612345678" },
      "payload": { "code": "847291" }
    }
    """
  Then el sistema responde con HTTP 400 Bad Request
  And el body contiene un campo "reason" que indica "Invalid channel: whatsapp. Must be one of: email, sms, push"
  And la notificación NO se encola
  And no se registra ningún evento en notification_events
```

```gherkin
Scenario: Rechazo por campos obligatorios faltantes
  When un microservicio envía POST /api/v1/notifications sin el campo "recipient":
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "channel": "email",
      "priority": "high",
      "templateCode": "otp.verification",
      "payload": { "code": "847291" }
    }
    """
  Then el sistema responde con HTTP 400 Bad Request
  And el body contiene "reason" indicando que "recipient" es requerido
```

```gherkin
Scenario: Rechazo por correlationId con formato UUID inválido
  When un microservicio envía POST /api/v1/notifications con:
    """
    {
      "correlationId": "not-a-uuid",
      "channel": "email",
      "priority": "high",
      "templateCode": "otp.verification",
      "recipient": { "type": "email", "value": "user@example.com" },
      "payload": { "code": "847291" }
    }
    """
  Then el sistema responde con HTTP 400 Bad Request
  And el body contiene "reason" indicando formato UUID inválido
```

```gherkin
Scenario: Rechazo por templateCode con caracteres no permitidos
  When un microservicio envía POST /api/v1/notifications con:
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "channel": "email",
      "priority": "high",
      "templateCode": "otp/verification<script>",
      "recipient": { "type": "email", "value": "user@example.com" },
      "payload": { "code": "847291" }
    }
    """
  Then el sistema responde con HTTP 400 Bad Request
  And el body contiene "reason" indicando que templateCode no cumple el patrón permitido
```

---

### Épica E2: Resiliencia y Gestión de Fallos

---

#### US-03: Fallback por Error en Proveedor Principal con Reintentos

**Como** equipo de operaciones
**Quiero** que el sistema reintente automáticamente cuando un proveedor falla temporalmente
**Para que** las notificaciones críticas no se pierdan por fallos transitorios del proveedor

**Contexto técnico:**
- Backoff exponencial: 0s → 1s → 2s → 4s → 8s (prioridad Alta: la mitad)
- Máximo 5 intentos
- Tras agotar reintentos: dead-letter queue
- Solo errores `retryable: true` generan reintentos

**Criterios de Aceptación (Gherkin):**

```gherkin
Scenario: Reintento exitoso tras fallo transitorio del proveedor (prioridad Alta)
  Given existe una notificación encolada con priority "high" y channel "email"
  And el primer intento de envío a SendGrid falla con error retryable (PROVIDER_RATE_LIMIT)
  When el sistema ejecuta la lógica de reintento
  Then el segundo intento se ejecuta después de 500ms (backoff reducido para alta prioridad)
  And si el segundo intento es exitoso:
    And se registra un evento "retried" con attemptNumber 2
    And se registra un evento "sent" con providerMessageId de SendGrid
    And la métrica notification_retry_total{priority="high", channel="email", error_code="PROVIDER_RATE_LIMIT"} incrementa en 1
    And la métrica notification_delivery_total{priority="high", channel="email", status="sent"} incrementa en 1
    And el status de la notificación en la tabla notifications es "sent"
```

```gherkin
Scenario: Dead-letter tras agotar todos los reintentos (prioridad Media)
  Given existe una notificación encolada con priority "medium" y channel "sms"
  And los 5 intentos de envío a Twilio fallan con error retryable
  When el sistema agota el máximo de reintentos
  Then la notificación se mueve a la dead-letter queue "notifications.dlq.medium"
  And el status en la tabla notifications cambia a "dead_lettered"
  And se registra un evento "dead_lettered" en notification_events
  And la métrica notification_dead_letter_total{priority="medium", channel="sms"} incrementa en 1
  And la notificación aparece en GET /api/v1/notifications/dead-letter
```

```gherkin
Scenario: No reintento para error no recuperable
  Given existe una notificación encolada con priority "high" y channel "email"
  And el intento de envío falla con error NO retryable (INVALID_RECIPIENT)
  When el sistema evalúa si debe reintentar
  Then la notificación va directamente a dead-letter queue sin reintentos
  And se registra un evento "dead_lettered" con error.code "INVALID_RECIPIENT"
  And notification_retry_total NO incrementa
```

```gherkin
Scenario: Secuencia completa de backoff exponencial para prioridad Media
  Given una notificación con priority "medium" falla en el primer intento
  When el sistema ejecuta los reintentos
  Then el intento 2 ocurre después de 1s
  And el intento 3 ocurre después de 2s adicionales (3s acumulado)
  And el intento 4 ocurre después de 4s adicionales (7s acumulado)
  And el intento 5 ocurre después de 8s adicionales (15s acumulado)
  And cada intento fallido registra un evento "retried" con el attemptNumber correspondiente
```

---

### Épica E3: Idempotencia y Trazabilidad

---

#### US-04: Idempotencia por CorrelationId Duplicado

**Como** desarrollador de un microservicio
**Quiero** que el sistema detecte cuando reenvío la misma notificación (mismo correlationId)
**Para que** no se genere una doble entrega al usuario final

**Contexto técnico:**
- correlationId es UUID v4, generado por el publisher
- La tabla `notifications` tiene constraint UNIQUE en correlation_id
- Si el correlationId ya existe, se responde con status "duplicate" sin reencolar

**Criterios de Aceptación (Gherkin):**

```gherkin
Scenario: Detección de correlationId duplicado
  Given ya existe una notificación con correlationId "550e8400-e29b-41d4-a716-446655440000" en la tabla notifications
  When un microservicio envía POST /api/v1/notifications con el mismo correlationId:
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "channel": "email",
      "priority": "high",
      "templateCode": "otp.verification",
      "recipient": { "type": "email", "value": "user@example.com" },
      "payload": { "code": "847291", "expiryMinutes": 10, "userName": "Ivan" }
    }
    """
  Then el sistema responde con HTTP 409 Conflict
  And el body de la respuesta contiene:
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "status": "duplicate",
      "acceptedAt": "<timestamp original de la primera aceptación>",
      "reason": "Duplicate correlationId"
    }
    """
  And NO se crea un nuevo registro en la tabla notifications
  And NO se encola ningún mensaje en RabbitMQ
  And NO se envía ningún email adicional a "user@example.com"
```

```gherkin
Scenario: Reenvío con correlationId diferente genera nueva notificación
  Given ya existe una notificación con correlationId "550e8400-e29b-41d4-a716-446655440000"
  When un microservicio envía POST /api/v1/notifications con un correlationId diferente:
    """
    {
      "correlationId": "660e8400-e29b-41d4-a716-446655440001",
      "channel": "email",
      "priority": "high",
      "templateCode": "otp.verification",
      "recipient": { "type": "email", "value": "user@example.com" },
      "payload": { "code": "847291", "expiryMinutes": 10, "userName": "Ivan" }
    }
    """
  Then el sistema responde con HTTP 202 Accepted
  And se crea un nuevo registro en notifications con el nuevo correlationId
  And la notificación se procesa normalmente
```

```gherkin
Scenario: Consulta de estado retorna historial completo de eventos
  Given existe una notificación con correlationId "550e8400-e29b-41d4-a716-446655440000" que fue enviada exitosamente
  When un microservicio consulta GET /api/v1/notifications/550e8400-e29b-41d4-a716-446655440000
  Then el sistema responde con HTTP 200 OK
  And el body contiene:
    """
    {
      "correlationId": "550e8400-e29b-41d4-a716-446655440000",
      "channel": "email",
      "priority": "high",
      "templateCode": "otp.verification",
      "status": "sent",
      "attemptNumber": 1,
      "provider": "sendgrid",
      "providerMessageId": "<sendgrid-message-id>",
      "createdAt": "<timestamp>",
      "updatedAt": "<timestamp>",
      "events": [
        { "type": "accepted", "timestamp": "<t1>" },
        { "type": "enqueued", "timestamp": "<t2>" },
        { "type": "dispatched", "timestamp": "<t3>" },
        { "type": "sent", "timestamp": "<t4>" }
      ]
    }
    """
  And los eventos están ordenados cronológicamente
```

---

### Épica E4: Gestión de Templates

---

#### US-05: Renderizado de Template con Variables Dinámicas

**Como** sistema de notificaciones
**Quiero** renderizar plantillas con variables del payload usando Handlebars
**Para que** cada notificación tenga contenido personalizado sin lógica en el template

**Criterios de Aceptación (Gherkin):**

```gherkin
Scenario: Renderizado exitoso de template OTP para email
  Given existe un template activo con:
    | code               | version | channel | subject_template              | body_template                                          |
    | otp.verification   | 1.0.0   | email   | Tu código de verificación     | Hola {{userName}}, tu código es: <strong>{{code}}</strong>. Expira en {{expiryMinutes}} minutos. |
  When el sistema procesa una notificación con payload:
    """
    { "code": "847291", "expiryMinutes": 10, "userName": "Ivan" }
    """
  Then el subject renderizado es: "Tu código de verificación"
  And el body renderizado es: "Hola Ivan, tu código es: <strong>847291</strong>. Expira en 10 minutos."
  And no se ejecuta lógica arbitraria en el template (logic-less)
```

```gherkin
Scenario: Error cuando template no existe
  Given NO existe un template con templateCode "non.existent.template"
  When el sistema intenta procesar una notificación con ese templateCode
  Then la notificación falla con error "TEMPLATE_RENDER_ERROR"
  And se registra un evento "failed" con error.code "TEMPLATE_RENDER_ERROR"
  And si el error es retryable, se intenta el reintento
  And si NO es retryable, va directo a dead-letter
```

```gherkin
Scenario: Uso de versión específica de template
  Given existen dos versiones del template "order.confirmed":
    | code            | version | body_template                          |
    | order.confirmed | 1.0.0   | Tu pedido {{orderNumber}} fue confirmado |
    | order.confirmed | 2.0.0   | Gracias por tu pedido {{orderNumber}}    |
  When se publica una notificación con templateVersion "1.0.0"
  Then el sistema renderiza con la versión 1.0.0
  And el body renderizado es: "Tu pedido ORD-2026-0042 fue confirmado"
```

```gherkin
Scenario: Uso de última versión activa cuando no se especifica version
  Given existen dos versiones del template "order.confirmed":
    | code            | version | is_active |
    | order.confirmed | 1.0.0   | false     |
    | order.confirmed | 2.0.0   | true      |
  When se publica una notificación SIN templateVersion
  Then el sistema usa la versión 2.0.0 (última activa)
```

---

### Épica E5: Observabilidad y Operación

---

#### US-06: Exposición de Métricas Prometheus y Health Check

**Como** equipo de operaciones (SRE)
**Quiero** consultar métricas en formato Prometheus y el estado de salud del servicio
**Para que** pueda monitorear la entrega de notificaciones y detectar anomalías

**Criterios de Aceptación (Gherkin):**

```gherkin
Scenario: Endpoint de métricas expone todas las métricas requeridas
  When un scraper de Prometheus consulta GET /api/v1/metrics
  Then el sistema responde con HTTP 200 OK
  And el Content-Type es "text/plain; version=0.0.4"
  And la respuesta incluye la métrica notification_enqueue_total con labels priority y channel
  And la respuesta incluye la métrica notification_delivery_total con labels priority, channel y status
  And la respuesta incluye la métrica notification_delivery_duration_ms (histogram)
  And la respuesta incluye la métrica notification_enqueue_latency_ms (histogram)
  And la respuesta incluye la métrica notification_retry_total con labels priority, channel y error_code
  And la respuesta incluye la métrica notification_dead_letter_total con labels priority y channel
  And la respuesta incluye la métrica notification_queue_depth con label priority
  And la respuesta incluye la métrica notification_template_render_duration_ms (histogram)
```

```gherkin
Scenario: Health check reporta estado saludable
  Given RabbitMQ está conectado y PostgreSQL está conectado
  When un load balancer consulta GET /api/v1/health
  Then el sistema responde con HTTP 200 OK
  And el body contiene:
    """
    {
      "status": "healthy",
      "checks": {
        "rabbitmq": "connected",
        "postgres": "connected"
      }
    }
    """
```

```gherkin
Scenario: Health check reporta degradación
  Given RabbitMQ NO está disponible
  When un load balancer consulta GET /api/v1/health
  Then el sistema responde con HTTP 503 Service Unavailable
  And el body contiene:
    """
    {
      "status": "unhealthy",
      "checks": {
        "rabbitmq": "disconnected",
        "postgres": "connected"
      }
    }
    """
```

---

## 6. Requisitos Funcionales

| ID | Requisito | Prioridad | Épica |
|---|---|---|---|
| **FR-001** | El sistema acepta publicaciones vía POST /api/v1/notifications con validación JSON Schema | P0 | E1 |
| **FR-002** | El sistema genera correlationId UUID v4 si el publisher no lo proporciona | P0 | E1 |
| **FR-003** | El sistema valida el schema antes de encolar; rechaza con 400 si es inválido | P0 | E1 |
| **FR-004** | El sistema encola en la cola física correspondiente según prioridad y canal | P0 | E1 |
| **FR-005** | El sistema renderiza templates Handlebars con variables del payload | P0 | E4 |
| **FR-006** | El sistema envía emails vía SendGrid usando IEmailRepository | P0 | E1 |
| **FR-007** | El sistema envía SMS vía Twilio usando ISMSRepository | P0 | E1 |
| **FR-008** | El sistema implementa reintentos con backoff exponencial (máx 5) | P0 | E2 |
| **FR-009** | El sistema mueve a DLQ las notificaciones que exceden máx reintentos | P0 | E2 |
| **FR-010** | El sistema detecta correlationId duplicado y responde 409 sin reencolar | P0 | E3 |
| **FR-011** | El sistema permite consultar estado por correlationId con historial de eventos | P0 | E3 |
| **FR-012** | El sistema expone métricas Prometheus en /api/v1/metrics | P0 | E5 |
| **FR-013** | El sistema expone health check en /api/v1/health | P1 | E5 |
| **FR-014** | El sistema permite listar notificaciones en DLQ vía GET /api/v1/notifications/dead-letter | P1 | E2 |
| **FR-015** | El sistema permite reintentar una notificación desde DLQ vía POST /api/v1/notifications/dead-letter/:id/retry | P1 | E2 |
| **FR-016** | El sistema registra cada transición de estado en notification_events | P0 | E3 |
| **FR-017** | El sistema soporta templates versionados con versión activa por defecto | P0 | E4 |
| **FR-018** | El sistema aplica rate limiting de 100 msg/min a la cola de baja prioridad | P1 | E1 |

---

## 7. Requisitos No Funcionales

| ID | Requisito | Métrica | Validación |
|---|---|---|---|
| **RNF-001** | Latencia de encolamiento (prioridad Alta) | p99 < 100ms | `notification_enqueue_latency_ms{priority="high"}` |
| **RNF-002** | Disponibilidad del servicio | 99.9% uptime mensual | Uptime monitoring (Pingdom/UptimeRobot) |
| **RNF-003** | Escalabilidad ante picos 10x | Sin degradación de prioridad Alta | Test de carga con k6 |
| **RNF-004** | Encriptación de datos sensibles | TLS 1.3+ en tránsito, AES-256 en reposo | Auditoría de configuración |
| **RNF-005** | Tasa de entrega (prioridad Alta) | > 99.5% | `notification_delivery_total` ratio |
| **RNF-006** | Idempotencia garantizada | 0 entregas duplicadas por correlationId | Test US-04 + auditoría post-migración |

---

## 8. Riesgos y Mitigaciones

| # | Riesgo | Impacto | Probabilidad | Mitigación |
|---|---|---|---|---|
| R1 | SendGrid/Twilio sufren outage prolongado | Alto | Media | DLQ con replay manual; alerta inmediata a SRE |
| R2 | Cola de baja prioridad satura recursos | Alto | Media | Rate limit 100 msg/min + queue length limit 200K + consumers aislados |
| R3 | Equipos resisten migración desde implementaciones actuales | Medio | Alta | API drop-in + documentación clara + equipos piloto |
| R4 | Templates se convierten en cuello de botella | Medio | Media | MVP con templates gestionados por equipo plataforma; self-service en Fase 3 |
| R5 | Costes de proveedores se disparan | Alto | Media | Rate limiting por consumidor + alertas de presupuesto |

---

## 9. Dependencias

| Dependencia | Tipo | Estado | Responsable |
|---|---|---|---|
| Cuenta SendGrid activa con API key | Externa | Pendiente | Equipo de plataforma |
| Cuenta Twilio activa con Account SID + Auth Token | Externa | Pendiente | Equipo de plataforma |
| RabbitMQ desplegado (o Docker Compose en dev) | Infraestructura | Pendiente | Equipo de plataforma |
| PostgreSQL 16+ desplegado | Infraestructura | Pendiente | Equipo de plataforma |
| Prometheus + Grafana disponibles | Infraestructura | Pendiente | Equipo SRE |
| Templates por defecto (OTP) definidos | Contenido | Pendiente | Equipo de producto |

---

## 10. Plan de Adopción

### Fase 0: Piloto (Semanas 1-2)
- Desplegar infraestructura (RabbitMQ + PostgreSQL + servicio)
- Integrar 1 microservicio piloto (auth-service para OTP)
- Validar K1 (tasa de entrega) y K2 (latencia) con tráfico real

### Fase 1: Expansión (Semanas 3-6)
- Migrar order-service y billing-service
- Validar K5 (trazabilidad 100%) y K3 (reducción de boilerplate)
- Activar alertas de Prometheus

### Fase 2: Generalización (Semanas 7-12)
- Migrar resto de microservicios
- Validar K6 (>50% migrados)
- Ejercicio de cambio de proveedor (K4) en staging

---

## 11. Glosario

| Término | Definición |
|---|---|
| **CorrelationId** | UUID v4 que identifica de forma única una notificación y garantiza idempotencia |
| **DLQ (Dead-Letter Queue)** | Cola donde se depositan mensajes que no pudieron entregarse tras agotar reintentos |
| **Backoff Exponencial** | Patrón de reintento donde el delay se duplica en cada intento (1s, 2s, 4s, 8s) |
| **TemplateCode** | Identificador de plantilla (ej: `otp.verification`) que referencia un template versionado |
| **Prioridad Alta** | Notificaciones que requieren entrega inmediata: OTPs, alertas de seguridad |
| **Prioridad Media** | Notificaciones transaccionales: confirmaciones de pedido, estados de cuenta |
| **Prioridad Baja** | Notificaciones no urgentes: resúmenes, marketing, newsletters |

---

## 12. Historial de Decisiones

| # | Decisión | Rationale | Fuente |
|---|---|---|---|
| D1 | MVP limitado a Email + SMS (sin Push) | Diseño emergente: validar los 2 canales principales antes de añadir complejidad | Technical Spec (Winston) |
| D2 | RabbitMQ como message broker | Routing por topic nativo, prefetch configurable, DLQ nativa | ADR-001 |
| D3 | Handlebars como template engine | Logic-less (seguridad), ampliamente adoptado, fácil de versionar | ADR-002 |
| D4 | PostgreSQL para persistencia de estado | DB estándar del ecosistema; JSONB para payload flexible | ADR-003 |
| D5 | TypeScript + Fastify como stack | Type safety, rendimiento, expertise del equipo | ADR-004 |
| D6 | CorrelationId como clave de idempotencia | Simple, sin estado distribuido adicional, constraint UNIQUE en DB | ADR-005 |
| D7 | Colas físicas separadas (no lógica) | Aislamiento total: flood de baja prioridad no afecta alta | Technical Spec §4 |
| D8 | Rate limit 100 msg/min solo en cola baja | Protege el sistema sin afectar SLA de colas alta y media | Technical Spec §4.3.3 |

---

*Documento generado por John (Product Manager) — 2026-05-18*
*Fuentes de verdad: 01-product-brief.md (Mary) + 03-technical-spec.md (Winston)*
*Estado: **approved** — listo para apertura de ramas de feature*
*Próximo paso: bmad-create-epics-and-stories para desglose en tareas de desarrollo*
