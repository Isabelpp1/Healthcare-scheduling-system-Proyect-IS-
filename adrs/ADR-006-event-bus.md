# ADR-006: Bus de Eventos — RabbitMQ

| Campo | Valor |
|---|---|
| Estado | Accepted |
| Fecha | 2026-04-05 |
| Responsables | Isabel y Gabriela  |

---

## Contexto

El sistema utiliza comunicación asíncrona entre módulos para desacoplar efectos secundarios del flujo principal del usuario (definido en ADR-002). En desarrollo, este bus opera en memoria, pero en producción se necesita un broker externo que garantice que los eventos no se pierdan si el proceso se reinicia, si un consumidor está temporalmente caído, o si ocurre un fallo en el servidor.

Los eventos críticos del sistema incluyen `CitaSolicitada`, `PagoConfirmado`, `PagoFallido` y `CitaCancelada`. Si alguno de estos se pierde, el sistema queda en un estado inconsistente: una cita reservada sin orden de pago, o un pago confirmado sin cita confirmada. La durabilidad y las garantías de entrega del broker son por tanto un requisito no negociable.

El volumen esperado de mensajes es moderado: en un escenario de 1,000 citas diarias, el sistema procesa aproximadamente 6,000–8,000 eventos por día, con picos en horarios de mañana. No se trata de streaming de datos en tiempo real ni de millones de mensajes por segundo.

---

## Opciones Consideradas

### Opción 1: RabbitMQ

Broker de mensajería tradicional basado en el modelo de colas (AMQP). Los productores publican mensajes en exchanges, que los enrutan a colas según reglas de binding. Los consumidores reciben mensajes de sus colas y envían un acknowledgment explícito cuando los procesan correctamente.

| Pros | Contras |
|---|---|
| Garantías de entrega "at-least-once" con acknowledgments explícitos | No retiene mensajes históricos después de su consumo (no es un log inmutable) |
| Modelo de colas flexible con routing por tipo de evento (fanout, topic, direct) | Sin capacidad de replay de eventos pasados |
| Amplio soporte en el ecosistema Node.js (librería `amqplib`) | Menor throughput máximo comparado con Kafka |
| Dead-letter queues (DLQ) configurables por cola para mensajes fallidos | Configuración de exchanges y bindings tiene curva de aprendizaje inicial |
| Versión gestionada disponible en CloudAMQP (plan gratuito suficiente para desarrollo) | — |

### Opción 2: Apache Kafka

Plataforma de streaming distribuida basada en un log inmutable particionado. Los consumidores leen mensajes según su offset y pueden reproducir eventos pasados.

| Pros | Contras |
|---|---|
| Throughput extremadamente alto (millones de mensajes/segundo) | Operativamente complejo: requiere ZooKeeper/KRaft, particiones, replication factor |
| Log inmutable que permite replay de eventos históricos | Sobrecarga desproporcionada para ~8,000 eventos/día |
| Ideal para event sourcing y auditoría completa de estado | Latencia más alta que RabbitMQ en escenarios de baja carga |
| Retención configurable por días o semanas | Versión gestionada (Confluent Cloud) tiene costo significativo para equipos pequeños |

### Opción 3: AWS SQS / Google Cloud Pub/Sub

Servicios de cola completamente gestionados por proveedores cloud, sin infraestructura propia que operar.

| Pros | Contras |
|---|---|
| Cero operación: el proveedor gestiona disponibilidad y escalado | Acoplamiento fuerte al proveedor cloud (vendor lock-in) |
| SLAs de disponibilidad garantizados | Migrar de proveedor implica cambios en el código de producción |
| Integración nativa con otros servicios del mismo cloud | SQS requiere polling activo (no push), añadiendo complejidad al consumer |
| — | Datos del sistema transitan por servidores de terceros |

---

## Decisión

Se adopta **RabbitMQ** como broker de mensajería para la comunicación asíncrona entre módulos en producción.

Razones específicas al proyecto:

1. **Volumen moderado y predecible:** Con 8,000 eventos/día, RabbitMQ maneja la carga con amplia capacidad de reserva. Kafka está diseñado para cargas órdenes de magnitud superiores; usarlo aquí sería sobredimensionamiento operativo injustificado para un equipo de 4–8 personas.

2. **Garantías de entrega suficientes:** El sistema necesita "at-least-once delivery" con acknowledgments explícitos, que RabbitMQ provee de forma nativa. No se requiere replay de eventos históricos ni event sourcing, por lo que la ventaja principal de Kafka no aplica.

3. **Dead-letter queues para resiliencia:** RabbitMQ permite configurar colas de mensajes fallidos por cola de eventos. Si el handler de `PagoConfirmado` en Scheduling falla tres veces consecutivas, el mensaje va a la DLQ para revisión manual en lugar de perderse silenciosamente.

4. **Compatibilidad con el bus en memoria de desarrollo:** La interfaz `EventBusPort` definida en `shared/` tiene dos implementaciones: `InMemoryEventBus` para desarrollo y tests, y `RabbitMQEventBus` para producción. El cambio entre implementaciones no requiere modificar ningún módulo de dominio.

5. **Sin vendor lock-in:** A diferencia de SQS o Pub/Sub, RabbitMQ corre en cualquier proveedor cloud, on-premise o en Docker, lo cual es importante para una plataforma de salud que debe controlar dónde residen sus datos.

---

## Consecuencias

Habilitado por esta decisión:
- Los eventos críticos (`CitaSolicitada`, `PagoConfirmado`, `CitaCancelada`) son durables: persisten en disco aunque el proceso del monolito se reinicie.
- Un consumidor temporalmente caído recibirá los mensajes pendientes al reconectarse, sin pérdida de eventos.
- Es posible agregar nuevos consumidores de un evento existente sin modificar el productor ni los consumidores actuales.
- La DLQ permite auditar y reprocesar manualmente eventos fallidos sin perder información.

Prevenido por esta decisión:
- No es posible hacer replay de eventos históricos para reconstruir el estado del sistema desde cero. Si se necesitara en el futuro, requeriría migrar a Kafka o implementar un event store complementario.

Riesgos:

| Riesgo | Mitigación |
|---|---|
| RabbitMQ se convierte en punto único de fallo | Desplegar en modo cluster con al menos 2 nodos, o usar CloudAMQP con plan HA |
| Un handler procesa el mismo evento dos veces por reintento | Todos los event handlers deben ser idempotentes: verificar si el efecto ya fue aplicado antes de ejecutarlo |
| La DLQ crece sin ser atendida | Alerta en el stack de observabilidad cuando la DLQ supere 5 mensajes acumulados |

---

## ADRs Relacionados

- **ADR-001** — El monolito modular define el contexto donde el bus opera. En desarrollo usa implementación en memoria; en producción, RabbitMQ.
- **ADR-002** — Define qué flujos usan comunicación asíncrona y por tanto dependen de este broker.
- **ADR-003** — La base de datos única reduce la necesidad de sagas distribuidas; RabbitMQ coordina los efectos secundarios restantes.
- **ADR-007** — El stack de observabilidad monitorea el tamaño de las colas y las DLQ como SLOs operacionales clave.
