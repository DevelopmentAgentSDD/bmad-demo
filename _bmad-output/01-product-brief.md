---
document_type: "Product Brief"
version: "1.0.0"
status: "draft"
created: "2026-05-18"
author: "Mary (Business Analyst)"
project_name: "Sistema de Notificaciones Omnicanal"
---

# Product Brief: Sistema de Notificaciones Omnicanal

## 1. Resumen Ejecutivo

### 1.1 Problema

Los equipos de desarrollo sufren **acoplamiento severo y fragmentación** en la capa de notificaciones. Cada microservicio implementa su propia lógica de conexión con proveedores de Email, SMS y Push, generando:

- **Duplicación de código** y esfuerzo de mantenimiento distribuido
- **Ausencia de control centralizado** de reintentos, rate limiting y circuit breakers
- **Inversión de prioridad**: notificaciones críticas (OTPs, alertas de seguridad) se encolan detrás de campañas masivas de marketing
- **Cero visibilidad unificada** de tasas de entrega, fallos y latencia por canal
- **Vendor lock-in implícito**: cambiar de proveedor requiere modificar N servicios

### 1.2 Solución Propuesta

Un **servicio centralizado de notificaciones omnicanal** que actúe como plataforma interna, ofreciendo:

- **API unificada** con contratos de eventos claros para publicar notificaciones
- **Abstracción de proveedores** mediante interfaces limpias (Email, SMS, Push) intercambiables
- **Sistema de priorización** con colas diferenciadas (Alta/Media/Baja) que garantice SLA por tipo de mensaje
- **Visibilidad centralizada** de métricas de entrega, reintentos y fallos
- **Diseño emergente**: MVP simple, escalable incrementalmente, libre de sobreingeniería

### 1.3 Propuesta de Valor

| Stakeholder | Valor |
|---|---|
| **Dev Teams** | Eliminan código boilerplate de notificaciones; integran con una sola API |
| **Arquitectos** | Control centralizado de patrones de resiliencia, observabilidad y gobernanza |
| **Producto/Operaciones** | Garantía de que alertas críticas llegan en tiempo real; visibilidad de delivery |
| **Negocio** | Reducción de costes operativos; capacidad de cambiar proveedores sin impacto en servicios |

---

## 2. Alcance

### 2.1 In-Scope (MVP)

| # | Capacidad | Descripción |
|---|---|---|
| C1 | **API de Publicación** | Endpoint REST y/o contrato de eventos para enviar notificaciones |
| C2 | **Canal Email** | Integración con al menos 1 proveedor SMTP (ej: SendGrid, SES) |
| C3 | **Canal SMS** | Integración con al menos 1 proveedor SMS (ej: Twilio) |
| C4 | **Canal Push** | Integración con al menos 1 proveedor Push (ej: FCM, APNs) |
| C5 | **Priorización** | 3 niveles de cola: Alta (OTP/seguridad), Media (transaccional), Baja (marketing/resumen) |
| C6 | **Reintentos** | Política configurable de retry con backoff exponencial por canal |
| C7 | **Observabilidad** | Métricas de entrega, fallos, latencia y estado de colas |
| C8 | **Template Engine** | Gestión centralizada de plantillas de notificación por canal |

### 2.2 Out-of-Scope (MVP)

| # | Elemento | Rationale |
|---|---|---|
| O1 | Portal de autogestión para equipos de marketing | Fase posterior; MVP se enfoca en devs como consumidores |
| O2 | A/B testing de contenido de notificaciones | No bloquea el problema central de acoplamiento |
| O3 | Canal WhatsApp u otros canales OTT | Se evalúa tras validar los 3 canales base |
| O4 | Orquestación de journeys multicanal complejos | Fuera del scope de notificación puntual |
| O5 | Preferencias de usuario (opt-in/opt-out granular) | Se aborda en fase 2 tras establecer delivery confiable |

---

## 3. Usuarios y Stakeholders

### 3.1 Usuarios Primarios

