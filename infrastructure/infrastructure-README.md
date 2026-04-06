# Componentes de Infraestructura

Esta carpeta documenta cada componente de infraestructura del sistema: qué hace específicamente en este proyecto, por qué se necesita, qué tecnología se eligió y por qué, sus trade-offs honestos y cómo se conecta con el resto de la arquitectura.

---

| Componente | Tecnología | Descripción |
|---|---|---|
| [01-api-gateway.md](01-api-gateway.md) | Nginx | Punto de entrada único: rate limiting, terminación TLS y enrutamiento diferenciado de webhooks de Stripe |
| [02-cdn.md](02-cdn.md) | Cloudflare | Distribución de assets estáticos, protección DDoS en el borde y caché de respuestas HTTP públicas |
| [03-load-balancer.md](03-load-balancer.md) | Nginx upstream | Distribución de tráfico entre réplicas del monolito con health checks automáticos |
| [04-cache.md](04-cache.md) | Redis 7 | Caché de slots disponibles, sesiones JWT, rate limiting distribuido y cola de jobs BullMQ |
| [05-message-queue.md](05-message-queue.md) | RabbitMQ 3.x | Entrega garantizada de eventos de dominio con dead-letter queues y routing por topic exchange |
| [06-background-workers.md](06-background-workers.md) | Node.js + BullMQ | Proceso independiente para recordatorios de cita y generación de reportes periódicos |
| [07-primary-database.md](07-primary-database.md) | PostgreSQL 16 | Base de datos principal con esquemas separados por bounded context y réplica de lectura |
| [08-observability-stack.md](08-observability-stack.md) | Loki + Prometheus + Grafana + OpenTelemetry | Tres pilares de observabilidad: logs estructurados, métricas con SLOs y trazas distribuidas |

---

Todos los componentes se referencian en el Diagrama 4 (topología de despliegue). Consulta ese diagrama para ver cómo se conectan físicamente dentro de la VPC privada.
