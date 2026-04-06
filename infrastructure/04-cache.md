# Componente: Caché — Redis

## 1. Responsabilidades

- **Caché de slots disponibles:** Almacena el resultado del cálculo de disponibilidad por médico y fecha (`slots:{doctorId}:{date}`) con TTL de 5 minutos, evitando recalcular la disponibilidad contra PostgreSQL en cada consulta durante los picos de mañana.
- **Almacenamiento de sesiones de usuario:** Guarda los tokens de refresco activos y la lista negra de tokens JWT revocados (cuentas suspendidas), permitiendo verificación local sin consultar la base de datos en cada solicitud autenticada.
- **Cola de jobs para el Worker (BullMQ):** Actúa como broker de jobs persistentes para el proceso `apps/worker`, almacenando los recordatorios de cita y los jobs de generación de reportes pendientes de ejecución.
- **Rate limiting distribuido:** Mantiene contadores de solicitudes por IP y por `userId` que el API Gateway (Nginx) consulta para aplicar los límites definidos, garantizando que el rate limiting funcione correctamente incluso con múltiples réplicas del monolito.
- **Invalidación por eventos de dominio:** Cuando se publican eventos como `PagoConfirmado` o `CitaCancelada`, el handler correspondiente ejecuta `DEL slots:{doctorId}:{date}` en Redis para invalidar el caché del slot afectado, manteniendo consistencia con la base de datos.

## 2. Por Qué Este Proyecto lo Necesita

El endpoint de consulta de slots disponibles (`GET /api/scheduling/slots`) es el más frecuente del sistema: un paciente puede consultarlo múltiples veces antes de elegir un horario, y durante los picos de mañana, decenas de pacientes consultan los slots del mismo médico simultáneamente. Cada consulta sin caché ejecuta una query que cruza tres tablas (disponibilidad base, bloqueos de agenda y citas existentes), generando carga innecesaria en PostgreSQL.

Sin Redis, el sistema tampoco podría implementar rate limiting efectivo con múltiples réplicas del monolito: cada instancia mantendría sus propios contadores en memoria, permitiendo que un usuario abuse del sistema mientras sus solicitudes se distribuyen entre instancias. Redis como store compartido resuelve este problema sin coordinación entre instancias.

## 3. Elección de Tecnología

**Elección: Redis 7 (self-hosted en contenedor Docker)**

| Dimensión | Redis *(Elección)* | Memcached *(Alternativa A)* | Redis Cloud Managed *(Alternativa B)* |
|---|---|---|---|
| **Managed / Self-hosted** | Self-hosted (contenedor Docker en el VPS) | Self-hosted | Managed (Redis Cloud / Upstash) |
| **Complejidad operativa** | Baja: imagen Docker oficial, configuración mínima | Muy baja: más simple que Redis en configuración básica | Muy baja: sin infraestructura propia que operar |
| **Costo a nuestra escala** | $0 adicional (corre en el mismo VPS) | $0 adicional | Desde $0 en plan gratuito de Upstash (10k comandos/día), $10/mes en plan de pago |
| **Feature diferenciador** | Estructuras de datos avanzadas (listas, sets, sorted sets, hashes), pub/sub nativo, persistencia opcional con AOF/RDB, soporte nativo de BullMQ | Solo clave-valor simple, sin persistencia, sin estructuras avanzadas | Serverless con escala automática, latencia global baja con réplicas en múltiples regiones |

Redis es la elección correcta porque el sistema necesita simultáneamente caché de datos, almacenamiento de sesiones, rate limiting distribuido y cola de jobs (BullMQ). Memcached solo cubre la primera necesidad y requeriría operar un segundo sistema para BullMQ. Redis Cloud tendría sentido si el equipo quisiera eliminar la operación de Redis por completo, pero el plan gratuito de Upstash tiene límites muy bajos para producción y el plan de pago añade costo sin beneficio real dado que Redis ya corre en el mismo VPS.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Un único componente cubre cuatro necesidades: caché, sesiones, rate limiting y cola de jobs | Redis es un punto de fallo adicional: si cae, el caché, el rate limiting y BullMQ dejan de funcionar |
| Operaciones atómicas nativas (INCR, DEL, SETNX) garantizan consistencia sin locks de aplicación | La persistencia AOF/RDB añade carga de I/O al disco si se habilita |
| TTL nativo por clave simplifica la gestión de expiración sin lógica adicional | La memoria es limitada: si los datos superan el límite configurado, la política de evicción elimina claves (configurable pero requiere atención) |
| Rendimiento sub-milisegundo para operaciones de lectura y escritura | Sin autenticación por defecto en instancias locales: debe configurarse `requirepass` en producción |

## 5. Integración

En el diagrama de despliegue (Diagrama 4), Redis se ubica en el Data Tier dentro de la VPC privada, accesible por `apps/api` y `apps/worker` pero no expuesto a internet. La separación de responsabilidades dentro de Redis se gestiona con bases de datos lógicas distintas:

```
Redis DB 0 → Caché de slots, perfiles y sesiones  (apps/api)
Redis DB 1 → Colas de BullMQ para jobs            (apps/worker)
Redis DB 2 → Contadores de rate limiting           (Nginx / apps/api)
```

El flujo de invalidación por eventos se integra con el bus de eventos (RabbitMQ) de la siguiente forma:

```
PagoConfirmado (evento) 
    → Handler en scheduling/
    → redis.del(`slots:${doctorId}:${date}`)
    → Próxima consulta de slots va a PostgreSQL y repopula Redis
```

Si Redis no está disponible, el sistema tiene un fallback: `apps/api` detecta el error de conexión y ejecuta la query directamente contra PostgreSQL, degradando el rendimiento pero manteniendo el sistema funcional. Este comportamiento se monitorea en el stack de observabilidad (Grafana) como métrica de `cache_miss_rate`.