| Rol | Perfil | Necesidad |
|---|---|---|
| **Desarrollador Backend** | Ing. de software que consume la API desde microservicios | Enviar notificaciones con una llamada simple, sin gestionar proveedores |
| **Arquitecto de Plataforma** | Define estándares técnicos y gobernanza | Control centralizado de resiliencia, observabilidad y costes |

### 3.2 Usuarios Secundarios

| Rol | Perfil | Necesidad |
|---|---|---|
| **Product Manager** | Responsable de experiencia del cliente | Confianza en que notificaciones críticas llegan |
| **Equipo de Operaciones** | SRE/DevOps | Dashboards de salud del sistema, alertas de fallos |

### 3.3 Stakeholders Indirectos

| Rol | Interés |
|---|---|
| **Cliente Final** | Recibe notificaciones correctas, en el canal correcto, en tiempo |
| **Equipo de Seguridad** | Cumplimiento de protección de datos en notificaciones (PII en OTPs) |
| **Finanzas** | Optimización de costes por proveedor y volumen |

---

## 4. Requisitos Funcionales

### 4.1 Publicación de Notificaciones

| ID | Requisito | Prioridad |
|---|---|---|
| RF-001 | El sistema debe aceptar solicitudes de notificación vía API REST | **Alta** |
| RF-002 | El sistema debe aceptar solicitudes de notificación vía contrato de eventos (message broker) | **Alta** |
| RF-003 | Cada notificación debe especificar: canal(es) destino, prioridad, template, y payload de datos | **Alta** |
| RF-004 | El sistema debe soportar envío multicanal simultáneo (ej: Email + SMS para OTP) | **Media** |
| RF-005 | El sistema debe validar el schema de entrada antes de encolar | **Alta** |

### 4.2 Priorización y Encolamiento

| ID | Requisito | Prioridad |
|---|---|---|
| RF-010 | El sistema debe implementar 3 niveles de prioridad: Alta, Media, Baja | **Alta** |
| RF-011 | Los mensajes de prioridad Alta deben procesarse antes que cualquier mensaje de prioridad Media o Baja | **Alta** |
| RF-012 | El sistema debe prevenir que mensajes de baja prioridad bloqueen colas de alta prioridad | **Alta** |
| RF-013 | El sistema debe permitir configuración de rate limiting por nivel de prioridad | **Media** |

### 4.3 Gestión de Proveedores

| ID | Requisito | Prioridad |
|---|---|---|
| RF-020 | El sistema debe abstraer proveedores de Email, SMS y Push mediante interfaces intercambiables | **Alta** |
| RF-021 | El sistema debe permitir configurar múltiples proveedores por canal con fallback automático | **Media** |
| RF-022 | El cambio de proveedor no debe requerir modificaciones en los microservicios consumidores | **Alta** |

### 4.4 Resiliencia

| ID | Requisito | Prioridad |
|---|---|---|
| RF-030 | El sistema debe implementar reintentos con backoff exponencial configurable por canal | **Alta** |
| RF-031 | El sistema debe implementar circuit breaker por proveedor | **Alta** |
| RF-032 | Los mensajes que excedan el máximo de reintentos deben ir a una cola de dead-letter | **Alta** |
| RF-033 | El sistema debe garantizar al-menos-una-entrega (at-least-once) para prioridad Alta | **Alta** |

### 4.5 Templates

| ID | Requisito | Prioridad |
|---|---|---|
| RF-040 | El sistema debe gestionar plantillas de notificación versionadas por canal | **Alta** |
| RF-041 | Las plantillas deben soportar variables dinámicas inyectadas desde el payload | **Alta** |
| RF-042 | El sistema debe permitir preview de templates antes de publicación | **Baja** |

### 4.6 Observabilidad

