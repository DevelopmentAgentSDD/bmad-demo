---
document_type: "Technical Specification"
version: "1.0.0"
status: "draft"
created: "2026-05-18"
author: "Winston (System Architect)"
project_name: "Sistema de Notificaciones Omnicanal"
phase: "Fase 1 - MVP"
source_documents:
  - "_bmad-output/01-product-brief.md"
---

# Technical Specification: Sistema de Notificaciones Omnicanal — Fase 1 (MVP)

## 1. Contexto y Alcance Técnico

Este documento traduce el Product Brief (Mary, 2026-05-18) en decisiones técnicas concretas para la **Fase 1 del MVP**. El alcance se limita a:

- **Canales**: Email + SMS (Push queda para Fase 2)
- **Entradas**: API REST + contrato de eventos (message broker)
- **Priorización**: 3 colas independientes (Alta, Media, Baja)
- **Resiliencia**: Retries con backoff exponencial + dead-letter queue
- **Templates**: Motor de plantillas versionado con variables dinámicas

### 1.1 Principios Arquitectónicos

| Principio | Decisión | Rationale |
|---|---|---|
| **Diseño Emergente** | Sin abstracciones prematuras; interfaces concretas por canal | Regla de 3: no abstraer hasta que haya 3 implementaciones reales |
| **Clean Architecture** | Capa de dominio pura; proveedores como plugins en infraestructura | Dependencias apuntan hacia adentro; el dominio no conoce proveedores |
| **Boring Technology** | Stack maduro y conocido por el equipo | Estabilidad > novedad en plataforma crítica |
| **Fail-Safe por Defecto** | Circuit breaker + retries + DLQ desde día 1 | Las notificaciones son superficie de fallo amplia; mejor prevenir |

---

## 2. Contratos de Eventos y Schemas

### 2.1 Contrato de Publicación (NotificationRequest)

Este es el schema que todo microservicio consumidor debe enviar, ya sea vía REST o vía message broker.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "notification-request.schema.json",
  "title": "NotificationRequest",
  "description": "Contrato de publicación para el Sistema de Notificaciones Omnicanal",
  "type": "object",
  "required": ["correlationId", "channel", "priority", "templateCode", "recipient", "payload"],
  "properties": {
    "correlationId": {
      "type": "string",
      "format": "uuid",
      "description": "Identificador único de la notificación. Idempotencia: reenviar con mismo ID no genera doble entrega.",
      "examples": ["550e8400-e29b-41d4-a716-446655440000"]
    },
    "channel": {
      "type": "string",
      "enum": ["email", "sms", "push"],
      "description": "Canal de destino de la notificación"
    },
    "priority": {
      "type": "string",
      "enum": ["high", "medium", "low"],
      "description": "Nivel de prioridad que determina la cola de procesamiento",
      "default": "medium"
    },
    "templateCode": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100,
      "pattern": "^[a-zA-Z0-9._-]+$",
      "description": "Código identificador de la plantilla a usar (versionada en el sistema)",
      "examples": ["otp.verification", "order.confirmed", "weekly.summary"]
    },
    "templateVersion": {
      "type": "string",
      "description": "Versión específica de la plantilla. Si se omite, se usa la última activa.",
      "examples": ["1.0.0", "2.1.0"]
    },
    "recipient": {
      "type": "object",
      "required": ["type", "value"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["email", "phone", "device_token", "user_id"],
          "description": "Tipo de identificador del destinatario"
        },
        "value": {
          "type": "string",
          "description": "Valor del identificador (email, phone number, device token, user ID)",
          "examples": ["user@example.com", "+34612345678", "fcm-token-abc123", "usr_001"]
        }
      },
      "additionalProperties": false
    },
    "payload": {
      "type": "object",
      "description": "Variables dinámicas que se inyectan en la plantilla. Keys deben coincidir con placeholders del template.",
      "additionalProperties": {
        "type": ["string", "number", "boolean", "null"]
      },
      "examples": [
        { "code": "847291", "expiryMinutes": 10, "userName": "Ivan" },
        { "orderNumber": "ORD-2026-0042", "total": 149.99, "currency": "EUR" }
      ]
    },
    "metadata": {
      "type": "object",
      "description": "Datos opcionales para trazabilidad, tagging o routing interno. No se exponen al destinatario.",
      "properties": {
        "sourceService": {
          "type": "string",
          "description": "Nombre del microservicio que originó la notificación",
          "examples": ["auth-service", "order-service", "billing-service"]
        },
        "tenantId": {
          "type": "string",
          "description": "Identificador de tenant para sistemas multi-tenant"
        },
        "campaignId": {
          "type": "string",
          "description": "ID de campaña para notificaciones de marketing (prioridad baja)"
        },
        "tags": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Tags libres para filtrado y agregación en observabilidad",
          "examples": [["otp", "security"], ["order", "transactional"]]
        }
      },
      "additionalProperties": true
    },
    "scheduledAt": {
      "type": "string",
      "format": "date-time",
      "description": "Fecha/hora futura para envío programado. Si se omite, envío inmediato.",
      "examples": ["2026-05-20T09:00:00Z"]
    }
  },
  "additionalProperties": false
}
```

### 2.2 Contrato de Respuesta (NotificationResponse)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "notification-response.schema.json",
  "title": "NotificationResponse",
  "type": "object",
  "required": ["correlationId", "status", "acceptedAt"],
  "properties": {
    "correlationId": {
      "type": "string",
      "format": "uuid",
      "description": "Mismo correlationId del request"
    },
    "status": {
      "type": "string",
      "enum": ["accepted", "rejected", "duplicate"],
      "description": "accepted: encolada correctamente | rejected: validación fallida | duplicate: correlationId ya procesado"
    },
    "acceptedAt": {
      "type": "string",
      "format": "date-time",
      "description": "Timestamp de aceptación por el sistema"
    },
    "reason": {
      "type": "string",
      "description": "Motivo de rechazo o duplicación (solo presente si status != accepted)",
      "examples": ["Template 'otp.verification' not found", "Invalid email format", "Duplicate correlationId"]
    },
    "estimatedDelivery": {
      "type": "string",
      "format": "date-time",
      "description": "Estimación de entrega basada en prioridad y latencia actual del canal"
    }
  },
  "additionalProperties": false
}
```

### 2.3 Contrato de Evento de Estado (NotificationStatusEvent)

Evento publicado al message broker para cada transición de estado de una notificación.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "notification-status-event.schema.json",
  "title": "NotificationStatusEvent",
  "type": "object",
  "required": ["correlationId", "eventType", "timestamp", "channel"],
  "properties": {
    "correlationId": {
      "type": "string",
      "format": "uuid"
    },
    "eventType": {
      "type": "string",
      "enum": ["enqueued", "dispatched", "sent", "delivered", "failed", "retried", "dead_lettered"],
      "description": "Transición de estado de la notificación"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "channel": {
      "type": "string",
      "enum": ["email", "sms", "push"]
    },
    "priority": {
      "type": "string",
      "enum": ["high", "medium", "low"]
    },
    "attemptNumber": {
      "type": "integer",
      "minimum": 1,
      "description": "Número de intento actual (1 = primer intento)"
    },
    "provider": {
      "type": "string",
      "description": "Proveedor externo utilizado",
      "examples": ["sendgrid", "twilio", "fcm"]
    },
    "providerMessageId": {
      "type": "string",
      "description": "ID asignado por el proveedor externo (para trazabilidad cruzada)"
    },
    "error": {
      "type": "object",
      "description": "Detalles del error (solo presente en failed/retried/dead_lettered)",
      "properties": {
        "code": {
          "type": "string",
          "examples": ["PROVIDER_RATE_LIMIT", "INVALID_RECIPIENT", "TEMPLATE_RENDER_ERROR"]
        },
        "message": {
          "type": "string"
        },
        "retryable": {
          "type": "boolean",
          "description": "Si el error es recuperable con retry"
        }
      }
    },
    "metadata": {
      "type": "object",
      "description": "Propagación del metadata original del request"
    }
  },
  "additionalProperties": false
}
```

### 2.4 Tabla de Transiciones de Estado

```
accepted ──► enqueued ──► dispatched ──► sent ──► delivered
                           │               │
                           │               ├─► failed ──► retried ──► dispatched (retry loop)
                           │               │                  │
                           │               │                  └─► dead_lettered (max retries exceeded)
                           │               │
                           │               └─► delivered (confirmación asíncrona del proveedor)
                           │
                           └─► rejected (validación síncrona fallida)
```

---

## 3. Abstracción de Proveedores — Interfaces de Repositorio

### 3.1 Principio de Diseño

Cada canal tiene su propia interfaz de repositorio en la **capa de dominio**. Las implementaciones concretas viven en la **capa de infraestructura**. El dominio nunca importa código de infraestructura.

```
┌──────────────────────────────────────────────────┐
│                   DOMINIO                        │
│                                                  │
│  interface INotificationRepository               │
│  interface IEmailRepository                      │
│  interface ISMSRepository                        │
│  interface IPushRepository                       │
│  interface ITemplateRepository                   │
│  interface IMetricsRepository                    │
│                                                  │
│  (entidades, value objects, servicios de dominio)│
└──────────────────────┬───────────────────────────┘
                       │  (depende de)
                       ▼
┌──────────────────────────────────────────────────┐
│               INFRAESTRUCTURA                    │
│                                                  │
│  SendGridEmailRepository : IEmailRepository      │
│  TwilioSMSRepository   : ISMSRepository          │
│  FcmPushRepository     : IPushRepository         │
│  HandlebarsTemplateRepo: ITemplateRepository     │
│  PrometheusMetricsRepo : IMetricsRepository      │
│                                                  │
│  (conexiones HTTP, SDKs externos, DB, cache)     │
└──────────────────────────────────────────────────┘
```

### 3.2 Value Objects del Dominio

```typescript
// --- Value Objects ---

type CorrelationId = string // UUID v4
type TemplateCode = string  // "otp.verification", "order.confirmed"
type TemplateVersion = string // semver: "1.0.0"

type Priority = "high" | "medium" | "low"
type Channel = "email" | "sms" | "push"
type NotificationStatus =
  | "accepted"
  | "enqueued"
  | "dispatched"
  | "sent"
  | "delivered"
  | "failed"
  | "retried"
  | "dead_lettered"
  | "rejected"
  | "duplicate"

interface Recipient {
  type: "email" | "phone" | "device_token" | "user_id"
  value: string
}

interface NotificationPayload {
  [key: string]: string | number | boolean | null
}

interface NotificationMetadata {
  sourceService?: string
  tenantId?: string
  campaignId?: string
  tags?: string[]
  [key: string]: unknown
}

// --- Entidad de Dominio ---

interface Notification {
  correlationId: CorrelationId
  channel: Channel
  priority: Priority
  templateCode: TemplateCode
  templateVersion?: TemplateVersion
  recipient: Recipient
  payload: NotificationPayload
  metadata: NotificationMetadata
  scheduledAt?: Date
  status: NotificationStatus
  createdAt: Date
  updatedAt: Date
  attemptNumber: number
  providerMessageId?: string
  lastError?: NotificationError
}

interface NotificationError {
  code: string
  message: string
  retryable: boolean
}
```

### 3.3 Interfaces de Repositorio (Capa de Dominio)

#### 3.3.1 INotificationRepository — Cola y Persistencia

```typescript
interface INotificationRepository {
  /**
   * Persiste una notificación recién aceptada y la encola según su prioridad.
   * Lanza DomainError si el correlationId ya existe (idempotencia).
   */
  enqueue(notification: Notification): Promise<void>

