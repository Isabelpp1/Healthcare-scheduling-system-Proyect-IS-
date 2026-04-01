# 04 - Flujos de Datos e Interacciones
Healthcare scheduling system

## Introducción

Este documento describe los flujos end-to-end más representativos del sistema. Cada flujo incluye el escenario de negocio, un diagrama de secuencia con los módulos involucrados, el flujo ideal (*happy path*) y al menos un camino de fallo o compensación.

Los flujos se trazan a nivel de módulos internos del monolito modular, reflejando la comunicación síncrona (llamadas directas vía HTTP o Application Service) y la comunicación asíncrona (eventos publicados al bus de eventos interno).

### Convenciones del Diagrama

| Notación | Significado |
|---|---|
| `-->>`  | Llamada síncrona (HTTP / llamada directa entre capas) |
| `-->>`  | Respuesta síncrona |
| `--)` | Publicación asíncrona al bus de eventos |
| `Note` | Acción interna relevante al flujo |

---

## Flujo 1: Registro y Verificación de Cuenta

### Escenario de Negocio

Un nuevo usuario (paciente) se registra en la plataforma. El sistema crea su cuenta, envía un correo de verificación y, al confirmar su email, inicializa automáticamente su perfil de paciente listo para agendar citas.

---

### Happy Path

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Identity as identity/
    participant EventBus as Event Bus
    participant PatientProfiles as patient-profiles/
    participant Notifications as notifications/
    participant Email as Proveedor Email<br/>(SendGrid)

    Paciente->>API: POST /api/auth/register<br/>{ email, password, rol: PACIENTE }
    API->>Identity: RegisterUserCommand
    Note over Identity: Valida email único<br/>Hashea contraseña<br/>Crea Usuario (estado: PENDIENTE_VERIFICACIÓN)<br/>Genera token de verificación (expira 24h)
    Identity-->>API: { userId, message: "Verificación enviada" }
    API-->>Paciente: 201 Created

    Identity-)EventBus: UsuarioRegistrado { userId, email, rol }

    EventBus-)PatientProfiles: UsuarioRegistrado
    Note over PatientProfiles: Inicializa PerfilPaciente vacío<br/>vinculado al userId

    EventBus-)Notifications: UsuarioRegistrado
    Note over Notifications: Selecciona plantilla BIENVENIDA<br/>Prepara email con link de verificación
    Notifications->>Email: Enviar email de verificación
    Email-->>Notifications: 200 OK
    Notifications-)EventBus: NotificaciónEnviada

    Paciente->>API: GET /api/auth/verify-email?token=<token>
    API->>Identity: VerifyEmailCommand { token }
    Note over Identity: Valida token (no expirado, no usado)<br/>Actualiza estadoCuenta → ACTIVO<br/>Marca token como usado
    Identity-->>API: { success: true }
    API-->>Paciente: 200 OK — Cuenta verificada

    Identity-)EventBus: UsuarioVerificado { userId }
```

---

### Camino de Fallo: Token de Verificación Expirado

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Identity as identity/
    participant EventBus as Event Bus
    participant Notifications as notifications/
    participant Email as Proveedor Email<br/>(SendGrid)

    Note over Paciente,Identity: El paciente intenta verificar con un token expirado (>24h)

    Paciente->>API: GET /api/auth/verify-email?token=<token_expirado>
    API->>Identity: VerifyEmailCommand { token }
    Note over Identity: Busca token → encontrado<br/>Verifica expiración → EXPIRADO
    Identity-->>API: Error: TOKEN_EXPIRED
    API-->>Paciente: 400 Bad Request<br/>{ error: "El enlace de verificación ha expirado" }

    Paciente->>API: POST /api/auth/resend-verification<br/>{ email }
    API->>Identity: ResendVerificationCommand { email }
    Note over Identity: Invalida tokens anteriores<br/>Genera nuevo token (expira 24h)
    Identity-->>API: { message: "Nuevo enlace enviado" }
    API-->>Paciente: 200 OK

    Identity-)EventBus: VerificaciónReenviada { userId, email }
    EventBus-)Notifications: VerificaciónReenviada
    Notifications->>Email: Enviar nuevo email de verificación
    Email-->>Notifications: 200 OK
```

