# ADR-002: Estilo de Comunicación — Síncrono vs. Asíncrono

| Campo        | Valor                                     |
|--------------|-------------------------------------------|
| Estado       | Accepted                                  |
| Fecha        | 2026-03-31                                |
| Responsables | Isabel y Gabriela                         |

---

## Contexto
El sistema tiene flujos que mezclan necesidades distintas de comunicación. Algunos requieren una respuesta inmediata para continuar (el paciente necesita saber si su espacio fue reservado antes de pagar), mientras que otros no necesitan bloquear el flujo principal (enviar un email de confirmación no debe retrasar la respuesta al paciente).

La elección del estilo de comunicación entre módulos determina el acoplamiento temporal, la capacidad de resiliencia ante fallos parciales y la experiencia de usuario percibida. Usar comunicación síncrona para todo crea dependencias en cadena que propagan fallos mientras que usar comunicación asíncrona para todo dificulta el manejo de errores y la consistencia inmediata cuando el usuario espera una respuesta.

El reto es definir criterios claros y aplicarlos de forma consistente en todos los flujos del sistema.

---

## Opciones Consideradas

### Opción 1: Comunicación Síncrona para Todo

Todos los módulos se comunican mediante llamadas directas síncronas. Los flujos de negocio son lineales, es decir, cada paso espera la respuesta del anterior antes de continuar.

| Pros | Contras |
|---|---|
| Modelo mental simple, un flujo lineal fácil de razonar y depurar                | Un módulo lento bloquea toda la cadena |
| Consistencia inmediata en donde el estado es coherente al finalizar la llamada  | Acoplamiento temporal total porque si Notifications falla, la confirmación de cita falla aunque el pago fue exitoso |
| Manejo de errores directo en donde el caller recibe el error y decide           | El módulo de Scheduling necesita conocer la interfaz de Payments, Notifications y Analytics directamente |
| —                                                                               | Imposible reemplazar un módulo sin modificar todos sus callers |

### Opción 2: Comunicación Asíncrona para Todo

Todos los módulos se comunican exclusivamente a través del bus de eventos. Ningún módulo llama directamente a otro. Las respuestas se entregan mediante eventos de respuesta.

| Pros | Contras |
|---|---|
| Desacoplamiento total ya que ningún módulo conoce a sus consumidores  | El patrón request/reply sobre eventos es complejo de implementar correctamente |
| Máxima resiliencia, en donde un módulo caído no bloquea a los demás   | Consistencia eventual en todos los flujos, incluso donde el usuario espera respuesta inmediata |
| Fácil agregar nuevos consumidores sin modificar productores           | Debugging de flujos difícil ya que el estado queda distribuido en eventos |
|                                                                       | El paciente no puede saber en tiempo real si su slot fue reservado exitosamente |

### Opción 3: Criterio Mixto — Síncrono donde se necesita respuesta inmediata, Asíncrono para efectos secundarios

Se define un criterio explícito para elegir el estilo según la naturaleza de la interacción:

- **Síncrono:** Cuando el resultado es necesario para continuar el flujo del usuario (validaciones, reservas, cobros).
- **Asíncrono:** Cuando la acción es un efecto secundario que no bloquea la respuesta principal (notificaciones, actualizaciones de analítica, inicialización de perfiles).

| Pros | Contras |
|---|---|
| Cada módulo usa el estilo que corresponde a su naturaleza               | Requiere criterio claro y disciplina para aplicarlo consistentemente |
| La experiencia del usuario no se degrada por efectos secundarios lentos | Dos patrones distintos en el mismo codebase (mayor curva de aprendizaje inicial) |
| Los efectos secundarios fallidos no abortan flujos críticos             | Los flujos asíncronos requieren manejo de idempotencia y reintentos |
| Notifications puede fallar sin afectar la confirmación de una cita      |  |

---

## Decisión
Se tomó la decisión de adoptar el criterio mixto (Opción 3), con las siguientes reglas aplicadas a los flujos del sistema:

Comunicación síncrona se usa cuando:
- El resultado es requerido para continuar el flujo inmediato del usuario.
- La operación involucra una validación de negocio que puede rechazar la solicitud.
- Ejemplos:
  - `GET /api/scheduling/slots`: Consulta de disponibilidad en tiempo real.
  - `POST /api/scheduling/appointments`: Reserva de espacios (debe confirmar o rechazar inmediatamente).
  - `POST /api/payments/orders/:id/pay`: Pprocesamiento del cobro (el paciente espera confirmación).
  - `POST /api/clinical-records/:id/sign`: Firma del registro (validación de integridad requerida).

Comunicación asíncrona se usa cuando:
- La acción es un efecto secundario que no cambia el resultado para el usuario en ese momento.
- El módulo productor no debe conocer ni depender de los módulos consumidores.
- Ejemplos:
  - `CitaSolicitada`: Dispara creación de orden en Payments (no bloquea la respuesta al paciente).
  - `PagoConfirmado`: Notifica a Scheduling para confirmar la cita y a Notifications para enviar el email.
  - `UsuarioRegistrado`: Inicializa el perfil de paciente y envía email de bienvenida.
  - `CitaCompletada`: Habilita el registro clínico, actualiza métricas de analítica, habilita calificación.

---

## Consecuencias

Habilitado por esta decisión:
- El módulo `notifications/` puede fallar sin afectar la confirmación de citas ni el procesamiento de pagos.
- Agregar un nuevo consumidor de eventos no requiere modificar ningún módulo existente.
- Los flujos de usuario críticos tienen latencia predecible porque no dependen de efectos secundarios.

Prevenido por esta decisión:
- No es posible garantizar consistencia inmediata en los flujos asíncronos. El perfil del paciente puede tardar milisegundos en inicializarse tras el registro.

Riesgos:
| Riesgo | Mitigación |
|--------|------------|
| Un evento asíncrono se procesa dos veces (duplicado por reintento) | Todos los handlers de eventos deben ser idempotentes — verificar si el efecto ya fue aplicado antes de ejecutarlo.|
| Un evento crítico se pierde y la cita nunca se confirma | El bus de eventos en producción usará RabbitMQ con persistencia de mensajes y confirmaciones explícitas.
| El criterio mixto genera confusión en el equipo sobre cuándo usar cada estilo | Las reglas de esta decisión se documentan en el README del repositorio y se revisan en code review.

---

## ADRs Relacionados

- **ADR-001** — El monolito modular es el contexto que hace viable el bus de eventos interno en memoria para desarrollo.
- **ADR-004** — La elección del bus de eventos (RabbitMQ) da soporte a la comunicación asíncrona en producción con las garantías de durabilidad requeridas.
