# 03 - Descomposición de Módulos y Servicios

## Sistema de Programación de Citas Médicas



## Introducción

Este documento describe la organización concreta del código fuente del sistema bajo la arquitectura: **Monolito Modular**. Cada módulo de alto nivel mapea directamente a uno o más de un bounded contexts definidos en el Documento 2. Los límites entre módulos se refuerzan mediante reglas estructurales explícitas: sin importaciones cruzadas directas, APIs internas bien definidas y comunicación asíncrona a través de un bus de eventos interno.


## Árbol de Directorios

```
medical-scheduling/
│
├── apps/
│   ├── api/                          # Punto de entrada HTTP 
│   │   ├── src/
│   │   │   ├── main.ts               # Bootstrap de la aplicación
│   │   │   ├── app.module.ts         # Módulo raíz — registra todos los módulos de dominio
│   │   │   └── middleware/
│   │   │       ├── auth.middleware.ts
│   │   │       └── logger.middleware.ts
│   │   └── package.json
│   │
│   └── worker/                       # Proceso independiente para tareas de fondo
│       ├── src/
│       │   ├── main.ts
│       │   └── jobs/
│       │       ├── reminder.job.ts   # Recordatorios de citas 
│       │       └── report.job.ts     # Generación de reportes 
│       └── package.json
│
├── modules/                          #  Núcleo del monolito modular
│   │
│   ├── identity/                     # BC-01: Identity & Access
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── user.entity.ts
│   │   │   │   └── session.entity.ts
│   │   │   ├── events/
│   │   │   │   ├── user-registered.event.ts
│   │   │   │   └── user-verified.event.ts
│   │   │   └── ports/
│   │   │       └── identity.repository.port.ts
│   │   ├── application/
│   │   │   ├── commands/
│   │   │   │   ├── register-user.command.ts
│   │   │   │   └── login.command.ts
│   │   │   └── queries/
│   │   │       └── get-user-by-id.query.ts
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   │   └── user.repository.ts
│   │   │   └── http/
│   │   │       └── auth.controller.ts
│   │   └── index.ts                  # API pública del módulo (barrel export)
│   │
│   ├── doctor-profiles/              # BC-02: Doctor Profiles
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── doctor-profile.entity.ts
│   │   │   │   ├── specialty.entity.ts
│   │   │   │   └── consultation-fee.entity.ts
│   │   │   ├── events/
│   │   │   │   ├── doctor-verified.event.ts
│   │   │   │   └── rating-registered.event.ts
│   │   │   └── ports/
│   │   │       └── doctor-profile.repository.port.ts
│   │   ├── application/
│   │   │   ├── commands/
│   │   │   │   └── create-doctor-profile.command.ts
│   │   │   └── queries/
│   │   │       └── get-doctor-availability.query.ts
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   │   └── doctor-profile.repository.ts
│   │   │   └── http/
│   │   │       └── doctor.controller.ts
│   │   └── index.ts
│   │
│   ├── patient-profiles/             # BC-03: Patient Profiles
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── patient-profile.entity.ts
│   │   │   │   └── medical-condition.entity.ts
│   │   │   ├── events/
│   │   │   │   └── patient-profile-created.event.ts
│   │   │   └── ports/
│   │   │       └── patient-profile.repository.port.ts
│   │   ├── application/
│   │   │   ├── commands/
│   │   │   │   └── create-patient-profile.command.ts
│   │   │   └── queries/
│   │   │       └── get-patient-profile.query.ts
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   │   └── patient-profile.repository.ts
│   │   │   └── http/
│   │   │       └── patient.controller.ts
│   │   └── index.ts
│   │
│   ├── scheduling/                   # BC-04: Scheduling (Core Domain)
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── appointment.entity.ts
│   │   │   │   ├── availability.entity.ts
│   │   │   │   └── schedule-block.entity.ts
│   │   │   ├── value-objects/
│   │   │   │   ├── appointment-status.vo.ts
│   │   │   │   └── time-slot.vo.ts
│   │   │   ├── events/
│   │   │   │   ├── appointment-requested.event.ts
│   │   │   │   ├── appointment-confirmed.event.ts
│   │   │   │   ├── appointment-cancelled.event.ts
│   │   │   │   └── appointment-completed.event.ts
│   │   │   ├── services/
│   │   │   │   └── slot-calculator.service.ts   # Lógica de disponibilidad
│   │   │   └── ports/
│   │   │       └── scheduling.repository.port.ts
│   │   ├── application/
│   │   │   ├── commands/
│   │   │   │   ├── request-appointment.command.ts
│   │   │   │   ├── confirm-appointment.command.ts
│   │   │   │   ├── cancel-appointment.command.ts
│   │   │   │   └── complete-appointment.command.ts
│   │   │   └── queries/
│   │   │       ├── get-available-slots.query.ts
│   │   │       └── get-appointment-by-id.query.ts
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   │   └── appointment.repository.ts
│   │   │   └── http/
│   │   │       └── scheduling.controller.ts
│   │   └── index.ts
│   │
│   ├── payments/                     # BC-05: Payments
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── order.entity.ts
│   │   │   │   └── transaction.entity.ts
│   │   │   ├── events/
│   │   │   │   ├── payment-confirmed.event.ts
│   │   │   │   ├── payment-failed.event.ts
│   │   │   │   └── refund-completed.event.ts
│   │   │   └── ports/
│   │   │       ├── order.repository.port.ts
│   │   │       └── payment-gateway.port.ts      # Puerto hacia proveedor externo
│   │   ├── application/
│   │   │   ├── commands/
│   │   │   │   ├── create-order.command.ts
│   │   │   │   └── process-refund.command.ts
│   │   │   └── queries/
│   │   │       └── get-order-status.query.ts
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   │   └── order.repository.ts
│   │   │   ├── gateways/
│   │   │   │   └── stripe.gateway.ts            # ACL hacia Stripe
│   │   │   └── http/
│   │   │       ├── payments.controller.ts
│   │   │       └── stripe-webhook.controller.ts
│   │   └── index.ts
│   │
│   ├── notifications/                # BC-06: Notifications
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── notification.entity.ts
│   │   │   │   └── message-template.entity.ts
│   │   │   ├── events/
│   │   │   │   └── notification-sent.event.ts
│   │   │   └── ports/
│   │   │       └── notification-channel.port.ts
│   │   ├── application/
│   │   │   └── commands/
│   │   │       └── send-notification.command.ts
│   │   ├── infrastructure/
│   │   │   ├── channels/
│   │   │   │   ├── email.channel.ts             # Adapter - SendGrid/SES
│   │   │   │   ├── sms.channel.ts               # Adapter - Twilio
│   │   │   │   └── push.channel.ts              # Adapter - FCM
│   │   │   └── http/
│   │   │       └── notifications.controller.ts
│   │   └── index.ts
│   │
│   ├── clinical-records/             # BC-07: Clinical Records
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── clinical-record.entity.ts
│   │   │   │   ├── prescription.entity.ts
│   │   │   │   └── attached-document.entity.ts
│   │   │   ├── events/
│   │   │   │   ├── record-created.event.ts
│   │   │   │   └── record-signed.event.ts
│   │   │   └── ports/
│   │   │       └── clinical-record.repository.port.ts
│   │   ├── application/
│   │   │   ├── commands/
│   │   │   │   ├── create-clinical-record.command.ts
│   │   │   │   └── sign-record.command.ts
│   │   │   └── queries/
│   │   │       └── get-patient-records.query.ts
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   │   └── clinical-record.repository.ts
│   │   │   └── http/
│   │   │       └── clinical-records.controller.ts
│   │   └── index.ts
│   │
│   └── analytics/                    # BC-08: Analytics
│       ├── domain/
│       │   ├── entities/
│       │   │   ├── appointment-metric.entity.ts
│       │   │   └── revenue-metric.entity.ts
│       │   └── ports/
│       │       └── metrics.repository.port.ts
│       ├── application/
│       │   └── queries/
│       │       ├── get-doctor-metrics.query.ts
│       │       └── get-revenue-report.query.ts
│       ├── infrastructure/
│       │   ├── persistence/
│       │   │   └── metrics.repository.ts
│       │   └── http/
│       │       └── analytics.controller.ts
│       └── index.ts
│
├── shared/                           # Código compartido sin lógica de dominio
│   ├── domain/
│   │   ├── event-bus.interface.ts    # Contrato del bus de eventos interno
│   │   └── base.entity.ts
│   ├── infrastructure/
│   │   ├── event-bus/
│   │   │   └── in-memory-event-bus.ts
│   │   ├── database/
│   │   │   └── database.module.ts
│   │   └── config/
│   │       └── env.config.ts
│   └── utils/
│       ├── pagination.ts
│       └── date-utils.ts
│
├── docker-compose.yml
├── package.json
└── tsconfig.json
```

