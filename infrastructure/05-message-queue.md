# Componente: Message Queue / Event Bus — RabbitMQ

## 1. Responsabilidades

- **Entrega garantizada de eventos de dominio críticos:** Persiste eventos como `CitaSolicitada`, `PagoConfirmado`, `PagoFallido` y `CitaCancelada` en disco, asegurando que no se pierdan si el proceso del monolito se reinicia antes de que los consumidores los procesen.
- **Desacoplamiento entre módulos productores y consumidores:** El módulo `scheduling/` publica `CitaSolicitada` sin saber quién lo consume. `payments/` y `notifications/` reaccionan de forma independiente sin que `scheduling/` tenga referencia a ellos.
- **Dead-Letter Queue (DLQ) para eventos fallidos:** Cuando un handler falla tres veces consecutivas al procesar un evento (por ejemplo, Stripe no responde al intentar crear la orden), el mensaje se mueve a una cola de mensajes muertos para revisión manual en lugar de perderse silenciosamente.
- **Enrutamiento flexible por tipo de evento:** Usando exchanges de tipo `topic` en RabbitMQ, cada tipo de evento se enruta a las colas de los consumidores suscritos. Un nuevo módulo puede suscribirse a eventos existentes sin modificar el productor ni los consumidores actuales.
- **Soporte a la comunicación entre `apps/api` y `apps/worker`:** El proceso Worker consume jobs de BullMQ (almacenados en Redis), pero los eventos de dominio que disparan notificaciones asíncronas y actualizaciones de analítica fluyen a través de RabbitMQ, separando las colas de jobs de los eventos de negocio.

## 2. Por Qué Este Proyecto lo Necesita

El flujo de agendamiento y pago es el más crítico del sistema: cuando un paciente solicita una cita, el evento `CitaSolicitada` debe llegar a `payments/` para crear la orden de cobro. Si ese evento se pierde (por ejemplo, porque el proceso se reinicia durante la publicación), la cita queda en estado `SOLICITADA` indefinidamente sin que se genere la orden de pago, creando una inconsistencia en el sistema que requeriría intervención manual.

Un bus de eventos en memoria (la implementación usada en desarrollo) no tiene persistencia: si el proceso cae, todos los eventos en tránsito se pierden. Para un sistema de salud con flujos financieros, esta situación es inaceptable en producción. RabbitMQ resuelve esto almacenando los mensajes en disco hasta que el consumidor confirma explícitamente su procesamiento con un `ack`.

## 3. Elección de Tecnología

**Elección: RabbitMQ 3.x (self-hosted en contenedor Docker)**

| Dimensión | RabbitMQ *(Elección)* | Apache Kafka *(Alternativa A)* | AWS SQS *(Alternativa B)* |
|---|---|---|---|
| **Managed / Self-hosted** | Self-hosted (contenedor Docker) | Self-hosted o Confluent Cloud | Managed (AWS) |
| **Complejidad operativa** | Baja: imagen Docker oficial, dashboard de administración web incluido | Alta: requiere ZooKeeper o KRaft, particiones, replication factor, configuración de topics | Muy baja: sin infraestructura propia, configuración por consola o IaC |
| **Costo a nuestra escala** | $0 adicional (corre en el mismo VPS) | $0 self-hosted pero con overhead operativo alto; Confluent Cloud desde $60/mes | $0.40 por millón de mensajes + $0.09/GB de transferencia |
| **Feature diferenciador** | Dead-letter queues nativas, acknowledgments explícitos, routing flexible con exchanges, dashboard web para monitoreo de colas | Log inmutable con replay de eventos históricos, throughput de millones de mensajes/segundo, ideal para event sourcing | Serverless, escala infinita, integración nativa con el ecosistema AWS |

RabbitMQ es la elección correcta para el volumen esperado del sistema (~8,000 eventos/día). Kafka ofrece capacidades que el sistema no necesita: replay de eventos históricos y throughput de millones de mensajes por segundo. Operar Kafka requiere conocimiento especializado que un equipo de 4–8 personas sin DevOps dedicado no debería invertir. AWS SQS requeriría que todo el stack esté en AWS y su modelo de polling activo añade complejidad al código del consumidor.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Dead-letter queues nativas: los mensajes fallidos no se pierden, quedan en la DLQ para revisión | RabbitMQ no retiene mensajes una vez consumidos: no es posible hacer replay de eventos históricos |
| Acknowledgments explícitos: un mensaje no se elimina de la cola hasta que el consumidor confirma su procesamiento exitoso | Sin un ingeniero dedicado, la configuración de alta disponibilidad (clustering) puede ser compleja |
| Dashboard web en el puerto 15672: visibilidad inmediata de colas, mensajes pendientes y DLQ sin herramientas externas | RabbitMQ puede convertirse en punto único de fallo si no se configura en modo cluster |
| El mismo contrato de interfaz (`EventBusPort`) permite cambiar la implementación sin modificar los módulos de dominio | Los handlers de eventos deben ser idempotentes: RabbitMQ garantiza "at-least-once delivery", no "exactly-once" |
| Routing por `topic` exchange permite que nuevos consumidores se suscriban sin tocar el código del productor | La memoria de RabbitMQ puede crecer si los consumidores no procesan mensajes a la velocidad de producción |

## 5. Integración

En el diagrama de despliegue (Diagrama 4), RabbitMQ se ubica en el Data Tier dentro de la VPC privada, accesible por `apps/api` y `apps/worker` pero sin exposición a internet. La arquitectura de exchanges y colas sigue este esquema:

```
Exchange: medical.events (tipo: topic)
    │
    ├── Routing key: scheduling.*  →  Cola: scheduling.payments
    │                                       scheduling.notifications
    │                                       scheduling.analytics
    │
    ├── Routing key: payments.*   →  Cola: payments.scheduling
    │                                       payments.notifications
    │                                       payments.analytics
    │
    └── Routing key: identity.*   →  Cola: identity.notifications
                                            identity.profiles
```

Cada cola tiene su Dead-Letter Exchange (DLX) configurado:
```
Cola: scheduling.payments  →  DLX: medical.dlx  →  Cola: dlq.scheduling.payments
```

La integración con el stack de observabilidad (Diagrama 4) se realiza exportando las métricas de RabbitMQ mediante el plugin `rabbitmq_prometheus`, que expone un endpoint `/metrics` consumido por Prometheus. Las alertas en Grafana se activan cuando cualquier DLQ supera 5 mensajes acumulados, notificando al equipo de operaciones para revisar y reprocesar manualmente los eventos fallidos.

La interfaz `EventBusPort` en `shared/infrastructure/event-bus/` abstrae RabbitMQ del código de dominio:

