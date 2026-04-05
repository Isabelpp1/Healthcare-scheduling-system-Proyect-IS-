# Diagrama 3 — Diagramas de Secuencia (Flujos de Datos)

## Sistema de Programación de Citas Médicas

---

## Convenciones

| Notación | Significado |
|---|---|
| `–>>` | Llamada síncrona (HTTP REST o llamada directa entre capas) |
| `-->>` | Respuesta síncrona |
| `-)` | Publicación asíncrona al bus de eventos interno |
| `Note` | Lógica interna relevante al flujo |

---

## Flujo A — Registro y Verificación de Cuenta (Happy Path)

**Escenario:** Un nuevo paciente se registra en la plataforma. El sistema crea su cuenta, envía un correo de verificación y, al confirmar su email, inicializa automáticamente su perfil de paciente.

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Identity as identity/
    participant EventBus as Event Bus
    participant PatProfiles as patient-profiles/
    participant Notifications as notifications/
    participant SendGrid as SendGrid

    Paciente->>API: POST /api/auth/register {email, password, rol: PACIENTE}
    API->>Identity: RegisterUserCommand
    Note over Identity: Valida email único · Hashea contraseña<br/>Crea Usuario (PENDIENTE_VERIFICACIÓN)<br/>Genera token verificación (TTL: 24h)
    Identity-->>API: {userId, message: "Verificación enviada"}
    API-->>Paciente: 201 Created

    Identity-)EventBus: UsuarioRegistrado {userId, email, rol}

    EventBus-)PatProfiles: UsuarioRegistrado
    Note over PatProfiles: Inicializa PerfilPaciente vacío<br/>vinculado al userId

    EventBus-)Notifications: UsuarioRegistrado
    Note over Notifications: Selecciona plantilla BIENVENIDA<br/>Prepara link de verificación
    Notifications->>SendGrid: POST /mail/send {to, subject, html}
    SendGrid-->>Notifications: 202 Accepted
    Notifications-)EventBus: NotificacionEnviada

    Paciente->>API: GET /api/auth/verify-email?token=<token>
    API->>Identity: VerifyEmailCommand {token}
    Note over Identity: Valida token (no expirado, no usado)<br/>Actualiza estadoCuenta → ACTIVO<br/>Marca token como usado
    Identity-->>API: {success: true}
    API-->>Paciente: 200 OK — Cuenta verificada

    Identity-)EventBus: UsuarioVerificado {userId}
```

---

## Flujo A — Fallo: Token de Verificación Expirado

**Escenario:** El paciente intenta verificar su cuenta con un enlace que ya caducó (más de 24 horas).

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Identity as identity/
    participant EventBus as Event Bus
    participant Notifications as notifications/
    participant SendGrid as SendGrid

    Paciente->>API: GET /api/auth/verify-email?token=<token_expirado>
    API->>Identity: VerifyEmailCommand {token}
    Note over Identity: Busca token → encontrado<br/>Verifica expiración → EXPIRADO
    Identity-->>API: Error TOKEN_EXPIRED
    API-->>Paciente: 400 Bad Request {error: "Enlace expirado"}

    Paciente->>API: POST /api/auth/resend-verification {email}
    API->>Identity: ResendVerificationCommand {email}
    Note over Identity: Invalida tokens anteriores<br/>Genera nuevo token (TTL: 24h)
    Identity-->>API: {message: "Nuevo enlace enviado"}
    API-->>Paciente: 200 OK

    Identity-)EventBus: VerificacionReenviada {userId, email}
    EventBus-)Notifications: VerificacionReenviada
    Notifications->>SendGrid: POST /mail/send {nuevo link}
    SendGrid-->>Notifications: 202 Accepted
```

---

## Flujo B — Agendamiento y Pago de Cita (Happy Path)

**Escenario:** Un paciente autenticado selecciona un médico, elige un slot disponible, solicita la cita y realiza el pago. La cita solo se confirma si el pago es exitoso (patrón Partnership entre Scheduling y Payments).

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Scheduling as scheduling/
    participant EventBus as Event Bus
    participant Payments as payments/
    participant Stripe as Stripe API

    Paciente->>API: GET /api/scheduling/slots?doctorId=X&date=2025-08-15
    API->>Scheduling: GetAvailableSlotsQuery {doctorId, date}
    Note over Scheduling: Calcula slots libres cruzando<br/>disponibilidad base, bloqueos y citas existentes
    Scheduling-->>API: [{hora:"09:00", disponible:true}, ...]
    API-->>Paciente: 200 OK — Lista de slots

    Paciente->>API: POST /api/scheduling/appointments {doctorId, slotId, tipo:PRESENCIAL, motivo}
    API->>Scheduling: RequestAppointmentCommand
    Note over Scheduling: Valida slot disponible<br/>Crea Cita (SOLICITADA)<br/>Reserva slot (bloqueo temporal 10 min)
    Scheduling-->>API: {appointmentId, monto:500.00, moneda:"GTQ"}
    API-->>Paciente: 202 Accepted — Pendiente de pago

    Scheduling-)EventBus: CitaSolicitada {appointmentId, pacienteId, medicoId, monto}
    EventBus-)Payments: CitaSolicitada
    Note over Payments: Crea Orden {monto:500.00, estado:PENDIENTE}

    Paciente->>API: POST /api/payments/orders/:orderId/pay {paymentMethodToken}
    API->>Payments: ProcessPaymentCommand {orderId, token}
    Payments->>Stripe: POST /charges {amount:500, currency:"GTQ", token}
    Stripe-->>Payments: {chargeId, status:"succeeded"}
    Note over Payments: Crea Transacción (COBRO)<br/>Actualiza Orden → COMPLETADA
    Payments-->>API: {status:"success", transactionId}
    API-->>Paciente: 200 OK — Pago exitoso

    Payments-)EventBus: PagoConfirmado {appointmentId, orderId, monto}
    EventBus-)Scheduling: PagoConfirmado
    Note over Scheduling: Actualiza Cita → CONFIRMADA<br/>Convierte reserva temporal en definitiva