  /**
   * Obtiene la siguiente notificación pendiente de la cola de prioridad especificada.
   * Retorna null si la cola está vacía.
   */
  dequeueNext(priority: Priority, channel: Channel): Promise<Notification | null>

  /**
   * Marca una notificación como enviada exitosamente.
   */
  markAsSent(correlationId: CorrelationId, providerMessageId: string): Promise<void>

  /**
   * Marca una notificación como fallida, incrementando el contador de intentos.
   * Si se excede el máximo de retries, mueve a dead-letter.
   */
  markAsFailed(correlationId: CorrelationId, error: NotificationError): Promise<void>

  /**
   * Obtiene una notificación por correlationId (para consultas de estado).
   */
  findById(correlationId: CorrelationId): Promise<Notification | null>

  /**
   * Obtiene notificaciones en dead-letter queue para replay manual.
   */
  listDeadLetter(limit?: number, offset?: number): Promise<Notification[]>

  /**
   * Reintenta una notificación desde dead-letter queue.
   */
  retryFromDeadLetter(correlationId: CorrelationId): Promise<void>
}
```

#### 3.3.2 IEmailRepository — Abstracción de Proveedor Email

```typescript
interface IEmailRepository {
  /**
   * Envía un email usando el proveedor configurado.
   * Retorna el ID asignado por el proveedor externo.
   */
  send(input: EmailInput): Promise<ProviderResult>
}

interface EmailInput {
  to: string
  subject: string
  body: string        // HTML renderizado desde el template
  bodyText?: string   // Versión plain-text (opcional, fallback accesibilidad)
  replyTo?: string
  correlationId: CorrelationId
}

interface ProviderResult {
  success: boolean
  providerMessageId?: string  // ID del proveedor (SendGrid message ID, Twilio SID, etc.)
  error?: {
    code: string
    message: string
    retryable: boolean
  }
}
```

#### 3.3.3 ISMSRepository — Abstracción de Proveedor SMS

```typescript
interface ISMSRepository {
  /**
   * Envía un SMS usando el proveedor configurado.
   * Retorna el ID asignado por el proveedor externo.
   */
  send(input: SMSInput): Promise<ProviderResult>
}

interface SMSInput {
  to: string            // E.164 format: "+34612345678"
  body: string          // Texto renderizado desde el template (max 160 chars por segmento)
  correlationId: CorrelationId
}
```

#### 3.3.4 IPushRepository — Abstracción de Proveedor Push (Fase 2)

```typescript
interface IPushRepository {
  send(input: PushInput): Promise<ProviderResult>
}

interface PushInput {
  deviceToken: string
  title: string
  body: string
  data?: Record<string, string>  // Payload custom para la app
  correlationId: CorrelationId
}
```

#### 3.3.5 ITemplateRepository — Gestión de Plantillas

```typescript
interface ITemplateRepository {
  /**
   * Obtiene una plantilla por código. Si version es null, retorna la última activa.
   * Lanza TemplateNotFoundError si no existe.
   */
  getTemplate(code: TemplateCode, version?: TemplateVersion): Promise<Template>

  /**
   * Renderiza una plantilla con las variables del payload.
   * Retorna el contenido renderizado (HTML para email, texto para SMS).
   */
  render(template: Template, payload: NotificationPayload): Promise<string>

  /**
   * Registra una nueva versión de una plantilla.
   */
  saveTemplate(code: TemplateCode, version: TemplateVersion, content: TemplateContent): Promise<void>
}

interface Template {
  code: TemplateCode
  version: TemplateVersion
  channel: Channel
  subjectTemplate?: string   // Solo para email
  bodyTemplate: string       // Handlebars template
  isActive: boolean
  createdAt: Date
}

interface TemplateContent {
  subjectTemplate?: string
  bodyTemplate: string
}
```

#### 3.3.6 IMetricsRepository — Observabilidad

```typescript
interface IMetricsRepository {
  /**
   * Registra una métrica de entrega.
   */
  recordDelivery(channel: Channel, priority: Priority, durationMs: number, success: boolean): Promise<void>

  /**
   * Registra un intento fallido con detalles.
   */
  recordFailure(channel: Channel, priority: Priority, errorCode: string, retryable: boolean): Promise<void>

  /**
   * Registra la latencia de encolamiento (tiempo entre aceptación y dispatch).
   */
  recordEnqueueLatency(priority: Priority, latencyMs: number): Promise<void>

  /**
   * Obtiene métricas agregadas para dashboard.
   */
  getDeliveryStats(channel: Channel, window: TimeWindow): Promise<DeliveryStats>
}

interface TimeWindow {
  from: Date
  to: Date
}