---

## Flujo 2: Agendamiento y Pago de Cita

### Escenario de Negocio

Un paciente autenticado selecciona un médico, elige un espacio disponible, confirma la cita y realiza el pago. Este es el flujo core del sistema: involucra coordinación entre Scheduling y Payments bajo el patrón Partnership, donde la cita solo se confirma si el pago es exitoso.

---

### Happy Path

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Scheduling as scheduling/
    participant DoctorProfiles as doctor-profiles/
    participant Payments as payments/
    participant Stripe as Stripe API
    participant EventBus as Event Bus
    participant Notifications as notifications/

    Paciente->>API: GET /api/scheduling/slots?doctorId=X&date=2025-08-15
    API->>Scheduling: GetAvailableSlotsQuery { doctorId, date }
    Note over Scheduling: Calcula espacios libres<br/>cruza disponibilidad base<br/>con bloqueos y citas existentes
    Scheduling-->>API: [ { hora: "09:00", disponible: true }, ... ]
    API-->>Paciente: 200 OK — Lista de slots

    Paciente->>API: POST /api/scheduling/appointments<br/>{ doctorId, slotId, tipo: PRESENCIAL, motivo }
    API->>Scheduling: RequestAppointmentCommand
    Note over Scheduling: Valida el espacio aún disponible<br/>Crea Cita (estado: SOLICITADA)<br/>Reserva el espacio (bloqueo temporal 10 min)
    Scheduling-->>API: { appointmentId, monto: 500.00, moneda: "GTQ" }
    API-->>Paciente: 202 Accepted — Cita solicitada, pendiente de pago

    Scheduling-)EventBus: CitaSolicitada { appointmentId, pacienteId, médicoId, monto }

    EventBus-)Payments: CitaSolicitada
    Note over Payments: Crea Orden { monto: 500.00, estado: PENDIENTE }<br/>vinculada al appointmentId

    Paciente->>API: POST /api/payments/orders/:orderId/pay<br/>{ paymentMethodToken }
    API->>Payments: ProcessPaymentCommand { orderId, token }
    Payments->>Stripe: Cargo { amount: 500, currency: "GTQ", token }
    Stripe-->>Payments: { chargeId, status: "succeeded" }
    Note over Payments: Crea Transacción (tipo: COBRO)<br/>Actualiza Orden → COMPLETADA
    Payments-->>API: { status: "success", transactionId }
    API-->>Paciente: 200 OK — Pago exitoso

    Payments-)EventBus: PagoConfirmado { appointmentId, orderId, monto }

    EventBus-)Scheduling: PagoConfirmado
    Note over Scheduling: Actualiza Cita → CONFIRMADA<br/>Convierte reserva temporal en definitiva

    EventBus-)Notifications: PagoConfirmado
    Note over Notifications: Envía confirmación de cita<br/>+ comprobante de pago al paciente<br/>Envía aviso al médico
```

---

### Camino de Fallo: Pago Rechazado por la Pasarela

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Payments as payments/
    participant Stripe as Stripe API
    participant EventBus as Event Bus
    participant Scheduling as scheduling/
    participant Notifications as notifications/

    Note over Paciente,Stripe: La cita está en estado SOLICITADA<br/>El espacio está reservado temporalmente

    Paciente->>API: POST /api/payments/orders/:orderId/pay<br/>{ paymentMethodToken }
    API->>Payments: ProcessPaymentCommand { orderId, token }
    Payments->>Stripe: Cargo { amount: 500, currency: "GTQ", token }
    Stripe-->>Payments: { status: "failed", code: "card_declined" }
    Note over Payments: Crea Transacción (tipo: COBRO, estado: FALLIDA)<br/>Actualiza Orden → FALLIDA
    Payments-->>API: 402 Payment Required<br/>{ error: "Tarjeta rechazada", retriable: true }
    API-->>Paciente: 402 — Pago rechazado

    Payments-)EventBus: PagoFallido { appointmentId, orderId, motivo: "card_declined" }

    EventBus-)Scheduling: PagoFallido
    Note over Scheduling: Actualiza Cita → CANCELADA<br/>Libera el slot reservado<br/>El espacio vuelve a estar disponible

    EventBus-)Notifications: PagoFallido
    Note over Notifications: Notifica al paciente sobre el fallo<br/>Incluye enlace para reintentar pago

    Note over Paciente: El paciente puede reintentar<br/>con otro método de pago<br/>creando una nueva orden
```

