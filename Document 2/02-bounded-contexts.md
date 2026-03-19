# 02 - Bounded Contexts
**Healthcare Scheduling System**


## 1. Introducción
Este documento descompone el dominio del sistema aplicando **Domain-Driven Design (DDD)** para identificar y definir los bounded contexts del Healthcare Scheduling System. Como parte de esto, se identificaron **bounded contexts**, cada bounded context representa una frontera explícita dentro de la cual un modelo de dominio es coherente, y consistente, es decir, los mismos términos significan exactamente lo mismo para todos los miembros del equipo que trabajan dentro de ese contexto. Los contextos se comunican mediante patrones de integración explícitos para evitar el acoplamiento estructural.

### Patrones de Integración utilizados

| Patrón | Sigla | Descripción |
|---|---|---|
| Open Host Service / Published Language | OHS/PL | El contexto upstream expone una API pública bien definida que cualquier downstream puede consumir |
| Anticorruption Layer | ACL | El contexto downstream traduce el modelo externo a su propio lenguaje interno para no corromperse |
| Conformist | CF | El downstream adopta el modelo del upstream sin traducción, aceptando su estructura |
| Shared Kernel | SK | Dos contextos comparten un subconjunto de código o modelo de dominio de forma explícita y controlada |
| Partnership | P | Dos contextos coordinan sus cambios mutuamente; ninguno puede evolucionar sin coordinar con el otro |

---

## 2. Bounded Contexts
### BC-01: Gestión de Identidad y Acceso (Identity and Access Management)
| Campo | Descripción |
|---|---|
| **Nombre** | Identity & Access Management |
| **Propósito** | Gestiona el registro, autenticación y autorización de todos los usuarios del sistema (pacientes, médicos, recepcionistas, administradores) y controla qué acciones puede realizar cada uno según su rol. Es la puerta de entrada a cualquier funcionalidad del sistema. |

**Entidades Principales**

| Entidad | Campos Clave | Restricciones |
|---|---|---|
| `User` | `id: UUID`, `email: string`, `passwordHash: string`, `rol: Enum`, `estadoCuenta: Enum`, `creadoEn: DateTime` | Email único en todo el sistema; role = PACIENTE, MÉDICO, ADMIN; estado de cuenta = ACTIVO, SUSPENDIDO, PENDIENTE_VERIFICACIÓN; password mínimo 8 caracteres |
| `Role` | `id: UUID`, `name: String`, `permissions: Permission[]` | Un usuario tiene exactamente un rol; los permisos son inmutables en runtime |
| `Session` | `id: UUID`, `usuarioId: UUID`, `tokenAcceso: string`, `tokenRefresco: string`, `expiraEn: DateTime`, `ip: string` | Un usuario puede tener múltiples sesiones activas |
| `EmailVerification` | `id: UUID`, `usuarioId: UUID`, `token: string`, `expiraEn: DateTime`, `usado: boolean` | Token de un solo uso, expira en 24h |


**Eventos de Dominio Publicados**
| Evento | Descripción |
|---|---|
| `UserRegistered` | Un nuevo usuario completó el registro y verificó su email |
| `UserLoggedIn` | Un usuario inició sesión exitosamente |
| `UserSuspended` | Un administrador suspendió una cuenta de usuario |
| `PasswordChanged` | Un usuario cambió su contraseña exitosamente |

**Eventos de Dominio Consumidos**

| Evento | Origen | Reacción |
|---|---|---|
| `PaymentFailed` (repetido) | Payments | Suspender cuenta si hay 3+ fallos de pago consecutivos |
| `AppointmentFraudDetected` | Scheduling Core | Marcar usuario para revisión manual |

**Contextos Upstream:** Ninguno — Identity Access Management es el contexto más fundamental del sistema.

**Contextos Downstream:** Scheduling Core, Payments, Notifications, Tenant Management, Reporting — todos dependen de Identity Access Management para validar identidad.

**Patrón de Integración:** **OHS/PL** — Identity Access Management expone un endpoint de validación de tokens (`GET /auth/validate`) y publica eventos de usuario. Los contextos downstream consumen esta API pública sin acceder al modelo interno de IAM.

---

## BC-02: Perfiles Médicos (Doctor Profiles)

| Campo | Descripción |
|---|---|
| **Nombre** | Doctor Profiles |
| **Propósito** | Administra la información profesional de los médicos: especialidades, credenciales, tarifas y disponibilidad base. Permite que los pacientes descubran y evalúen a los profesionales de salud. |

---
