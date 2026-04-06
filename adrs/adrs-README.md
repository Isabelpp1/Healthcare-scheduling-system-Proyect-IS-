# Architecture Decision Records (ADRs)

Esta carpeta documenta las decisiones de diseño significativas del sistema: el contexto que las motivó, las alternativas evaluadas y el razonamiento detrás de cada elección. Los ADRs no son resúmenes de lo que se eligió — son registros del **por qué**.

---

| ADR | Título | Estado |
|---|---|---|
| [ADR-001](ADR-001-deployment-model.md) | Modelo de Despliegue — Monolito Modular | Accepted |
| [ADR-002](ADR-002-communication-style.md) | Estilo de Comunicación — Síncrono vs Asíncrono | Accepted |
| [ADR-003](ADR-003-database-strategy.md) | Estrategia de Base de Datos — PostgreSQL con esquemas por contexto | Accepted |
| [ADR-004](ADR-004-authentication-authorization.md) | Autenticación y Autorización — JWT autogestionado con RBAC | Accepted |
| [ADR-005](ADR-005-api-design.md) | Diseño de API — REST externo, llamadas directas internas | Accepted |
| [ADR-006](ADR-006-event-bus.md) | Bus de Eventos — RabbitMQ | Accepted |
| [ADR-007](ADR-007-observability.md) | Estrategia de Observabilidad — Loki + Prometheus + Grafana + OpenTelemetry | Accepted |
| [ADR-008](ADR-008-caching.md) | Estrategia de Caché — Redis con invalidación por eventos | Accepted |
| [ADR-009](ADR-009-deployment-cicd.md) | Despliegue y CI/CD — Docker + GitHub Actions + Railway | Accepted |

---

Todos los ADRs son internamente consistentes: las decisiones se referencian entre sí y no se contradicen. Consulta las secciones "ADRs Relacionados" al final de cada documento para navegar entre decisiones conectadas.