interface DeliveryStats {
  totalSent: number
  totalFailed: number
  successRate: number
  p50LatencyMs: number
  p95LatencyMs: number
  p99LatencyMs: number
}
```

### 3.4 Implementaciones Concretas (Capa de Infraestructura) — MVP

Para la Fase 1, cada interfaz tiene **una sola implementación**. No hay factory ni registry todavía — se inyecta directamente vía DI.

```
IEmailRepository    ──► SendGridEmailRepository    (usa @sendgrid/mail SDK)
ISMSRepository      ──► TwilioSMSRepository         (usa twilio SDK)
IPushRepository     ──► (vacío — Fase 2)
ITemplateRepository ──► HandlebarsTemplateRepository (usa handlebars + DB/storage)
IMetricsRepository  ──► PrometheusMetricsRepository (usa prom-client)
INotificationRepository ──► RabbitMQNotificationRepository (usa amqplib + DB para persistencia)
```

**Trade-off deliberado**: No creamos una capa de factory/registry de proveedores en el MVP. Cuando haya un segundo proveedor por canal (Fase 2), extraemos la factory. **Regla de 3 de Fowler**: no abstraer hasta que haya 3 usos concretos.

---

## 4. Estrategia de Infraestructura — Colas de Prioridad

### 4.1 Problema a Resolver

El riesgo R2 del Product Brief es explícito: *"Cola de baja prioridad satura recursos y afecta alta"*. La solución debe garantizar que **una avalancha de notificaciones de marketing (baja) nunca bloquee un OTP (alta)**.

### 4.2 Opciones Evaluadas

| Opción | Descripción | Pros | Contras | Decisión |
|---|---|---|---|---|
| **A: Cola única con prioridad por mensaje** | Un solo queue; el broker reordena por campo `priority` | Simple de operar | Un flood de baja prioridad puede saturar el consumer; el reorder tiene overhead | ❌ Rechazada |
| **B: Colas físicas separadas con consumers dedicados** | 3 queues independientes; cada una con su propio consumer group | Aislamiento total; cada cola tiene su throughput budget | Más infraestructura que gestionar | ✅ **Seleccionada** |
| **C: Cola única con weighted fair scheduling** | Un queue; el consumer procesa en ratio 50/30/20 (alta/media/baja) | Balance automático | Si alta está vacía, se desperdicia capacidad; complejidad en scheduler | ❌ Rechazada |
| **D: Colas separadas + consumer pool compartido con prioridades de prefetch** | 3 queues; un pool de workers que hace prefetch preferente de alta | Buen balance aislamiento/eficiencia | Complejidad adicional en lógica de prefetch | ⚠️ Reservada para Fase 3 |

### 4.3 Diseño Seleccionado: Colas Físicas Separadas (Opción B)

#### 4.3.1 Topología de Colas (RabbitMQ)

```
Exchange: notifications.exchange (topic)
│
├─ routing_key: notification.high.*  ──► notifications.queue.high
│                                      ──► consumers: 2+ instancias dedicadas
│                                      ──► prefetch: 5 (procesamiento agresivo)
│                                      ──► DLQ: notifications.dlq.high
│
├─ routing_key: notification.medium.* ──► notifications.queue.medium
│                                       ──► consumers: 1+ instancia
│                                       ──► prefetch: 10
│                                       ──► DLQ: notifications.dlq.medium
│
└─ routing_key: notification.low.*   ──► notifications.queue.low
                                       ──► consumers: 1 instancia
                                       ──► prefetch: 20
                                       ──► DLQ: notifications.dlq.low
                                       ──► rate_limit: 100 msg/min (protección)
```

#### 4.3.2 Naming Convention

```
Cola de trabajo:     notifications.queue.{priority}
Dead-Letter Queue:   notifications.dlq.{priority}
Exchange de entrada: notifications.exchange
Routing keys:        notification.{priority}.{channel}
                     Ej: notification.high.email, notification.low.sms
```

#### 4.3.3 Configuración de Colas

| Parámetro | Alta | Media | Baja |
|---|---|---|---|
| **Consumer instances** | 2+ (auto-scale) | 1+ | 1 |
| **Prefetch count** | 5 | 10 | 20 |
| **Message TTL** | Sin límite | 24h | 48h |
| **Max length** | 50,000 | 100,000 | 200,000 |
| **Rate limit** | Sin límite | 500 msg/min | 100 msg/min |
| **DLQ habilitada** | Sí | Sí | Sí |
| **Auto-scale trigger** | Queue depth > 1,000 | Queue depth > 5,000 | N/A (fijo) |

#### 4.3.4 Mecanismo de Protección Anti-Saturación

```
┌─────────────────────────────────────────────────────────────┐
│                    Protección en 3 Capas                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Capa 1: Rate Limiting por Cola                             │
│  ─ La cola LOW tiene un rate limit estricto de 100 msg/min  │
│  ─ Si se excede, el gateway rechaza con HTTP 429            │
│  ─ Las colas HIGH y MEDIUM no tienen rate limit (SLA)       │
│                                                             │
│  Capa 2: Consumer Isolation                                 │
│  ─ Cada cola tiene sus propios consumers dedicados          │
│  ─ Los consumers de LOW nunca compiten con HIGH por CPU/mem │
│  ─ Si LOW se satura, sus consumers no afectan a los demás   │
│                                                             │
│  Capa 3: Queue Length Limits + Overflow Policy              │
│  ─ Cada cola tiene un max_length configurado                │
│  ─ Cuando se alcanza el límite: overflow = reject-publish   │
│  ─ El publisher recibe error y puede decidir reintentar     │
│    o degradar (ej: log local, retry con backoff)            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 4.3.5 Flujo de Publicación

```
1. Microservicio publica ──► POST /api/notifications
                              o
                              ► AMQP: notifications.exchange (routing_key)

2. Gateway valida schema ──► Si inválido: 400 + motivo
                              Si válido: genera correlationId

3. Gateway determina cola ──► routing_key = "notification.{priority}.{channel}"

4. Message Broker enruta ──► Cola correspondiente según routing_key

5. Consumer dequeue ───────► Obtiene mensaje (prefetch limitado)
                              ─► Resuelve template
                              ─► Renderiza con payload
                              ─► Llama al repositorio del canal
                              ─► Marca como sent/failed
                              ─► Ack/Nack al broker

6. Si failed ──────────────► Reintenta con backoff exponencial
                              ─► Máx 5 intentos
                              ─► Si excede: dead-letter queue
```

#### 4.3.6 Backoff Exponencial — Configuración

| Intento | Delay | Acumulado |
|---|---|---|
| 1 (inicial) | 0s | 0s |
| 2 | 1s | 1s |
| 3 | 2s | 3s |
| 4 | 4s | 7s |
| 5 | 8s | 15s |

**Máximo de reintentos**: 5
**Total tiempo máximo de retry**: ~15 segundos
**Tras 5 intentos fallidos**: mensaje va a DLQ

