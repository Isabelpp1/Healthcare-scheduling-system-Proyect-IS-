# Sistema de Programación de Citas Médicas

Plataforma digital que conecta pacientes con profesionales de la salud, permitiendo agendar, gestionar y pagar consultas médicas en línea. El sistema atiende a cuatro roles: pacientes, médicos, recepcionistas y administradores, resolviendo la fragmentación y fricción del proceso tradicional de agendamiento telefónico.

---

## Enfoques Arquitectónicos Comparados

- **Monolito Modular:** Un único artefacto desplegable organizado en módulos con límites de dominio explícitos, comunicación interna vía bus de eventos y sin importaciones cruzadas directas. → [Ver comparación completa](proposals/01-high-level-architecture.md)
- **Microservicios:** Cada bounded context como servicio independiente con su propia base de datos, pipeline de despliegue y ciclo de vida. → [Ver comparación completa](proposals/01-high-level-architecture.md)

## Enfoque Recomendado

Se adoptó el **Monolito Modular** porque el equipo es de tamaño reducido (4–8 personas), las fronteras del dominio aún están estabilizándose y el costo operativo de microservicios sería desproporcionado para la carga esperada. → [Ver justificación completa](proposals/01-high-level-architecture.md#recomendación-monolito-modular)

---

## Navegación del Repositorio

### proposals/ — Propuestas de Arquitectura

| Documento | Descripción |
|---|---|
| [01-high-level-architecture.md](proposals/01-high-level-architecture.md) | Comparación de Monolito Modular vs Microservicios con tabla de dimensiones y recomendación justificada |
| [02-bounded-contexts.md](proposals/02-bounded-contexts.md) | Descomposición DDD en 8 bounded contexts con entidades, eventos de dominio y mapa de contextos |
| [03-service-module-decomposition.md](proposals/03-service-module-decomposition.md) | Árbol de directorios del monolito, descripción de cada módulo y reglas de enforcement de límites |
| [04-data-flow-and-interactions.md](proposals/04-data-flow-and-interactions.md) | 4 flujos end-to-end con diagramas de secuencia, happy path y caminos de fallo y compensación |

### adrs/ — Architecture Decision Records

| Documento | Decisión |
|---|---|
| [ADR-001-deployment-model.md](adrs/ADR-001-deployment-model.md) | Monolito Modular sobre Microservicios como modelo de despliegue inicial |
| [ADR-002-communication-style.md](adrs/ADR-002-communication-style.md) | Criterio mixto: síncrono para flujos del usuario, asíncrono para efectos secundarios |
| [ADR-003-database-strategy.md](adrs/ADR-003-database-strategy.md) | PostgreSQL única con esquemas separados por bounded context |
| [ADR-004-authentication-authorization.md](adrs/ADR-004-authentication-authorization.md) | JWT autogestionado con RBAC de 4 roles sobre proveedor de identidad externo |
| [ADR-005-api-design.md](adrs/ADR-005-api-design.md) | REST para APIs externas, llamadas directas entre módulos para comunicación interna |
| [ADR-006-event-bus.md](adrs/ADR-006-event-bus.md) | RabbitMQ como broker de eventos en producción sobre Kafka y SQS |
| [ADR-007-observability.md](adrs/ADR-007-observability.md) | Stack open source Loki + Prometheus + Grafana + OpenTelemetry |
| [ADR-008-caching.md](adrs/ADR-008-caching.md) | Redis con invalidación por eventos de dominio como estrategia de caché |
| [ADR-009-deployment-cicd.md](adrs/ADR-009-deployment-cicd.md) | Docker + GitHub Actions + Railway como pipeline de CI/CD |

### diagrams/ — Diagramas de Arquitectura

| Documento | Descripción |
|---|---|
| [01-system-context.md](diagrams/01-system-context.md) | Diagrama C4 Nivel 1: el sistema como caja única rodeada de actores y sistemas externos |
| [02-bounded-context-map.md](diagrams/02-bounded-context-map.md) | Mapa DDD de los 8 contextos agrupados por tipo de subdominio con patrones de integración |
| [03-data-flow.md](diagrams/03-data-flow.md) | Diagramas de secuencia de los flujos de registro, agendamiento, pago y cancelación |
| [04-deployment.md](diagrams/04-deployment.md) | Topología de despliegue: VPC, contenedores, data stores, observabilidad y zonas de red |

### infrastructure/ — Componentes de Infraestructura

| Documento | Componente |
|---|---|
| [01-api-gateway.md](infrastructure/01-api-gateway.md) | Nginx como punto de entrada único: rate limiting, TLS termination y routing de webhooks |
| [02-cdn.md](infrastructure/02-cdn.md) | Cloudflare: distribución de assets, protección DDoS y caché de respuestas públicas |
| [03-load-balancer.md](infrastructure/03-load-balancer.md) | Nginx upstream: distribución de tráfico entre réplicas del monolito con health checks |
| [04-cache.md](infrastructure/04-cache.md) | Redis: caché de slots, sesiones JWT, rate limiting distribuido y cola BullMQ |
| [05-message-queue.md](infrastructure/05-message-queue.md) | RabbitMQ: entrega garantizada de eventos con dead-letter queues y routing por topic |
| [06-background-workers.md](infrastructure/06-background-workers.md) | Worker process: jobs de recordatorios de cita y generación de reportes periódicos |
| [07-primary-database.md](infrastructure/07-primary-database.md) | PostgreSQL con esquemas por contexto, réplica de lectura y estrategia de migraciones |
| [08-observability-stack.md](infrastructure/08-observability-stack.md) | Loki + Prometheus + Grafana + OpenTelemetry: logs, métricas, trazas y SLOs |

---

## Estructura del Repositorio

```
healthcare-scheduling/
├── README.md                         
├── proposals/
│   ├── README.md
│   ├── 01-high-level-architecture.md
│   ├── 02-bounded-contexts.md
│   ├── 03-service-module-decomposition.md
│   └── 04-data-flow-and-interactions.md
├── adrs/
│   ├── README.md
│   ├── ADR-001-deployment-model.md
│   ├── ADR-002-communication-style.md
│   ├── ADR-003-database-strategy.md
│   ├── ADR-004-authentication-authorization.md
│   ├── ADR-005-api-design.md
│   ├── ADR-006-event-bus.md
│   ├── ADR-007-observability.md
│   ├── ADR-008-caching.md
│   └── ADR-009-deployment-cicd.md
├── diagrams/
│   ├── README.md
│   ├── 01-system-context.md
│   ├── 02-bounded-context-map.md
│   ├── 03-data-flow.md
│   └── 04-deployment.md
└── infrastructure/
    ├── README.md
    ├── 01-api-gateway.md
    ├── 02-cdn.md
    ├── 03-load-balancer.md
    ├── 04-cache.md
    ├── 05-message-queue.md
    ├── 06-background-workers.md
    ├── 07-primary-database.md
    └── 08-observability-stack.md
```