---

## Descripción de Módulos de Alto Nivel

### `modules/identity/`
**Bounded Context:** BC-01 Identity & Access

**Qué posee:** Registro de usuarios, login, refresh de tokens JWT, verificación de email, restablecimiento de contraseña, gestión de sesiones.

**Qué expone (API pública vía `index.ts`):**
```typescript
// Funciones exportadas hacia otros módulos
export { AuthGuard }           // Guard reutilizable para proteger nuestras rutas
export { CurrentUser }         // Decorator para poder extraer los usuario del token
export { UserRole }            // Enum de roles compartido
```
Todos los endpoints REST propios los estaremos registrando bajo el prefijo `/api/auth/`.

---

### `modules/doctor-profiles/`
**Bounded Context:** BC-02 Doctor Profiles

**Qué posee:** CRUD de perfiles médicos, catálogo de especialidades, gestión de tarifas, registro de calificaciones.

**Qué expone:**
```typescript
export { DoctorProfileSummaryDto }  // DTO de solo lectura para otros módulos
export { DoctorVerifiedEvent }      // Evento consumible por Scheduling
```
Endpoints REST bajo `/api/doctors/`.

---

### `modules/patient-profiles/`
**Bounded Context:** BC-03 Patient Profiles

**Qué posee:** CRUD de perfil del paciente, condiciones médicas, contactos de emergencia.

