---
stepsCompleted: [1]
inputDocuments:
  - "_bmad-output/02-prd.md"
  - "_bmad-output/03-technical-spec.md"
---

# Sistema de Notificaciones Omnicanal - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for Sistema de Notificaciones Omnicanal, decomposing the requirements from the PRD, UX Design if it exists, and Architecture requirements into implementable stories.

## Requirements Inventory

### Functional Requirements

FR-001: El sistema acepta publicaciones vía POST /api/v1/notifications con validación JSON Schema
FR-002: El sistema genera correlationId UUID v4 si el publisher no lo proporciona
FR-003: El sistema valida el schema antes de encolar; rechaza con 400 si es inválido
FR-004: El sistema encola en la cola física correspondiente según prioridad y canal
FR-005: El sistema renderiza templates Handlebars con variables del payload
FR-006: El sistema envía emails vía SendGrid usando IEmailRepository
FR-007: El sistema envía SMS vía Twilio usando ISMSRepository
FR-008: El sistema implementa reintentos con backoff exponencial (máx 5)
FR-009: El sistema mueve a DLQ las notificaciones que exceden máx reintentos
FR-010: El sistema detecta correlationId duplicado y responde 409 sin reencolar
FR-011: El sistema permite consultar estado por correlationId con historial de eventos
FR-012: El sistema expone métricas Prometheus en /api/v1/metrics
FR-013: El sistema expone health check en /api/v1/health
FR-014: El sistema permite listar notificaciones en DLQ vía GET /api/v1/notifications/dead-letter
FR-015: El sistema permite reintentar una notificación desde DLQ vía POST /api/v1/notifications/dead-letter/:id/retry
FR-016: El sistema registra cada transición de estado en notification_events
FR-017: El sistema soporta templates versionados con versión activa por defecto
FR-018: El sistema aplica rate limiting de 100 msg/min a la cola de baja prioridad

### NonFunctional Requirements

RNF-001: Latencia de encolamiento (prioridad Alta) p99 < 100ms
RNF-002: Disponibilidad del servicio 99.9% uptime mensual
RNF-003: Escalabilidad ante picos 10x sin degradación de prioridad Alta
RNF-004: Encriptación de datos sensibles TLS 1.3+ en tránsito, AES-256 en reposo
RNF-005: Tasa de entrega (prioridad Alta) > 99.5%
RNF-006: Idempotencia garantizada — 0 entregas duplicadas por correlationId

### Additional Requirements

- **Stack**: TypeScript + Node.js + Fastify (ADR-004)
- **Message Broker**: RabbitMQ con topic exchange `notifications.exchange` (ADR-001)
- **Template Engine**: Handlebars logic-less (ADR-002)
- **Database**: PostgreSQL 16+ con tablas notifications, templates, notification_events (ADR-003)
- **Clean Architecture**: Capa de dominio pura con interfaces de repositorio; proveedores como plugins en infraestructura
- **Cola Topology**: 3 colas físicas separadas (high/medium/low) con consumers dedicados y DLQ por cola
- **Rate Limiting**: Cola low limitada a 100 msg/min; high y medium sin límite
- **Backoff Exponencial**: 1s, 2s, 4s, 8s (prioridad Alta: mitad — 0.5s, 1s, 2s, 4s); máx 5 intentos
- **Idempotencia**: CorrelationId UUID con constraint UNIQUE en PostgreSQL (ADR-005)
- **Docker Compose**: RabbitMQ 3-management + PostgreSQL 16-alpine + notification-service
- **JSON Schemas**: NotificationRequest, NotificationResponse, NotificationStatusEvent (draft-07)
- **Métricas Prometheus**: 8 métricas con labels específicos (enqueue_total, delivery_total, delivery_duration_ms, enqueue_latency_ms, retry_total, dead_letter_total, queue_depth, template_render_duration_ms)
- **Alertas**: 4 alertas configuradas (HighPriorityQueueDepth, DeliveryFailureRate, DeadLetterGrowth, HighLatency)
- **Plan de Pruebas**: Unitarias (Jest), Integración (Testcontainers), E2E (Supertest + Testcontainers), Carga (k6), Resiliencia (chaos manual)
- **Estructura de Proyecto**: domain/ports, application/services+use-cases, infrastructure/providers+queue+templates+metrics+persistence, api/controllers+middleware+schemas

### UX Design Requirements

_No UX Design document exists — internal API service with no user-facing interface._

### FR Coverage Map

| FR ID | Epic | Story(s) |
|---|---|---|
| FR-001 | E1 | 1.1, 1.2 |
| FR-002 | E1 | 1.1 |
| FR-003 | E1 | 1.2 |
| FR-004 | E1 | 1.3 |
| FR-005 | E4 | 4.1, 4.2 |
| FR-006 | E1 | 1.4 |
| FR-007 | E1 | 1.5 |
| FR-008 | E2 | 2.1 |
| FR-009 | E2 | 2.2 |
| FR-010 | E3 | 3.1 |
| FR-011 | E3 | 3.2 |
| FR-012 | E5 | 5.1 |
| FR-013 | E5 | 5.2 |
| FR-014 | E2 | 2.3 |
| FR-015 | E2 | 2.4 |
| FR-016 | E3 | 3.3 |
| FR-017 | E4 | 4.1, 4.3 |
| FR-018 | E1 | 1.3 |

## Epic List

| Epic | Goal | Priority | Stories |
|---|---|---|---|
| **E1**: Infraestructura Base y Publicación | Desplegar infraestructura (RabbitMQ, PostgreSQL), definir dominio, y habilitar publicación de notificaciones vía REST con encolamiento por prioridad | P0 | 1.1 – 1.5 |
| **E2**: Resiliencia y Dead-Letter Management | Implementar reintentos con backoff exponencial, dead-letter queue, y gestión operativa de fallos | P0 | 2.1 – 2.4 |
| **E3**: Idempotencia y Trazabilidad | Garantizar idempotencia por correlationId, consulta de estado con historial de eventos, y auditoría completa | P0 | 3.1 – 3.3 |
| **E4**: Motor de Templates | Implementar renderizado Handlebars con templates versionados en PostgreSQL y gestión de plantillas por canal | P0 | 4.1 – 4.3 |
| **E5**: Observabilidad y Operación | Exponer métricas Prometheus, health check, y alertas operativas | P1 | 5.1 – 5.2 |