---

### Camino de Fallo: Espacio Ocupado en el Momento de Confirmar

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Scheduling as scheduling/

    Note over Paciente,Scheduling: Dos pacientes seleccionan el mismo espacio<br/>casi simultáneamente

    Paciente->>API: POST /api/scheduling/appointments<br/>{ doctorId, slotId: "09:00", tipo: PRESENCIAL }
    API->>Scheduling: RequestAppointmentCommand
    Note over Scheduling: Intenta reservar slot 09:00<br/>→ El espacio ya fue tomado por otro paciente<br/>(lock de base de datos / check de concurrencia)
    Scheduling-->>API: 409 Conflict
    API-->>Paciente: 409 — Slot no disponible<br/>{ error: "El horario seleccionado ya no está disponible",<br/>  suggestion: "Consulte los espacips actualizados" }

    Paciente->>API: GET /api/scheduling/slots?doctorId=X&date=2025-08-15
    Note over Paciente: El paciente consulta los espacios<br/>actualizados y elige otro horario
```

---

## Flujo 3: Cancelación de Cita con Reembolso

### Escenario de Negocio

Un paciente cancela una cita previamente confirmada con más de 24 horas de anticipación. El sistema cancela la cita, libera el espacio del médico y procesa el reembolso completo al método de pago original del paciente.

---

### Happy Path

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Scheduling as scheduling/
    participant EventBus as Event Bus
    participant Payments as payments/
    participant Stripe as Stripe API
    participant Notifications as notifications/

    Paciente->>API: DELETE /api/scheduling/appointments/:appointmentId<br/>{ motivo: "Conflicto de horario" }
    API->>Scheduling: CancelAppointmentCommand { appointmentId, motivo }
    Note over Scheduling: Valida que cita existe y está CONFIRMADA<br/>Valida política: cancelación >24h antes → reembolso total<br/>Actualiza Cita → CANCELADA<br/>Libera el espacio del médico
    Scheduling-->>API: { status: "cancelled" }
    API-->>Paciente: 200 OK — Cita cancelada

    Scheduling-)EventBus: CitaCancelada { appointmentId, pacienteId, médicoId, horasAnticipación: 36 }

    EventBus-)Payments: CitaCancelada
    Note over Payments: Consulta orden asociada al appointmentId<br/>Aplica política: >24h → reembolso 100%<br/>Calcula monto a reembolsar: 500.00 GTQ<br/>Inicia proceso de reembolso

    Payments->>Stripe: Reembolso { chargeId, amount: 500 }
    Stripe-->>Payments: { refundId, status: "succeeded" }
    Note over Payments: Crea Transacción (tipo: REEMBOLSO)<br/>Actualiza Orden → REEMBOLSADA
    Payments-)EventBus: ReembolsoCompletado { appointmentId, monto: 500.00 }

    EventBus-)Notifications: CitaCancelada
    Note over Notifications: Notifica cancelación al paciente y al médico

    EventBus-)Notifications: ReembolsoCompletado
    Note over Notifications: Envía comprobante de reembolso al paciente<br/>con plazo estimado de acreditación (5-10 días hábiles)

    EventBus-)Analytics: CitaCancelada
    Note over Analytics: Incrementa contador de cancelaciones<br/>en métricas del médico para el período
```

---

### Camino de Fallo: Cancelación Tardía (Sin Derecho a Reembolso)