| ID | Requisito | Prioridad |
|---|---|---|
| RF-050 | El sistema debe exponer métricas de: tasa de entrega, latencia p95/p99, fallos por canal | **Alta** |
| RF-051 | El sistema debe registrar cada intento de envío con trazabilidad completa (correlation ID) | **Alta** |
| RF-052 | El sistema debe emitir eventos de estado (sent, failed, retried, dead-lettered) | **Alta** |
| RF-053 | El sistema debe proveer un dashboard de salud de colas y proveedores | **Media** |

---

## 5. Requisitos No Funcionales

| ID | Requisito | Métrica |
|---|---|---|
| RNF-001 | **Latencia**: Notificaciones de prioridad Alta deben encolarse en < 100ms | p99 < 100ms |
| RNF-002 | **Disponibilidad**: El servicio debe mantener 99.9% uptime | SLA mensual |
| RNF-003 | **Escalabilidad**: Soportar picos de 10x volumen sin degradación de prioridad Alta | Auto-scaling |
| RNF-004 | **Seguridad**: Datos sensibles en tránsito y en reposo encriptados | TLS 1.3+, AES-256 |
| RNF-005 | **Auditoría**: Todo cambio de configuración debe ser trazable | Audit log inmutable |
| RNF-006 | **Idempotencia**: Envío duplicado con mismo correlation ID no debe generar doble entrega | Deduplicación |

---

## 6. Arquitectura de Referencia (Diseño Emergente)

### 6.1 Principios

| Principio | Aplicación |
|---|---|
| **Diseño Emergente** | MVP simple; complejidad se añade solo cuando el dominio la demanda |
| **Clean Architecture** | Interfaces de repositorio limpias; proveedores como plugins intercambiables |
| **Event-Driven** | Contrato de eventos como patrón primario de integración |
| **Fail-Safe** | Circuit breakers, retries, dead-letter queues desde el día 1 |

### 6.2 Componentes MVP

```
┌─────────────────────────────────────────────────────┐
│                 Microservicios Consumidores          │
│         (publican vía REST o Message Broker)         │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              Notification Gateway API                │
│         (validación, routing, correlation ID)        │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│          Priority Queue Manager                      │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│    │  HIGH   │  │  MEDIUM │  │   LOW   │            │
│    │  OTP    │  │ Transac │  │ Mktg    │            │
│    └────┬────┘  └────┬────┘  └────┬────┘            │
└─────────┼────────────┼────────────┼─────────────────┘
          │            │            │
          ▼            ▼            ▼
┌─────────────────────────────────────────────────────┐
│              Channel Dispatchers                     │
│    ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│    │  Email   │  │   SMS    │  │   Push   │         │
│    │ Provider │  │ Provider │  │ Provider │         │
│    └──────────┘  └──────────┘  └──────────┘         │
└─────────────────────────────────────────────────────┘
          │            │            │
          ▼            ▼            ▼
┌─────────────────────────────────────────────────────┐
│         External Providers (SendGrid, Twilio, FCM)   │
└─────────────────────────────────────────────────────┘
```

### 6.3 Decisiones Técnicas (MVP)

| Decisión | Opción | Rationale |
|---|---|---|
| Message Broker | RabbitMQ o AWS SQS | Soporte nativo de múltiples colas con prioridad |
| API | REST + OpenAPI | Familiaridad del equipo; event bridge como complemento |
| Template Engine | Handlebars o similar | Lenguaje simple, sin lógica compleja en templates |
| Observabilidad | OpenTelemetry + métricas Prometheus | Estándar; no vendor lock-in |

---

## 7. Criterios de Éxito

| # | KPI | Meta |
|---|---|---|
| K1 | Tasa de entrega (prioridad Alta) | > 99.5% |
| K2 | Latencia de encolamiento (prioridad Alta) | p99 < 100ms |
| K3 | Reducción de código duplicado en microservicios | > 80% del boilerplate eliminado |
| K4 | Tiempo para cambiar de proveedor | < 1 día de configuración (sin deploy de servicios) |
| K5 | Visibilidad de fallos | 100% de intentos trazables con correlation ID |
| K6 | Adopción interna | > 50% de microservicios migrados en 90 días post-lanzamiento |

