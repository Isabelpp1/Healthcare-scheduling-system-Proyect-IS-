# Diagramas de Arquitectura

Esta carpeta contiene los cuatro diagramas que comunican la arquitectura visualmente. Todos los diagramas usan sintaxis Mermaid embebida en Markdown para renderizarse directamente en GitHub, y son consistentes con los documentos de propuestas y los ADRs.

---

| Diagrama | Descripción |
|---|---|
| [01-system-context.md](01-system-context.md) | C4 Nivel 1: el sistema como caja única rodeada de los tres roles de usuario y los cuatro sistemas externos (Stripe, SendGrid, Twilio, FCM) |
| [02-bounded-context-map.md](02-bounded-context-map.md) | Mapa DDD de los 8 bounded contexts agrupados por Core / Supporting / Generic, con patrones de integración etiquetados en cada relación |
| [03-data-flow.md](03-data-flow.md) | Diagramas de secuencia de los flujos de registro, agendamiento y pago, y cancelación con reembolso — incluye happy path y caminos de fallo |
| [04-deployment.md](04-deployment.md) | Topología de despliegue: zonas de red, contenedores, data stores, broker de eventos, CDN y stack de observabilidad |

---

Cada diagrama incluye una sección escrita que explica qué muestra el diagrama y las decisiones de diseño notables. Los diagramas de secuencia tienen máximo 6 participantes por diagrama para mantener la legibilidad.