```mermaid
sequenceDiagram
    actor Paciente
    participant API as API Gateway
    participant Scheduling as scheduling/
    participant EventBus as Event Bus
    participant Payments as payments/
    participant Notifications as notifications/

    Note over Paciente,Scheduling: El paciente cancela con menos de 24h de anticipación

    Paciente->>API: DELETE /api/scheduling/appointments/:appointmentId
    API->>Scheduling: CancelAppointmentCommand { appointmentId }
    Note over Scheduling: Valida política: cancelación <24h antes<br/>→ sin derecho a reembolso<br/>Actualiza Cita → CANCELADA<br/>Libera el espacio del médico
    Scheduling-->>API: { status: "cancelled", reembolso: false }
    API-->>Paciente: 200 OK<br/>{ message: "Cita cancelada. Sin reembolso por política de cancelación tardía." }

    Scheduling-)EventBus: CitaCancelada { appointmentId, horasAnticipación: 8 }

    EventBus-)Payments: CitaCancelada
    Note over Payments: Consulta política: <24h → sin reembolso<br/>Mantiene Orden en estado COMPLETADA<br/>No genera Transacción de reembolso

    EventBus-)Notifications: CitaCancelada
    Note over Notifications: Notifica cancelación al paciente<br/>Informa explícitamente la política aplicada<br/>Notifica al médico la liberación del espacio
```

---

### Camino de Fallo: Reembolso Fallido en Stripe

```mermaid
sequenceDiagram
    participant Payments as payments/
    participant Stripe as Stripe API
    participant EventBus as Event Bus
    participant Notifications as notifications/

    Note over Payments: Intento de reembolso automático tras CitaCancelada

    Payments->>Stripe: Reembolso { chargeId, amount: 500 }
    Stripe-->>Payments: { status: "failed", code: "charge_already_refunded" }
    Note over Payments: Registra fallo en Transacción<br/>Marca Orden para revisión manual<br/>Alerta al equipo de operaciones

    Payments-)EventBus: ReembolsoFallido { appointmentId, motivo: "charge_already_refunded" }

    EventBus-)Notifications: ReembolsoFallido
    Note over Notifications: Notifica al paciente que el reembolso<br/>requiere revisión manual<br/>Incluye número de caso para seguimiento
```

---

## Flujo 4: Consulta Completada y Creación de Historial Clínico

### Escenario de Negocio

El médico atiende al paciente, finaliza la consulta y crea el registro clínico correspondiente, incluyendo diagnóstico, notas clínicas y prescripción. Al firmar el registro, este queda inmutable. El flujo también actualiza las métricas de analítica del sistema.

---

### Happy Path

```mermaid
sequenceDiagram
    actor Médico
    participant API as API Gateway
    participant Scheduling as scheduling/
    participant EventBus as Event Bus
    participant ClinicalRecords as clinical-records/
    participant Analytics as analytics/
    participant DoctorProfiles as doctor-profiles/
    participant Notifications as notifications/

    Médico->>API: PATCH /api/scheduling/appointments/:appointmentId/complete
    API->>Scheduling: CompleteAppointmentCommand { appointmentId }
    Note over Scheduling: Valida que la cita está CONFIRMADA<br/>Valida que la hora de inicio ya pasó<br/>Actualiza Cita → COMPLETADA
    Scheduling-->>API: { status: "completed" }
    API-->>Médico: 200 OK

    Scheduling-)EventBus: CitaCompletada { appointmentId, pacienteId, médicoId }

    EventBus-)ClinicalRecords: CitaCompletada
    Note over ClinicalRecords: Crea RegistroClínico vacío<br/>estado: BORRADOR<br/>vinculado al appointmentId<br/>Solo el médico tratante puede editarlo

    EventBus-)Analytics: CitaCompletada
    Note over Analytics: Incrementa totalCompletadas<br/>en MétricaCita del médico para la fecha

    Médico->>API: PUT /api/clinical-records/:recordId<br/>{ notasSubjetivas, diagnóstico, codigoCIE10, plan }
    API->>ClinicalRecords: UpdateClinicalRecordCommand
    Note over ClinicalRecords: Actualiza campos del registro<br/>Valida que estado sea BORRADOR<br/>Valida que el médico sea el tratante
    ClinicalRecords-->>API: { recordId, status: "draft" }
    API-->>Médico: 200 OK — Registro actualizado

    Médico->>API: POST /api/clinical-records/:recordId/prescriptions<br/>{ medicamento, dosis, frecuencia, duración }
    API->>ClinicalRecords: AddPrescriptionCommand
    Note over ClinicalRecords: Agrega Prescripción al registro<br/>Validaciones de campos requeridos
    ClinicalRecords-->>API: { prescriptionId }
    API-->>Médico: 201 Created

    Médico->>API: POST /api/clinical-records/:recordId/sign
    API->>ClinicalRecords: SignRecordCommand { recordId, médicoId }
    Note over ClinicalRecords: Valida integridad del registro<br/>Establece firmadoEn: DateTime.now()<br/>Estado → FIRMADO (inmutable)<br/>Solo lectura a partir de este punto
    ClinicalRecords-->>API: { recordId, firmadoEn }
    API-->>Médico: 200 OK — Registro firmado

    ClinicalRecords-)EventBus: RegistroFirmado { recordId, pacienteId, médicoId }

    EventBus-)Notifications: RegistroFirmado
    Note over Notifications: Notifica al paciente que su registro<br/>clínico está disponible para consulta

    EventBus-)DoctorProfiles: CitaCompletada
    Note over DoctorProfiles: Habilita al paciente para registrar<br/>una calificación para este médico
```