---

## 8. Riesgos y Mitigaciones

| # | Riesgo | Impacto | Probabilidad | Mitigación |
|---|---|---|---|---|
| R1 | Un proveedor externo sufre outage prolongado | Alto | Media | Fallback a proveedor secundario; dead-letter con replay |
| R2 | Cola de baja prioridad satura recursos y afecta alta | Alto | Media | Límites de throughput por cola; aislamiento de recursos |
| R3 | Equipos resisten migración desde sus implementaciones actuales | Medio | Alta | SDK cliente con drop-in replacement; documentación clara |
| R4 | Templates se convierten en cuello de botella de gestión | Medio | Media | Self-service en fase 2; MVP con templates gestionados por equipo plataforma |
| R5 | Costes de proveedores se disparan con volumen no controlado | Alto | Media | Rate limiting por consumidor; alertas de presupuesto |

---

## 9. Roadmap Propuesto

### Fase 1: MVP (Semanas 1-6)

- API REST de publicación con validación
- Canal Email (1 proveedor)
- Canal SMS (1 proveedor)
- 3 niveles de prioridad con colas separadas
- Reintentos con backoff exponencial
- Métricas básicas de entrega y fallos
- Templates versionados simples

### Fase 2: Resiliencia Avanzada (Semanas 7-10)

- Canal Push (FCM/APNs)
- Circuit breakers por proveedor
- Fallback automático a proveedor secundario
- Dead-letter queue con replay manual
- Dashboard de observabilidad
- Correlation ID end-to-end

### Fase 3: Adopción y Escala (Semanas 11-16)

- SDK cliente para lenguajes principales
- Migración guiada de microservicios existentes
- Rate limiting y quotas por consumidor
- Preferencias de usuario (opt-in/opt-out)
- Portal de gestión de templates (self-service)

### Fase 4: Omnicanal Avanzado (Semanas 17+)

- Canal WhatsApp / OTT
- Orquestación de journeys multicanal
- A/B testing de contenido
- Optimización inteligente de canal y timing

---

## 10. Supuestos y Dependencias

| # | Supuesto | Validación |
|---|---|---|
| S1 | Los equipos de desarrollo prefieren integración vía SDK sobre llamadas REST directas | Validar con 2-3 equipos piloto |
| S2 | El message broker elegido soporta priorización de colas de forma nativa | POC técnica en semana 1 |
| S3 | Los proveedores externos (SendGrid, Twilio, FCM) tienen SLAs compatibles con nuestros requisitos de prioridad Alta | Revisar contratos de SLA |
| S4 | Existe infraestructura de observabilidad (Prometheus/Grafana) reutilizable | Confirmar con equipo de plataforma |

---

## 11. Glosario

| Término | Definición |
|---|---|
| **Omnicanal** | Capacidad de enviar notificaciones por múltiples canales (Email, SMS, Push) desde un sistema unificado |
| **Prioridad Alta** | Notificaciones que requieren entrega inmediata: OTPs, alertas de seguridad, verificaciones |
| **Prioridad Media** | Notificaciones transaccionales del día a día: confirmaciones de pedido, estados de cuenta |
| **Prioridad Baja** | Notificaciones no urgentes: resúmenes semanales, campañas de marketing, newsletters |
| **Dead-Letter Queue (DLQ)** | Cola donde se depositan mensajes que no pudieron ser entregados tras agotar reintentos |
| **Correlation ID** | Identificador único que traza una notificación desde su publicación hasta su entrega o fallo |
| **Circuit Breaker** | Patrón que detiene temporalmente las llamadas a un proveedor que está fallando repetidamente |
| **Diseño Emergente** | Enfoque que prioriza simplicidad inicial y añade complejidad solo cuando el dominio la justifica |

---

*Documento generado por Mary (Business Analyst) — 2026-05-18*
*Próximo paso recomendado: Revisión con arquitecto (Winston) para validar decisiones técnicas del MVP*
