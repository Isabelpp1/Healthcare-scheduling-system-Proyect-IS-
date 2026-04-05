# Diagrama 1 — Contexto del Sistema (C4 Nivel 1)

## Sistema de Programación de Citas Médicas

---

## Diagrama

```mermaid
C4Context
    title Sistema de Programación de Citas Médicas — Contexto del Sistema

    Person(paciente, "Paciente", "Busca médicos, agenda citas, realiza pagos y consulta su historial clínico.")
    Person(medico, "Médico", "Gestiona su agenda, atiende citas y crea registros clínicos.")
    Person(admin, "Administrador / Recepcionista", "Verifica credenciales médicas, gestiona la clínica y supervisa reportes.")

    System(sistema, "Sistema de Programación de Citas Médicas", "Plataforma web que conecta pacientes con médicos. Gestiona agendamiento, pagos, historial clínico y notificaciones.")

    System_Ext(stripe, "Stripe", "Pasarela de pago. Procesa cobros, reembolsos y emite webhooks de confirmación.")
    System_Ext(sendgrid, "SendGrid", "Servicio de email transaccional. Envía confirmaciones, recordatorios y comprobantes de pago.")
    System_Ext(twilio, "Twilio", "Servicio de SMS. Envía recordatorios de cita y alertas urgentes por mensaje de texto.")
    System_Ext(fcm, "Firebase Cloud Messaging", "Servicio de notificaciones push para la aplicación móvil.")

    Rel(paciente, sistema, "Registra cuenta, busca médicos, agenda citas y paga consultas", "HTTPS")
    Rel(medico, sistema, "Gestiona disponibilidad, completa consultas y crea registros clínicos", "HTTPS")
    Rel(admin, sistema, "Verifica médicos, administra clínica y genera reportes", "HTTPS")

    Rel(sistema, stripe, "Procesa cobros y reembolsos", "HTTPS / REST")
    Rel(stripe, sistema, "Confirma resultados de pago vía webhook", "HTTPS / Webhook POST")
    Rel(sistema, sendgrid, "Envía emails transaccionales", "HTTPS / REST")
    Rel(sistema, twilio, "Envía SMS de recordatorio y alerta", "HTTPS / REST")
    Rel(sistema, fcm, "Envía notificaciones push", "HTTPS / REST")
```

---

## Explicación

Este diagrama representa el **nivel 1 del modelo C4**: el sistema completo como una única caja negra, rodeado por los actores humanos y los sistemas externos con los que se integra.

### Actores Humanos

El sistema atiende a tres roles diferenciados. El **Paciente** es el usuario principal: busca médicos por especialidad, revisa disponibilidad, agenda citas y realiza el pago en línea. El **Médico** gestiona su propia agenda de disponibilidad, marca citas como completadas y redacta los registros clínicos correspondientes. El **Administrador / Recepcionista** verifica las credenciales profesionales de los médicos antes de que puedan operar en la plataforma, y accede a los reportes operacionales y financieros de la clínica.

### Sistemas Externos

**Stripe** es la pasarela de pago principal. El sistema le envía solicitudes de cobro y reembolso vía REST, y Stripe responde con el resultado final a través de webhooks hacia el endpoint `/api/payments/webhook`. La comunicación es bidireccional y asíncrona en el caso de los webhooks.

**SendGrid** recibe solicitudes REST del sistema para el envío de correos transaccionales: bienvenida al registrarse, confirmación de cita, comprobante de pago, aviso de cancelación y recordatorios programados 24 horas antes de la consulta.

**Twilio** complementa el canal de email con SMS para recordatorios de cita. Se usa especialmente para usuarios que no tienen la aplicación instalada o que prefieren comunicación directa por mensaje de texto.

**Firebase Cloud Messaging (FCM)** gestiona las notificaciones push hacia la aplicación móvil, permitiendo alertas en tiempo real cuando la cita es confirmada, cuando se aproxima la fecha o cuando ocurre un cambio en el estado de la cita.

### Decisiones de Diseño Notables

- El sistema **no expone datos clínicos a ningún sistema externo**. Stripe, SendGrid y Twilio solo reciben los datos mínimos necesarios para cumplir su función (monto, email de destinatario, número de teléfono).
- Los **webhooks de Stripe** entran por un endpoint dedicado con validación de firma (`Stripe-Signature`), lo que previene solicitudes maliciosas que intenten simular confirmaciones de pago falsas.
- La elección de **tres canales de notificación independientes** (email, SMS, push) garantiza que los recordatorios lleguen al usuario incluso si uno de los canales falla, respetando las preferencias configuradas por cada usuario.