---

### Camino de Fallo: Intento de Modificar un Registro ya Firmado

```mermaid
sequenceDiagram
    actor Médico
    participant API as API Gateway
    participant ClinicalRecords as clinical-records/

    Note over Médico,ClinicalRecords: El médico intenta editar un registro<br/>que ya fue firmado digitalmente

    Médico->>API: PUT /api/clinical-records/:recordId<br/>{ diagnóstico: "Diagnóstico corregido" }
    API->>ClinicalRecords: UpdateClinicalRecordCommand
    Note over ClinicalRecords: Verifica estado del registro<br/>→ Estado: FIRMADO<br/>→ Operación no permitida
    ClinicalRecords-->>API: 409 Conflict
    API-->>Médico: 409 — Registro inmutable<br/>{ error: "El registro ya fue firmado y no puede modificarse.",<br/>  suggestion: "Cree una nota de addendum si requiere agregar información." }
```

---

### Camino de Fallo: Médico No Autorizado Intenta Acceder al Registro

```mermaid
sequenceDiagram
    actor OtroMédico as Médico No Autorizado
    participant API as API Gateway
    participant ClinicalRecords as clinical-records/
    participant Identity as identity/

    OtroMédico->>API: GET /api/clinical-records/:recordId
    API->>Identity: ValidateToken (AuthGuard)
    Identity-->>API: { userId, rol: MÉDICO }
    API->>ClinicalRecords: GetClinicalRecordQuery { recordId, requesterId }
    Note over ClinicalRecords: Verifica que requesterId<br/>sea el médico tratante del registro<br/>→ No coincide
    ClinicalRecords-->>API: 403 Forbidden
    API-->>OtroMédico: 403 — Acceso denegado<br/>{ error: "Solo el médico tratante puede acceder a este registro." }
```

---

## Resumen de Flujos

| Flujo | Módulos Involucrados | Tipo de Comunicación | Patrones Aplicados |
|---|---|---|---|
| Registro y verificación | Identity, PatientProfiles, Notifications | Síncrona + Asíncrona | OHS/PL (JWT), ACL (perfil) |
| Agendamiento y pago | Scheduling, Payments, Notifications | Síncrona + Asíncrona | Partnership (Scheduling ↔ Payments) |
| Cancelación y reembolso | Scheduling, Payments, Notifications, Analytics | Asíncrona dominante | Partnership, CF |
| Consulta y registro clínico | Scheduling, ClinicalRecords, DoctorProfiles, Analytics, Notifications | Asíncrona dominante | ACL (ClinicalRecords ← Scheduling) |
