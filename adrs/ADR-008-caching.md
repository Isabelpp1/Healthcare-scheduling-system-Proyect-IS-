# ADR-008: Estrategia de Caché — Redis con Invalidación por Eventos

| Campo | Valor |
|---|---|
| Estado | Accepted |
| Fecha | 2026-04-05 |
| Responsables | Isabel y Gabriela  |

---

## Contexto

El sistema tiene patrones de acceso a datos muy dispares entre sus módulos. El endpoint de consulta de slots disponibles (`GET /api/scheduling/slots`) es el más crítico en términos de frecuencia: un paciente puede consultarlo múltiples veces antes de decidir un horario, y durante los picos de mañana muchos pacientes consultan los slots del mismo médico simultáneamente. Sin caché, cada consulta recalcula la disponibilidad cruzando tres tablas (disponibilidad base, bloqueos y citas existentes), generando carga innecesaria en la base de datos.

Por otro lado, los datos del sistema tienen distintos niveles de volatilidad: los slots disponibles cambian con cada cita confirmada, pero los perfiles de médicos y especialidades cambian muy raramente. Una estrategia de caché única para todos los datos sería ineficiente.

El sistema también usa Redis como cola de jobs para el Worker (BullMQ), por lo que ya existe una instancia de Redis en la infraestructura. Usar el mismo componente para caché evita añadir complejidad operativa innecesaria.

---

## Opciones Consideradas

### Opción 1: Redis con Invalidación por Eventos

Redis como caché en memoria con TTLs distintos por tipo de dato. La invalidación se dispara cuando se publican eventos de dominio relevantes (por ejemplo, al confirmarse un pago se invalida el caché de slots del médico correspondiente).

| Pros | Contras |
|---|---|
| Latencia de lectura sub-milisegundo para datos cacheados | Requiere lógica de invalidación por evento en cada módulo relevante |
| TTLs por tipo de dato permiten políticas diferenciadas | Un fallo de Redis en producción degradaría el rendimiento (no el funcionamiento) del sistema |
| La instancia ya existe en la infraestructura (usada por BullMQ) | La invalidación incorrecta puede servir datos obsoletos temporalmente |
| Invalidación por eventos mantiene consistencia cuando hay cambios | — |
| Soporte nativo en Node.js con librería `ioredis` | — |

### Opción 2: Memcached

Caché en memoria distribuida con API más simple que Redis. Solo soporta estructuras clave-valor simples.

| Pros | Contras |
|---|---|
| Más simple operativamente que Redis en configuración básica | No soporta estructuras de datos avanzadas (listas, sets, hashes) |
| Rendimiento ligeramente superior en operaciones de lectura pura | Sin persistencia: todos los datos se pierden al reiniciar |
| — | No tiene soporte de pub/sub ni streams, por lo que no puede reemplazar a Redis para BullMQ |
| — | Requeriría operar dos sistemas de caché (Redis para BullMQ + Memcached para caché), añadiendo complejidad |

### Opción 3: Caché HTTP en API Gateway (Nginx)

Configurar Nginx para cachear las respuestas HTTP de los endpoints de consulta más frecuentes directamente en el gateway, sin Redis.

| Pros | Contras |
|---|---|
| Sin dependencia de componentes adicionales para caché | Solo funciona para respuestas GET idénticas (mismos query params) |
| Transparente para el código de la aplicación | No distingue entre datos que cambiaron y datos que no: invalida por TTL, no por evento |
| Reduce carga en el monolito, no solo en la base de datos | Un slot recién ocupado puede servirse como disponible hasta que expira el TTL del gateway |
| — | Sin visibilidad de hit/miss ratio desde la aplicación |

---

## Decisión

Se adopta **Redis con invalidación por eventos** como estrategia de caché, usando la misma instancia que BullMQ.

Razones específicas al proyecto:

1. **Slots disponibles son el dato más consultado y más volátil:** Pueden cambiar en segundos (cuando otro paciente confirma una cita). Un caché HTTP con TTL fijo serviría slots ocupados como disponibles durante la ventana del TTL, causando conflictos de concurrencia. La invalidación por evento (`PagoConfirmado` → invalida caché del médico) garantiza que el caché solo se usa mientras los datos son válidos.

2. **Reutilización de infraestructura existente:** Redis ya está en la arquitectura para BullMQ. Añadir caché en la misma instancia es un cambio de configuración, no un nuevo componente. Memcached requeriría operar dos sistemas en memoria distintos sin beneficio real.

3. **TTLs diferenciados por tipo de dato:** No todos los datos necesitan la misma política. Los perfiles de médicos y el catálogo de especialidades pueden tener TTL de horas; los slots disponibles, TTL de minutos como máximo antes de la invalidación por evento.

---

## Política de Caché por Tipo de Dato

| Dato cacheado | Clave Redis | TTL | Evento que invalida |
|---|---|---|---|
| Slots disponibles por médico y fecha | `slots:{doctorId}:{date}` | 5 min | `PagoConfirmado`, `CitaCancelada`, `BloqueoAgendaCreado` |
| Perfil público de médico | `doctor:profile:{doctorId}` | 2 horas | `PerfilMédicoActualizado`, `MédicoVerificado` |
| Catálogo de especialidades | `specialties:all` | 24 horas | `EspecialidadCreada` (admin) |
| Sesión de usuario activa | `session:{userId}` | 1 hora (igual que JWT) | `SesiónCerrada`, `CuentaSuspendida` |

**Datos que nunca se cachean:**
- Registros clínicos y prescripciones (sensibilidad y frecuencia de cambio impredecible)
- Estado de órdenes de pago (consistencia crítica requerida)
- Datos personales del paciente

---

## Consecuencias

Habilitado por esta decisión:
- El endpoint de slots disponibles responde desde Redis en < 5 ms en lugar de ejecutar una query compleja contra PostgreSQL en cada solicitud.
- Los picos de carga en horarios de mañana se absorben sin escalar la base de datos.
- La invalidación por eventos garantiza que los datos obsoletos no se sirven más allá del tiempo estrictamente necesario.

Prevenido por esta decisión:
- No es posible cachear datos con consistencia fuerte sin invalidación por evento. Si un evento de invalidación se pierde (fallo del bus), el caché puede servir datos obsoletos hasta que expire el TTL de respaldo.

Riesgos:

| Riesgo | Mitigación |
|---|---|
| Redis falla y el sistema no puede leer slots desde caché | El sistema tiene un fallback: si Redis no responde, la query va directamente a PostgreSQL. El rendimiento se degrada pero el sistema funciona. |
| Dos instancias del monolito invalidan el mismo caché simultáneamente (race condition) | Las operaciones de invalidación usan `DEL` atómico de Redis. La siguiente lectura repopula el caché con datos frescos de la DB. |
| El caché crece sin límite si las claves no expiran correctamente | Redis configurado con política de evicción `allkeys-lru` y límite de memoria de 256 MB para la capa de caché. |

---

## ADRs Relacionados

- **ADR-001** — El monolito modular centraliza la lógica de invalidación de caché; no hay coordinación de caché distribuida entre servicios.
- **ADR-002** — Los eventos asíncronos (`PagoConfirmado`, `CitaCancelada`) son el mecanismo de invalidación de caché, conectando la estrategia de comunicación con la estrategia de caché.
- **ADR-003** — Redis complementa a PostgreSQL: las lecturas frecuentes de baja latencia van a Redis; las escrituras y lecturas críticas van siempre a PostgreSQL.
- **ADR-006** — La misma instancia de Redis que sirve de caché aloja las colas de BullMQ para el Worker. La separación se gestiona con bases de datos Redis distintas (DB 0 para caché, DB 1 para BullMQ).