**Qué expone:**
```typescript
export { PatientSummaryDto }        // DTO de solo lectura para Scheduling
```
Endpoints REST bajo `/api/patients/`.

---

### `modules/scheduling/`
**Bounded Context:** BC-04 Scheduling 

**Qué posee:** Gestión de disponibilidad de médicos, ciclo de vida completo de citas, cálculo de slots disponibles, máquina de estados de citas.

**Qué expone:**
```typescript
export { AppointmentConfirmedEvent }   // Consumido por Notifications, Analytics
export { AppointmentCancelledEvent }   // Consumido por Payments, Notifications
export { AppointmentCompletedEvent }   // Consumido por Clinical Records, Analytics
```
Endpoints REST bajo `/api/scheduling/`.

Este módulo **no importa directamente** de `payments/` ni de `notifications/`. Se comunica con ellos exclusivamente a través del bus de eventos.

---

### `modules/payments/`
**Bounded Context:** BC-05 Payments

**Qué posee:** Creación y seguimiento de órdenes de pago, procesamiento vía Stripe (ACL), gestión de reembolsos, recepción de webhooks externos.

**Qué expone:**
```typescript
export { PaymentConfirmedEvent }    // Consumido por Scheduling y Notifications
export { PaymentFailedEvent }       // Consumido por Scheduling
export { RefundCompletedEvent }     // Consumido por Notifications, Analytics
```
Endpoints REST bajo `/api/payments/`. Webhook de Stripe en `/api/payments/webhook`.

---

### `modules/notifications/`
**Bounded Context:** BC-06 Notifications

**Qué posee:** Envío de emails (SendGrid), SMS (Twilio) y push (FCM), gestión de plantillas, preferencias de notificación por usuario.

**Qué expone:** Ninguna exportación pública. Es un consumidor puro de eventos; nunca es llamado directamente por otros módulos.

---

### `modules/clinical-records/`
**Bounded Context:** BC-07 Clinical Records

**Qué posee:** Creación y firma de registros clínicos, emisión de prescripciones, adjunto de documentos médicos, control de acceso a nivel de registro.

**Qué expone:** Ninguna exportación hacia otros módulos. Solo expone endpoints REST bajo `/api/clinical-records/` para médicos y pacientes autorizados.

