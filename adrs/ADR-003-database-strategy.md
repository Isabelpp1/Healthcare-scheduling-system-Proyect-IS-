# ADR-003: Estrategia de Base de Datos — Base de Datos Única con Esquemas Separados por Contexto

| Campo        | Valor                                     |
|--------------|-------------------------------------------|
| Estado       | Accepted                                  |
| Fecha        | 2026-03-31                                |
| Responsables | Isabel y Gabriela                         |

---

## Contexto
El sistema tiene 8 bounded contexts, cada uno con entidades propias y distintos patrones de acceso a datos. Algunos contextos tienen necesidades de consistencia fuertes como Scheduling y Payments, los cuales deben coordinarse para confirmar o revertir una cita según el resultado del pago. Otros son puramente de lectura, como Analytics o tienen datos de naturaleza semiestructurada como Clinical Records con documentos adjuntos.

La estrategia de base de datos determina el nivel de aislamiento entre contextos, las garantías de consistencia disponibles, el costo operativo y la complejidad del esquema. En el contexto de un monolito modular con un equipo pequeño, la elección también debe ser operativamente sostenible.

---

## Opciones Consideradas

### Opción 1: Base de Datos Única, Esquema Único Compartido

Una sola base de datos PostgreSQL con todas las tablas en un único esquema público. Todos los módulos acceden libremente a cualquier tabla.

| Pros | Contras |
|---|---|
| Configuración mínima, sin overhead de gestión                         | Cualquier módulo puede acceder a los datos de otro directamente, rompiendo los límites de dominio |
| Joins entre tablas de distintos contextos sin restricciones           | Sin aislamiento, una migración en una tabla puede romper otro módulo |
| Transacciones ACID disponibles para cualquier combinación de tablas   | El esquema crece sin orden, acoplamiento estructural difícil de revertir |

### Opción 2: Base de Datos por Bounded Context

Cada bounded context tiene su propia base de datos.

| Pros | Contras |
|---|---|
| Aislamiento total, en donde ningún módulo puede acceder a los datos de otro   | Gestionar 8 bases de datos distintas es inviable para un equipo de 4–8 personas |
| Cada contexto puede usar la tecnología más adecuada a su naturaleza           | Sin transacciones ACID entre contextos, el patrón Partnership entre Scheduling y Payments requiere sagas distribuidas |
| Escala de almacenamiento independiente por contexto                           | Backup, monitoreo, migraciones y acceso de emergencia multiplicados por 8 |
|                                                                               | Costo desproporcionado para la etapa actual del proyecto |

### Opción 3: Base de Datos Única con Esquemas PostgreSQL Separados por Contexto

Una sola instancia de PostgreSQL con un esquema de base de datos por bounded context: `identity`, `doctor_profiles`, `patient_profiles`, `scheduling`, `payments`, `notifications`, `clinical_records`, `analytics`. Cada módulo solo accede a su propio esquema mediante su repositorio.

| Pros | Contras |
|---|---|
| Aislamiento lógico porque las tablas de un contexto no son visibles por convención desde otro módulo  | El aislamiento es por convención de código, no por separación de procesos |
| Una sola instancia a operar, respaldar y monitorear                                                   | Joins físicos entre esquemas son técnicamente posibles |
| Transacciones ACID disponibles para el patrón Partnership                                             | Una instancia grande puede ser un punto único de fallo sin alta disponibilidad |
| Migraciones por esquema en donde cada módulo gestiona sus propias migraciones de forma independiente  |  |
| Fácil evolución porque si un contexto requiere base de datos propia, el esquema ya está delimitado    |  |

---

## Decisión
Se optó por adoptar la Opción 3, una base de datos PostgreSQL única con esquemas separados por bounded context.

Razones específicas al proyecto:

1. **Consistencia en el flujo de pago:** El patrón Partnership entre `scheduling` y `payments` requiere que una cita y su orden de pago puedan participar en la misma transacción ACID en casos de compensación. 

2. **Viabilidad operativa:** Un equipo de 4–8 personas no puede operar 8 bases de datos distintas con sus respectivos backups, monitoreos, planes de recuperación y conexiones de pool. Una instancia gestionada es suficiente y económica.

3. **Aislamiento preservado por disciplina modular:** Cada módulo accede exclusivamente a su propio esquema a través de su repositorio. Ningún repositorio importa tablas de otro esquema. 

4. **Camino de evolución:** Si en el futuro `clinical_records` necesita una base de datos documental o `analytics` requiere un data warehouse separado, el esquema ya está delimitado y la extracción es directa.

---

## Consecuencias

Habilitado por esta decisión:
- Transacciones ACID en el flujo crítico de pago sin complejidad de sagas distribuidas.
- Migraciones independientes por módulo: el equipo de scheduling puede modificar su esquema sin coordinar con otros módulos.
- Una sola conexión de monitoreo, backup y restauración de emergencia.
- Las métricas de analítica se sirven desde vistas materializadas sin impactar el rendimiento de las tablas transaccionales.

Prevenido por esta decisión:
- No es posible escalar el almacenamiento de `scheduling` de forma independiente sin escalar toda la instancia.
- No es posible usar tecnologías de almacenamiento especializadas por contexto.

Riesgos:
| Riesgo | Mitigación |
|--------|------------|
| Un desarrollador escribe un query que cruza esquemas directamente | Los repositorios son la única capa que toca la base de datos, se prohíbe SQL crudo en la capa de aplicación, el code review aquí se vuelve indispensable |
| La instancia única es un punto de fallo | PostgreSQL gestionado con réplica de lectura y failover automático |

---

## ADRs Relacionados

- **ADR-001** — El monolito modular hace viable y suficiente la base de datos única.
- **ADR-002** — La comunicación asíncrona entre módulos evita que los módulos necesiten leer tablas ajenas directamente, los datos necesarios viajan en los eventos.