Para **prioridad Alta**, el backoff se reduce a la mitad (0.5s, 1s, 2s, 4s) para minimizar el tiempo total de recuperación.

---

## 5. Arquitectura de Componentes — Fase 1

### 5.1 Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Microservicios Consumidores                   │
│              (auth-service, order-service, billing-service, ...)     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
┌─────────────────────┐    ┌─────────────────────────────┐
│   REST API Gateway  │    │   AMQP Publisher            │
│   POST /notifications│   │   notifications.exchange    │
│   (validación schema│    │   routing_key:              │
│    + correlationId) │    │   notification.{p}.{c}      │
└──────────┬──────────┘    └──────────────┬──────────────┘
           │                              │
           └──────────────┬───────────────┘
                          ▼
              ┌───────────────────────┐
              │   Message Broker      │
              │   (RabbitMQ)          │
              │                       │
              │   ┌─────────────────┐ │
              │   │ notifications.  │ │
              │   │ queue.high      │ │──┐
              │   └─────────────────┘ │  │
              │   ┌─────────────────┐ │  │
              │   │ notifications.  │ │  │
              │   │ queue.medium    │ │──┤
              │   └─────────────────┘ │  │
              │   ┌─────────────────┐ │  │
              │   │ notifications.  │ │  │
              │   │ queue.low       │ │──┤
              │   └─────────────────┘ │  │
              └───────────────────────┘  │
                                         │
                    ┌────────────────────┘
                    ▼
        ┌───────────────────────────┐
        │  Notification Processor   │
        │                           │
        │  ┌─────────────────────┐  │
        │  │ Template Resolver   │  │
        │  │ (HandlebarsEngine)  │  │
        │  └─────────┬───────────┘  │
        │            │              │
        │  ┌─────────▼───────────┐  │
        │  │ Channel Dispatcher  │  │
        │  │                     │  │
        │  │  ┌───────────────┐  │  │
        │  │  │ Email Sender  │──┼──┼──► SendGrid API
        │  │  │ (IEmailRepo)  │  │  │
        │  │  └───────────────┘  │  │
        │  │  ┌───────────────┐  │  │
        │  │  │ SMS Sender    │──┼──┼──► Twilio API
        │  │  │ (ISMSRepo)    │  │  │
        │  │  └───────────────┘  │  │
        │  └─────────────────────┘  │
        │                           │
        │  ┌─────────────────────┐  │
        │  │ Metrics Recorder    │──┼──► Prometheus
        │  │ (IMetricsRepo)      │  │
        │  └─────────────────────┘  │
        └───────────────────────────┘
```

### 5.2 Estructura de Proyecto Sugerida

```
notification-service/
├── src/
│   ├── domain/
│   │   ├── entities/
│   │   │   └── notification.ts
│   │   ├── value-objects/
│   │   │   ├── priority.ts
│   │   │   ├── channel.ts
│   │   │   └── recipient.ts
│   │   ├── errors/
│   │   │   ├── domain-error.ts
│   │   │   ├── template-not-found-error.ts
│   │   │   └── duplicate-notification-error.ts
│   │   └── ports/                    ← Interfaces de repositorio
│   │       ├── email-repository.interface.ts
│   │       ├── sms-repository.interface.ts
│   │       ├── push-repository.interface.ts
│   │       ├── template-repository.interface.ts
│   │       ├── notification-repository.interface.ts
│   │       └── metrics-repository.interface.ts
│   │
│   ├── application/
│   │   ├── services/
│   │   │   ├── notification-service.ts        ← Orquestador principal
│   │   │   └── template-service.ts            ← Renderizado de templates
│   │   ├── use-cases/
│   │   │   ├── publish-notification.use-case.ts
│   │   │   ├── retry-notification.use-case.ts
│   │   │   └── get-notification-status.use-case.ts
│   │   └── dtos/
│   │       ├── notification-request.dto.ts
│   │       └── notification-response.dto.ts
│   │
│   ├── infrastructure/
│   │   ├── providers/
│   │   │   ├── sendgrid-email.repository.ts
│   │   │   └── twilio-sms.repository.ts
│   │   ├── queue/
│   │   │   ├── rabbitmq-connection.ts
│   │   │   ├── rabbitmq-notification.repository.ts
│   │   │   └── queue-config.ts
│   │   ├── templates/
│   │   │   └── handlebars-template.repository.ts
│   │   ├── metrics/
│   │   │   └── prometheus-metrics.repository.ts
│   │   └── persistence/
│   │       └── notification-store.ts          ← DB para estado y DLQ
│   │
│   ├── api/
│   │   ├── controllers/
│   │   │   └── notification.controller.ts
│   │   ├── middleware/
│   │   │   ├── validation.middleware.ts
│   │   │   ├── correlation-id.middleware.ts
│   │   │   └── error-handler.middleware.ts
│   │   └── schemas/
│   │       ├── notification-request.schema.json
│   │       └── notification-response.schema.json
│   │
│   └── index.ts                               ← Composición DI + bootstrap
│
├── templates/                                  ← Templates por defecto
│   ├── email/
│   │   └── otp.verification.hbs
│   └── sms/
│       └── otp.verification.hbs
│
├── config/
│   ├── default.yaml
│   └── production.yaml
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── docker-compose.yml
├── Dockerfile
└── package.json
```

### 5.3 Secuencia de Publicación — Diagrama

```
Consumer Service          API Gateway          Message Broker        Notification Processor
     │                        │                      │                        │
     │──POST /notifications──►│                      │                        │
     │  {channel, priority,   │                      │                        │
     │   templateCode,        │                      │                        │
     │   recipient, payload}  │                      │                        │
     │                        │                      │                        │
     │                        │──validate schema─────│                        │
     │                        │──generate corrId─────│                        │
     │                        │                      │                        │
     │                        │──publish─────────────►│                        │
     │                        │  routing_key:         │                        │
     │                        │  notification.high.email                      │
     │                        │                      │                        │
     │◄──202 Accepted─────────│                      │                        │
     │   {correlationId,      │                      │                        │
     │    status: "accepted", │                      │                        │
     │    acceptedAt}         │                      │                        │
     │                        │                      │──dequeue──────────────►│
     │                        │                      │  (consumer dedicado)   │
     │                        │                      │                        │
     │                        │                      │                        │──resolve template
     │                        │                      │                        │──render with payload
     │                        │                      │                        │──send via provider
     │                        │                      │                        │
     │                        │                      │                        │──record metrics
     │                        │                      │                        │──ack message
     │                        │                      │                        │──publish status event