---

### `modules/analytics/`
**Bounded Context:** BC-08 Analytics

**Qué posee:** Proyecciones de métricas agregadas (solo lectura), generación de reportes periódicos. Actualiza sus modelos al consumir eventos de otros contextos.

**Qué expone:** Endpoints REST bajo `/api/analytics/` (solo roles ADMIN y MÉDICO).

---

### `apps/worker/`
**Proceso independiente** que comparte la misma base de código pero corre en un proceso separado.

**Por qué proceso propio:** Las tareas de fondo (recordatorios 24h antes de la cita, generación semanal de reportes) no deben bloquear el proceso HTTP principal. Permiten reinicio independiente sin interrumpir las peticiones activas.

**Jobs:**
- `reminder.job.ts` — Consulta citas próximas y publica eventos de recordatorio (cron: cada hora)
- `report.job.ts` — Genera reportes periódicos para Analytics (cron: domingos 23:00)

---

## Cómo se Refuerzan los Límites de Contexto

### Regla 1: Sin importaciones cruzadas entre módulos de dominio

```typescript
//inncorrect: scheduling importando directamente de payments
import { OrderService } from '../payments/application/order.service';

// correcto: — scheduling reacciona a un evento publicado por payments
@OnEvent('PaymentConfirmed')
async handlePaymentConfirmed(event: PaymentConfirmedEvent) {
  await this.confirmAppointment(event.appointmentId);
}
```

Cada módulo solo puede importar desde:
- Su propio directorio (`./`)
- El directorio `shared/` (utilidades y contratos sin lógica de dominio)
- El `index.ts` (barrel export) de otro módulo — nunca rutas internas

### Regla 2: Comunicación asíncrona entre contextos vía Event Bus

El bus de eventos interno (`shared/infrastructure/event-bus/`) desacopla a los productores de los consumidores. En desarrollo usa un bus en memoria; en producción puede reemplazarse por RabbitMQ sin cambiar el código de los módulos.

```typescript
// Productor (scheduling) — no sabe quién escucha
this.eventBus.publish(new AppointmentConfirmedEvent(appointmentId, patientId, doctorId));

// Consumidor (notifications) — no sabe quién publicó
@OnEvent('AppointmentConfirmed')
async sendConfirmationNotification(event: AppointmentConfirmedEvent) { ... }
```

### Regla 3: API pública explícita por módulo (barrel exports)

Cada módulo expone únicamente lo que está en su `index.ts`. El resto de sus archivos internos son privados por convención. Esto imita el concepto de `public` / `internal` en lenguajes compilados y previene el acoplamiento accidental.

### Regla 4: ACL en los adaptadores de infraestructura

Las integraciones externas (Stripe, SendGrid, Twilio) están completamente aisladas dentro de la carpeta `infrastructure/` de su módulo correspondiente. El dominio nunca conoce los detalles del proveedor; solo interactúa con el puerto (interfaz) definido en `domain/ports/`.

```typescript
// Puerto (dominio no conoce a Stripe)
export interface PaymentGatewayPort {
  charge(amount: number, currency: string, token: string): Promise<ChargeResult>;
  refund(transactionId: string, amount: number): Promise<RefundResult>;
}

// Adaptador (ACL que traduce al modelo de Stripe)
export class StripeGateway implements PaymentGatewayPort { ... }
```

---

## Mapa Módulo → Bounded Context

| Módulo | Bounded Context | Proceso |
|---|---|---|
| `modules/identity/` | BC-01 Identity & Access | `apps/api` |
| `modules/doctor-profiles/` | BC-02 Doctor Profiles | `apps/api` |
| `modules/patient-profiles/` | BC-03 Patient Profiles | `apps/api` |
| `modules/scheduling/` | BC-04 Scheduling | `apps/api` |
| `modules/payments/` | BC-05 Payments | `apps/api` |
| `modules/notifications/` | BC-06 Notifications | `apps/api` + `apps/worker` |
| `modules/clinical-records/` | BC-07 Clinical Records | `apps/api` |
| `modules/analytics/` | BC-08 Analytics | `apps/api` + `apps/worker` |
