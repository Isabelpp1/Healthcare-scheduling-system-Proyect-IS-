# Componente: Base de Datos Principal (PostgreSQL)

## 1. Responsabilidades

- **Persistencia transaccional del core domain:** Almacena todas las entidades del sistema bajo esquemas PostgreSQL separados por bounded context (`scheduling`, `payments`, `identity`, `doctor_profiles`, `patient_profiles`, `notifications`, `clinical_records`, `analytics`), garantizando que los datos de cada contexto solo sean accesibles a través de su propio repositorio.
- **Transacciones ACID para el flujo de pago:** El patrón Partnership entre `scheduling/` y `payments/` requiere que la creación de una orden y la actualización del estado de una cita puedan participar en la misma transacción de base de datos cuando se necesite atomicidad en casos de compensación.
- **Réplica de lectura para analítica:** El Primary gestiona todas las escrituras y lecturas críticas. Una réplica de solo lectura recibe el tráfico de consultas pesadas de Analytics y del Worker, protegiéndose de que reportes de larga ejecución impacten la latencia del API.
- **Migraciones independientes por módulo:** Cada módulo mantiene su propia carpeta de migraciones (`modules/scheduling/infrastructure/persistence/migrations/`) que solo afecta tablas dentro de su esquema, permitiendo que los equipos migren de forma independiente sin coordinar.
- **Integridad referencial dentro de cada esquema:** Las claves foráneas se definen dentro del esquema de cada bounded context, no entre esquemas. La consistencia entre contextos se garantiza mediante eventos de dominio, no por FK cruzadas.

## 2. Por Qué Este Proyecto lo Necesita

Los datos de un sistema de salud son de alta criticidad: un registro clínico perdido, una cita doble-reservada o una transacción de pago inconsistente pueden tener consecuencias legales y de seguridad del paciente. Estas garantías requieren una base de datos relacional con soporte ACID completo, no una solución eventual-consistent.

La decisión de usar un único PostgreSQL con esquemas separados está directamente vinculada al patrón Partnership entre Scheduling y Payments: cuando un pago falla y la cita debe revertirse, la operación de compensación es más segura y simple dentro de una transacción ACID que coordinando dos bases de datos distintas con sagas distribuidas.

## 3. Elección de Tecnología

**Elección: PostgreSQL 15**

| Dimensión | PostgreSQL gestionado *(Elección)* | MySQL / MariaDB *(Alternativa)* | MongoDB *(Alternativa)* |
|---|---|---|---|
| **Managed / Self-hosted** | Managed | Managed o Self-hosted | Managed o Self-hosted |
| **Complejidad operativa** | Muy baja: backups, parches y failover automáticos | Baja | Baja (Atlas) o Media (self-hosted) |
| **Costo a nuestra escala** | ~$15–25/mes en Railway; ~$25/mes RDS t3.micro | Similar a PostgreSQL | Atlas M0 gratuito con límites; M2 ~$9/mes |
| **Feature diferenciador** | ACID completo, esquemas nativos para aislamiento de contextos, JSONB para datos semiestructurados, extensiones maduras | ACID completo, ampliamente conocido, ligeramente más rápido en workloads simples | Documentos flexibles, escalado horizontal nativo, sin esquema rígido |

PostgreSQL es la elección porque la naturaleza relacional del dominio (citas vinculadas a médicos, pacientes, pagos y registros clínicos) encaja perfectamente con el modelo relacional. Los esquemas PostgreSQL son el mecanismo nativo de aislamiento que la arquitectura modular requiere. MongoDB, aunque tiene soporte ACID multi-documento desde la versión 4, añade complejidad en las relaciones entre entidades sin aportar beneficios para este dominio predominantemente estructurado.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Esquemas PostgreSQL como mecanismo nativo de aislamiento de bounded contexts | Escalado vertical principalmente |
| Transacciones ACID para compensaciones en el flujo de pago sin sagas distribuidas | Las migraciones mal planificadas pueden bloquear tablas durante el alter |
| JSONB para campos semiestructurados en Clinical Records  | Un único Primary es punto de fallo para escrituras |
| Instancia gestionada: backups diarios automáticos, failover, parches de seguridad sin intervención | Costo mensual fijo aunque el volumen sea bajo en etapas iniciales |
| `pg_trgm` para búsqueda de texto en nombres de médicos y especialidades sin Search Engine adicional |  |

## 5. Integración

PostgreSQL se ubica en el Data Tier de la VPC privada. Las conexiones son:

```
apps/api    -> PostgreSQL Primary :5432  (lectura + escritura de todos los módulos)
apps/worker -> PostgreSQL Replica :5432  (solo lectura para reportes y recordatorios)
```

La réplica se configura con streaming replication nativa de PostgreSQL (flecha punteada en el Diagrama 4). El pool de conexiones en `apps/api` usa PgBouncer en modo transaction pooling para evitar agotar las conexiones máximas del Primary bajo carga concurrente. La conexión con Redis es complementaria: los slots de disponibilidad se cachean en Redis y se invalidan cuando los eventos del bus de eventos actualizan el estado de las citas en PostgreSQL.