```

---

## 6. Decisiones Técnicas Documentadas

### ADR-001: Message Broker — RabbitMQ sobre AWS SQS

| Campo | Valor |
|---|---|
| **Estado** | Propuesto |
| **Contexto** | Necesitamos un broker que soporte múltiples colas con aislamiento, routing por topic, dead-letter queues nativas, y prefetch configurable por consumer |
| **Decisión** | RabbitMQ |
| **Rationale** | Routing por topic (notification.{priority}.{channel}) es nativo en RabbitMQ. SQS requiere un queue separado por prioridad (3 queues) sin routing flexible. RabbitMQ ofrece prefetch por consumer, DLQ nativa, y exchange patterns que simplifican la topología. Si el equipo ya está en AWS sin infra propia, SQS + EventBridge es alternativa válida pero más compleja de configurar para este patrón |
| **Consecuencias** | Operar RabbitMQ requiere gestión de infra (o usar servicio managed como CloudAMQP). SQS sería menos operativo pero menos flexible |

### ADR-002: Template Engine — Handlebars

| Campo | Valor |
|---|---|
| **Estado** | Propuesto |
| **Contexto** | Necesitamos renderizar templates con variables dinámicas para email (HTML) y SMS (texto plano) |
| **Decisión** | Handlebars |
| **Rationale** | Handlebars es logic-less: no permite lógica compleja en templates, lo cual es una feature de seguridad (los templates no pueden ejecutar código arbitrario). Es ampliamente adoptado, tiene buena validación de schemas y es fácil de versionar. Alternativas como Mustache son más restrictivas; EJS permite lógica completa (riesgo de seguridad) |
| **Consecuencias** | Si en el futuro se necesitan helpers complejos en templates, habrá que extender Handlebars con helpers registrados en el dominio |

### ADR-003: Persistencia de Estado — PostgreSQL para Notification Store

| Campo | Valor |
|---|---|
| **Estado** | Propuesto |
| **Contexto** | Necesitamos persistir el estado de las notificaciones para: idempotencia (detectar correlationId duplicados), consulta de estado, dead-letter queue con replay, y auditoría |
| **Decisión** | PostgreSQL |
| **Rationale** | Ya es la base de datos estándar del ecosistema. No necesitamos una DB adicional. La tabla de notificaciones es simple: correlationId (PK), channel, priority, status, payload (JSONB), created_at, updated_at. JSONB permite almacenar el payload sin schema rígido. Índices en correlationId y status para consultas frecuentes |
| **Consecuencias** | A alto volumen (>1M notificaciones/día), considerar particionamiento por fecha o migrar el histórico a un data lake |

### ADR-004: Lenguaje y Framework — TypeScript + Node.js

| Campo | Valor |
|---|---|
| **Estado** | Propuesto |
| **Contexto** | El servicio debe ser productivo para el equipo de desarrollo existente |
| **Decisión** | TypeScript + Node.js + Fastify |
| **Rationale** | Fastify ofrece mejor rendimiento que Express (validación schema integrada con Ajv, serialización rápida). TypeScript da type safety en las interfaces de dominio. El equipo ya tiene expertise en este stack. Alternativa Go sería más performante pero añade complejidad de lenguaje nuevo |
| **Consecuencias** | Si el throughput requerido supera las capacidades de Node.js en Fase 3, evaluar migración del processor a Go manteniendo la misma interfaz de cola |

### ADR-005: Idempotencia — CorrelationId como Clave de Deduplicación

| Campo | Valor |
|---|---|
| **Estado** | Propuesto |
| **Contexto** | Los microservicios pueden reenviar la misma notificación por fallos de red. No debemos generar doble entrega |
| **Decisión** | CorrelationId UUID como clave de idempotencia con constraint UNIQUE en PostgreSQL |
| **Rationale** | Simple, sin estado distribuido adicional. El gateway valida que el correlationId no exista en la tabla de notificaciones antes de encolar. Si existe, retorna status "duplicate" sin reencolar. El publisher es responsable de generar y mantener el correlationId en sus reintentos |
| **Consecuencias** | La tabla de notificaciones crece con el tiempo. Necesitamos política de retención (ej: archivar a cold storage después de 90 días) |

---

## 7. Configuración de Infraestructura — MVP

### 7.1 Variables de Entorno

```yaml
# Required
NOTIFICATION_EMAIL_PROVIDER: "sendgrid"
NOTIFICATION_EMAIL_API_KEY: "<api-key>"
NOTIFICATION_EMAIL_FROM: "noreply@domain.com"

NOTIFICATION_SMS_PROVIDER: "twilio"
NOTIFICATION_SMS_ACCOUNT_SID: "<sid>"
NOTIFICATION_SMS_AUTH_TOKEN: "<token>"
NOTIFICATION_SMS_FROM: "+1234567890"

RABBITMQ_URL: "amqp://localhost:5672"
RABBITMQ_EXCHANGE: "notifications.exchange"
RABBITMQ_QUEUE_HIGH: "notifications.queue.high"
RABBITMQ_QUEUE_MEDIUM: "notifications.queue.medium"
RABBITMQ_QUEUE_LOW: "notifications.queue.low"

