# ADR-001: Modelo de Despliegue — Monolito Modular

| Campo         | Valor                                      |
|---------------|--------------------------------------------|
| Estado        | Accepted                                   |
| Fecha         | 2026-03-31                                 |
| Responsables  | Isabel y Gabriela                          |

---

## Contexto
El sistema de programación de citas médicas debe soportar múltiples roles (paciente, médico, recepcionista, administrador), flujos de pago, notificaciones y generación de historial clínico. El dominio está compuesto por 8 bounded contexts identificados.

El equipo de desarrollo es de tamaño reducido (4–8 personas), el producto se encuentra en etapa de construcción inicial, y las fronteras exactas entre algunos contextos todavía pueden ajustarse conforme el dominio se comprende mejor. El pico de carga más predecible es en horarios de mañana en días laborables, un patrón estable y moderado, muy distinto al tráfico impredecible de plataformas de consumo masivo.

La decisión sobre el modelo de despliegue determina la complejidad operativa, la velocidad de desarrollo y el costo de infraestructura desde el primer día.

---

## Opciones Consideradas

### Opción 1: Monolito Modular

Un único artefacto desplegable, organizado internamente en módulos con límites de dominio explícitos. Los módulos se comunican a través de un bus de eventos interno y barrel exports (`index.ts`), sin importaciones cruzadas directas entre dominios.

| Pros | Contras |
|---|---|
| Un solo pipeline CI/CD, un solo artefacto a operar                                | Escala todo el proceso, no por módulo individual |
| Refactorización de límites de dominio sin costo de contratos de red               | Disciplina modular debe mantenerse manualmente |
| Debugging local sencillo, un proceso, logs centralizados                          | Un bug crítico no aislado puede afectar todo el proceso |
| Transacciones ACID entre módulos cuando se necesiten                              | Migración futura a microservicios requiere esfuerzo de extracción |
| Costo de infraestructura mínimo, con un servidor y una base de datos gestionada   | Despliegue de un cambio pequeño requiere redesplegar todo el artefacto |
| Velocidad de desarrollo alta desde el primer sprint                               |    |

### Opción 2: Microservicios desde el Inicio

Cada bounded context se implementa como un servicio independiente con su propio proceso, base de datos y pipeline de despliegue. Comunicación vía REST/gRPC y un message broker dedicado.

| Pros | Contras |
|---|---|
| Escalabilidad independiente por servicio                                 | Requiere orquestador de contenedores, service mesh, trazabilidad distribuida desde el primer día |
| Aislamiento de fallos real pues un servicio caído no derrumba el sistema | Overhead de coordinación enorme para un equipo de 4–8 personas |
| Equipos autónomos pueden desplegar sin coordinación                      | Las fronteras del dominio aún no están estabilizadas, cambiarlas implica modificar contratos de API entre servicios |
| Tecnología heterogénea por servicio si se necesita                       | Consistencia eventual entre servicios requiere diseño cuidadoso de sagas o eventos compensatorios |
|                                                                          | Costo operativo desproporcionado para la etapa actual del proyecto |

---

## Decisión

Se decide adoptar el Monolito Modular como arquitectura de despliegue inicial.

Las razones son específicas al proyecto:

1. **Tamaño del equipo:** Con 4–8 desarrolladores, dedicar personas a operar Kubernetes, gestionar bases de datos por servicio y mantener contratos de API entre 8 servicios independientes consumiría la mayor parte de la capacidad del equipo en trabajo de infraestructura, no en funcionalidad de negocio.

2. **Dominio en evolución:** Aunque los 8 bounded contexts están identificados, la experiencia indica que en etapas tempranas los límites se ajustan. En un monolito modular ese ajuste cuesta una refactorización interna, en microservicios implica versionamiento de APIs y coordinación de despliegues entre equipos.

3. **Carga esperada:** El pico predecible (mañanas de días laborables) no justifica la complejidad de escalar servicios de forma independiente. Un monolito desplegado con réplicas de lectura y caché maneja decenas de miles de citas diarias sin problema.

4. **Camino de evolución preservado:** La disciplina modular estricta (sin importaciones cruzadas, bus de eventos interno, barrel exports) garantiza que los módulos puedan extraerse a servicios independientes de forma incremental si la escala o el equipo lo justifican en el futuro.

---

## Consecuencias

Habilitado por esta decisión:
- Pipeline CI/CD simple desde el primer sprint.
- Debugging y observabilidad sin necesidad de trazabilidad distribuida.
- Costo de infraestructura inicial reducido.
- Capacidad de refactorizar límites de dominio sin costo de red ni versionamiento de contratos.

Prevenido por esta decisión:
- No es posible escalar el módulo de `scheduling` de forma independiente bajo carga puntual sin escalar todo el proceso. Sería mitigado con réplicas del proceso completo detrás de un balanceador de carga, lo cual es suficiente para el volumen esperado.
- No hay aislamiento de fallos a nivel de proceso entre módulos. Sería mitigado con una disciplina modular estricta, manejo de errores defensivo en cada módulo, y monitoreo de salud del proceso.

Riesgos:
| Riesgo | Mitigación |
|--------|------------|
| Acoplamiento accidental entre módulos si se relaja la disciplina de importaciones | Reglas de linting que rechacen importaciones que violen los límites declarados |
| El monolito crece en tamaño y se vuelve difícil de mantener a largo plazo. | Revisar en cada trimestre si algún módulo justifica extracción basándose en métricas de cambio de código y carga |

---

## ADRs Relacionados

- **ADR-002** — La elección de comunicación síncrona vs. asíncrona dentro del monolito refuerza la separación modular.
- **ADR-003** — La estrategia de base de datos única con esquemas separados es coherente con el monolito modular.