```

---

## Flujo B — Fallo: Pago Rechazado por Stripe

**Escenario:** La tarjeta del paciente es rechazada. El sistema revierte la reserva del slot y libera la disponibilidad del médico.

```mermaid
sequenceDiagram
    participant API as API Gateway
    participant Payments as payments/
    participant Stripe as Stripe API
    participant EventBus as Event Bus
    participant Scheduling as scheduling/
    participant Notifications as notifications/

    Note over Payments,Stripe: Cita en estado SOLICITADA · Slot reservado temporalmente

    API->>Payments: ProcessPaymentCommand {orderId, token}
    Payments->>Stripe: POST /charges {amount:500, currency:"GTQ", token}
    Stripe-->>Payments: {status:"failed", code:"card_declined"}
    Note over Payments: Crea Transacción (COBRO, FALLIDA)<br/>Actualiza Orden → FALLIDA
    Payments-->>API: 402 Payment Required {error:"Tarjeta rechazada", retriable:true}

    Payments-)EventBus: PagoFallido {appointmentId, orderId, motivo:"card_declined"}

    EventBus-)Scheduling: PagoFallido
    Note over Scheduling: Actualiza Cita → CANCELADA<br/>Libera slot · Disponibilidad restaurada

    EventBus-)Notifications: PagoFallido
    Note over Notifications: Alerta al paciente sobre fallo<br/>Incluye enlace para reintentar
```

---

## Flujo C — Cancelación con Reembolso (Happy Path)

**Escenario:** Un paciente cancela una cita confirmada con más de 24 horas de anticipación. El sistema cancela la cita, libera el slot y procesa el reembolso completo vía Stripe.

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Scheduling as scheduling/
    participant EventBus as Event Bus
    participant Payments as payments/
    participant Stripe as Stripe API
    participant Notifications as notifications/

    Paciente->>API: DELETE /api/scheduling/appointments/:appointmentId {motivo}
    API->>Scheduling: CancelAppointmentCommand {appointmentId, motivo}
    Note over Scheduling: Valida cita CONFIRMADA<br/>Política: >24h anticipación → reembolso 100%<br/>Actualiza Cita → CANCELADA · Libera slot
    Scheduling-->>API: {status:"cancelled"}
    API-->>Paciente: 200 OK — Cita cancelada

    Scheduling-)EventBus: CitaCancelada {appointmentId, horasAnticipacion:36}

    EventBus-)Payments: CitaCancelada
    Note over Payments: Política >24h → reembolso 100%<br/>Monto a devolver: 500.00 GTQ
    Payments->>Stripe: POST /refunds {chargeId, amount:500}
    Stripe-->>Payments: {refundId, status:"succeeded"}
    Note over Payments: Crea Transacción (REEMBOLSO)<br/>Actualiza Orden → REEMBOLSADA
    Payments-)EventBus: ReembolsoCompletado {appointmentId, monto:500.00}

    EventBus-)Notifications: CitaCancelada
    Note over Notifications: Notifica cancelación al paciente y médico

    EventBus-)Notifications: ReembolsoCompletado
    Note over Notifications: Envía comprobante de reembolso<br/>Plazo estimado: 5-10 días hábiles
```

---

## Flujo C — Fallo: Reembolso Fallido en Stripe

**Escenario:** Stripe rechaza el reembolso (ej. cargo ya reembolsado previamente). El sistema registra el fallo y escala a revisión manual.

```mermaid
sequenceDiagram
    participant Payments as payments/
    participant Stripe as Stripe API
    participant EventBus as Event Bus
    participant Notifications as notifications/

    Payments->>Stripe: POST /refunds {chargeId, amount:500}
    Stripe-->>Payments: {status:"failed", code:"charge_already_refunded"}
    Note over Payments: Registra fallo en Transacción<br/>Marca Orden para revisión manual<br/>Alerta al equipo de operaciones

    Payments-)EventBus: ReembolsoFallido {appointmentId, motivo:"charge_already_refunded"}

    EventBus-)Notifications: ReembolsoFallido
    Note over Notifications: Notifica al paciente que el reembolso<br/>requiere revisión manual<br/>Incluye número de caso para seguimiento
```

---

## Resumen de Flujos Cubiertos

| Flujo | Happy Path | Fallo / Compensación | Comunicación |
|---|---|---|---|
| A — Registro y verificación | Registro + verificación email | Token expirado + reenvío | Síncrona + Asíncrona |
| B — Agendamiento y pago | Solicitud + pago + confirmación | Pago rechazado + reversión de slot | Síncrona + Asíncrona |
| C — Cancelación con reembolso |  Cancelación + reembolso Stripe | Reembolso fallido + escalación manual | Asíncrona dominante |