DATABASE_URL: "postgresql://user:pass@localhost:5432/notifications"

# Optional
METRICS_ENABLED: "true"
METRICS_PORT: "9090"

MAX_RETRIES_HIGH: "5"
MAX_RETRIES_MEDIUM: "5"
MAX_RETRIES_LOW: "3"

RATE_LIMIT_LOW_MSG_PER_MIN: "100"

TEMPLATE_STORAGE_PATH: "./templates"
```

### 7.2 Docker Compose (Desarrollo)

```yaml
version: "3.8"

services:
  notification-service:
    build: .
    ports:
      - "3000:3000"
      - "9090:9090"  # metrics
    environment:
      RABBITMQ_URL: amqp://rabbitmq:5672
      DATABASE_URL: postgresql://notify:notify@postgres:5432/notifications
      NOTIFICATION_EMAIL_PROVIDER: sendgrid
      NOTIFICATION_EMAIL_API_KEY: ${SENDGRID_API_KEY}
      NOTIFICATION_SMS_PROVIDER: twilio
      NOTIFICATION_SMS_ACCOUNT_SID: ${TWILIO_ACCOUNT_SID}
      NOTIFICATION_SMS_AUTH_TOKEN: ${TWILIO_AUTH_TOKEN}
    depends_on:
      rabbitmq:
        condition: service_healthy
      postgres:
        condition: service_healthy

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"  # management UI
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 5s
      timeout: 3s
      retries: 5

  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: notify
      POSTGRES_PASSWORD: notify
      POSTGRES_DB: notifications
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U notify"]
      interval: 5s
      timeout: 3s
      retries: 5
```

---

## 8. Esquema de Base de Datos — MVP

### 8.1 Tabla: notifications

```sql
CREATE TABLE notifications (
    correlation_id    UUID PRIMARY KEY,
    channel           VARCHAR(10) NOT NULL CHECK (channel IN ('email', 'sms', 'push')),
    priority          VARCHAR(10) NOT NULL CHECK (priority IN ('high', 'medium', 'low')),
    template_code     VARCHAR(100) NOT NULL,
    template_version  VARCHAR(20),
    recipient_type    VARCHAR(20) NOT NULL,
    recipient_value   VARCHAR(500) NOT NULL,
    payload           JSONB NOT NULL DEFAULT '{}',
    metadata          JSONB NOT NULL DEFAULT '{}',
    status            VARCHAR(20) NOT NULL DEFAULT 'accepted'
                      CHECK (status IN ('accepted', 'enqueued', 'dispatched', 'sent', 'delivered', 'failed', 'retried', 'dead_lettered', 'rejected', 'duplicate')),
    attempt_number    INTEGER NOT NULL DEFAULT 0,
    provider_message_id VARCHAR(200),
    last_error_code   VARCHAR(50),
    last_error_message TEXT,
    scheduled_at      TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_status ON notifications (status);
CREATE INDEX idx_notifications_priority_channel ON notifications (priority, channel);
CREATE INDEX idx_notifications_created_at ON notifications (created_at DESC);
CREATE INDEX idx_notifications_dead_letter ON notifications (status) WHERE status = 'dead_lettered';
```

### 8.2 Tabla: templates

```sql
CREATE TABLE templates (
    id              SERIAL PRIMARY KEY,
    code            VARCHAR(100) NOT NULL,
    version         VARCHAR(20) NOT NULL,
    channel         VARCHAR(10) NOT NULL CHECK (channel IN ('email', 'sms', 'push')),
    subject_template TEXT,
    body_template   TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(100),
    UNIQUE (code, version, channel)
);

CREATE INDEX idx_templates_active ON templates (code, channel) WHERE is_active = true;
```

### 8.3 Tabla: notification_events (audit log)

```sql
CREATE TABLE notification_events (
    id                BIGSERIAL PRIMARY KEY,
    correlation_id    UUID NOT NULL REFERENCES notifications(correlation_id),
    event_type        VARCHAR(30) NOT NULL,
    channel           VARCHAR(10) NOT NULL,
    priority          VARCHAR(10) NOT NULL,
    attempt_number    INTEGER,
    provider          VARCHAR(50),
    provider_message_id VARCHAR(200),
    error_code        VARCHAR(50),
    error_message     TEXT,
    duration_ms       INTEGER,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_events_correlation_id ON notification_events (correlation_id);
CREATE INDEX idx_events_created_at ON notification_events (created_at DESC);
```

---

## 9. API REST — Especificación MVP

### 9.1 Endpoints

| Método | Path | Descripción | Status Code |
|---|---|---|---|
| `POST` | `/api/v1/notifications` | Publicar una notificación | 202, 400, 409, 429 |
| `GET` | `/api/v1/notifications/:correlationId` | Consultar estado de una notificación | 200, 404 |
| `GET` | `/api/v1/notifications/dead-letter` | Listar notificaciones en DLQ | 200 |
| `POST` | `/api/v1/notifications/dead-letter/:correlationId/retry` | Reintentar desde DLQ | 202, 404, 409 |
| `GET` | `/api/v1/health` | Health check | 200 |
| `GET` | `/api/v1/metrics` | Prometheus metrics endpoint | 200 |

### 9.2 POST /api/v1/notifications — Request

```http
POST /api/v1/notifications
Content-Type: application/json
X-Source-Service: auth-service

{
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "channel": "email",
  "priority": "high",
  "templateCode": "otp.verification",
  "templateVersion": "1.0.0",
  "recipient": {
    "type": "email",
    "value": "ivan@example.com"
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
```

### 9.3 POST /api/v1/notifications — Response

```http
HTTP/1.1 202 Accepted
Content-Type: application/json
X-Correlation-Id: 550e8400-e29b-41d4-a716-446655440000

{
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "accepted",
  "acceptedAt": "2026-05-18T18:30:00.000Z",
  "estimatedDelivery": "2026-05-18T18:30:00.100Z"
}
```

### 9.4 GET /api/v1/notifications/:correlationId — Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "channel": "email",
  "priority": "high",
  "templateCode": "otp.verification",
  "status": "sent",
  "attemptNumber": 1,
  "provider": "sendgrid",
  "providerMessageId": "sg-msg-abc123",
  "createdAt": "2026-05-18T18:30:00.000Z",
  "updatedAt": "2026-05-18T18:30:00.085Z",
  "events": [
    { "type": "accepted", "timestamp": "2026-05-18T18:30:00.000Z" },
    { "type": "enqueued", "timestamp": "2026-05-18T18:30:00.005Z" },
    { "type": "dispatched", "timestamp": "2026-05-18T18:30:00.050Z" },
    { "type": "sent", "timestamp": "2026-05-18T18:30:00.085Z" }
  ]
}
```

---

## 10. Métricas — MVP

### 10.1 Métricas Expuestas (Prometheus)

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `notification_enqueue_total` | Counter | `priority`, `channel` | Total notificaciones encoladas |
| `notification_delivery_total` | Counter | `priority`, `channel`, `status` | Total intentos de entrega (sent/failed) |
| `notification_delivery_duration_ms` | Histogram | `priority`, `channel` | Latencia de entrega (p50/p95/p99) |
| `notification_enqueue_latency_ms` | Histogram | `priority` | Tiempo entre aceptación y dispatch |
| `notification_retry_total` | Counter | `priority`, `channel`, `error_code` | Total reintentos por tipo de error |
| `notification_dead_letter_total` | Counter | `priority`, `channel` | Total mensajes enviados a DLQ |
| `notification_queue_depth` | Gauge | `priority` | Mensajes pendientes por cola |
| `notification_template_render_duration_ms` | Histogram | `channel` | Latencia de renderizado de templates |

### 10.2 Alertas Sugeridas

| Alerta | Condición | Severidad |
|---|---|---|
| `HighPriorityQueueDepth` | `notification_queue_depth{priority="high"} > 1000` por 2min | Critical |
| `DeliveryFailureRate` | `rate(notification_delivery_total{status="failed"}[5m]) / rate(notification_delivery_total[5m]) > 0.05` | Warning |
| `DeadLetterGrowth` | `increase(notification_dead_letter_total[15m]) > 50` | Warning |
| `HighLatency` | `histogram_quantile(0.99, notification_delivery_duration_ms) > 5000` | Warning |

---

## 11. Plan de Pruebas — MVP

| Tipo | Scope | Herramienta |
|---|---|---|
| **Unitarias** | Servicios de dominio, use cases, validación de schemas | Jest + TypeScript |
| **Integración** | Repositorios contra RabbitMQ, PostgreSQL, proveedores (mocks) | Testcontainers |
| **E2E** | Flujo completo: POST → queue → dispatch → provider mock → status event | Supertest + Testcontainers |
| **Carga** | Throughput por cola, latencia p99 bajo pico | k6 |
| **Resiliencia** | Caída de proveedor, saturación de cola baja, recovery | Chaos testing manual |

---

## 12. Checklist de Implementación — Fase 1

- [ ] **Infraestructura base**: Docker Compose con RabbitMQ + PostgreSQL
- [ ] **Schema de BD**: Migraciones para notifications, templates, notification_events
- [ ] **Dominio**: Value objects, entidades, interfaces de repositorio, errores de dominio
- [ ] **API REST**: POST /notifications con validación JSON Schema + GET /notifications/:id
- [ ] **Queue Setup**: Exchange + 3 colas + 3 DLQs con configuración de prioridad
- [ ] **Template Engine**: Handlebars + ITemplateRepository + templates por defecto (OTP)
- [ ] **Email Provider**: SendGridEmailRepository con send()
- [ ] **SMS Provider**: TwilioSMSRepository con send()
- [ ] **Processor**: Consumer de las 3 colas con dispatch a canal correspondiente
- [ ] **Retry Logic**: Backoff exponencial + dead-letter después de 5 intentos
- [ ] **Métricas**: Prometheus metrics + endpoint /metrics
- [ ] **Idempotencia**: CorrelationId UNIQUE + detección de duplicados
- [ ] **Health Check**: GET /health con verificación de RabbitMQ + PostgreSQL
- [ ] **Tests**: Unitarios + integración + E2E mínimo
- [ ] **Documentación**: OpenAPI spec + README de integración

---

## 13. Notas para el PM (John)

Este Technical Spec está diseñado para que puedas usarlo como base del PRD. Los puntos clave que necesitas reflejar:

1. **Los contratos de eventos** (Sección 2) son la API pública del sistema — cualquier cambio requiere versionado y comunicación a los equipos consumidores.
2. **Las interfaces de repositorio** (Sección 3) definen los límites de abstracción — el PRD debe mencionar que cambiar de proveedor es una operación de configuración, no de código.
3. **La estrategia de colas** (Sección 4) es la respuesta técnica al problema de priorización — el PRD debe comunicar que las colas están aisladas físicamente, no solo lógicamente.
4. **Los ADRs** (Sección 6) documentan las decisiones con rationale — si el equipo quiere cambiar alguna (ej: SQS en vez de RabbitMQ), cada ADR tiene las consecuencias documentadas.
5. **El plan de pruebas** (Sección 11) y el **checklist** (Sección 12) pueden usarse directamente como criterios de aceptación del PRD.

---

*Documento generado por Winston (System Architect) — 2026-05-18*
*Fuente de verdad: 01-product-brief.md (Mary)*
*Próximo paso recomendado: John (PM) genera el PRD usando este spec como base técnica*
