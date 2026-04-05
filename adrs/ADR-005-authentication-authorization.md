# ADR-005: Autenticación y Autorización — JWT Auto-gestionado con RBAC

| Campo        | Valor                                     |
|--------------|-------------------------------------------|
| Estado       | Accepted                                  |
| Fecha        | 2026-03-31                                |
| Responsables | Isabel y Gabriela                         |

---

## Contexto

El sistema tiene cuatro roles de usuario con permisos distintos: Paciente, Médico, Recepcionista y Admin. Cada rol puede acceder a un subconjunto diferente de recursos y operaciones. Por ejemplo, solo un médico puede firmar registros clínicos, solo un paciente puede ver sus propias condiciones médicas, y solo un admin puede verificar credenciales médicas.

El módulo `identity/` es el upstream de todos los demás contextos. La solución de autenticación debe ser sencilla de integrar en todos los módulos del monolito, sin introducir una dependencia externa crítica en el camino de cada petición.

El sistema no requiere autenticación social en la primera versión, ni federación con sistemas externos de identidad hospitalaria. El volumen de usuarios esperado en etapa inicial es de cientos a pocos miles, no millones.

---

## Opciones Consideradas

### Opción 1: Proveedor de Identidad Gestionado (Auth0 / AWS Cognito / Firebase Auth)

Servicio externo que gestiona el ciclo de vida completo de usuarios, sesiones, tokens y flujos de recuperación.

| Pros | Contras |
|---|---|
| Sin código de autenticación propio: registro, login, MFA, recuperación de contraseña gestionados  | Dependencia externa crítica: si el proveedor tiene downtime, nadie puede autenticarse |
| Flujos de MFA y detección de anomalías incluidos                                                  | El modelo de roles y permisos puede no mapear exactamente a los 4 roles del sistema |
| Ideal si se necesita federación con otros sistemas de salud (SSO)                                 | Los datos de usuarios quedan en infraestructura de un tercero lo cual puede ser sensible para un sistema de salud |
|                                                                                                   | Personalizar el modelo de autorización requiere lógica adicional fuera del proveedor |

### Opción 2: JWT Auto-gestionado con RBAC Propio

El módulo `identity/` emite tokens JWT firmados con clave secreta propia. El rol del usuario se incluye en el payload del token. Cada módulo valida el token localmente con `AuthGuard` sin hacer una llamada externa en cada petición.

| Pros | Contras |
|---|---|
| Sin dependencias externas en el critical path de autenticación                                | El equipo es responsable de la seguridad de la implementación (hashing, rotación de claves, expiración) |
| El rol viaja en el token, es decir, autorización sin consulta a base de datos por petición    | La revocación inmediata de tokens requiere un mecanismo adicional |
| Control total sobre el modelo de roles y la granularidad de permisos                          | Si la clave de firma se compromete, todos los tokens activos son vulnerables |
| Latencia mínima (~1ms)                                                                        |  |
| Sin costo adicional ni límites de usuarios activos                                            |  |
| Datos de identidad permanecen en la infraestructura propia                                    |  |

### Opción 3: Sesiones en Base de Datos (sin JWT)

El servidor emite un `sessionId` opaco al cliente. En cada petición, se consulta la base de datos para verificar si la sesión es válida y recuperar el usuario.

| Pros | Contras |
|---|---|
| Revocación inmediata: borrar la sesión de la DB invalida el acceso al instante | Una consulta a la base de datos por cada petición autenticada implica latencia y carga adicional |
| Sin lógica de expiración de tokens en el cliente                               | No funciona bien con múltiples réplicas del proceso sin sesiones compartidas (requiere Redis o DB centralizada) |
|                                                                                | No es stateless, es decir, dificulta el escalado horizontal sin infraestructura adicional |

---

## Decisión

Se optó por adoptar la Opción 2, JWT auto-gestionado con RBAC propio.

Razones específicas al proyecto:

1. **Volumen y etapa:** Con cientos a pocos miles de usuarios activos en la etapa inicial, el costo y la complejidad de Auth0 o Cognito no están justificados. El equipo puede implementar autenticación segura con JWT en menos tiempo del que tomaría integrar y personalizar un proveedor externo.

2. **Datos sensibles:** Un sistema de salud maneja datos de identidad vinculados a historiales médicos. Mantener estos datos en infraestructura propia reduce la superficie de exposición a terceros.

3. **Sin dependencia en el critical path:** La validación de JWT es local a cada módulo mediante `AuthGuard`. Una eventual degradación del módulo `identity/` solo afecta los nuevos inicios de sesión, no las sesiones activas.

4. **RBAC suficiente para los 4 roles:** El sistema no requiere permisos dinámicos ni claims complejos por ahora. El rol en el token es suficiente para todas las reglas de autorización.

**Modelo de Roles (RBAC):**

| Rol | Permisos Principales |
|---|---|
| `PACIENTE`        | Ver y editar su propio perfil, buscar médicos, agendar/cancelar sus citas, ver sus registros clínicos, realizar pagos, calificar médicos |
| `MÉDICO`          | Ver y editar su perfil profesional, gestionar su disponibilidad, ver sus citas, crear/firmar registros clínicos de sus propias consultas, ver sus métricas de analítica |
| `RECEPCIONISTA`   | Ver citas de todos los médicos de la clínica, agendar citas en nombre de pacientes, marcar asistencia/inasistencia |
| `ADMIN`           | Todo lo anterior y además verificar credenciales médicas, suspender cuentas, ver analítica global de la clínica, gestionar tarifas y configuración |

**Configuración de tokens:**
- Access token JWT: expira en 15 minutos (corto para minimizar el riesgo de tokens comprometidos).
- Refresh token opaco en base de datos, expira en 7 días, rotación en cada uso.
- Algoritmo de firma: RS256 (la clave privada nunca sale del módulo `identity/`).

**Revocación:**
- Las sesiones activas se almacenan en la tabla `Sesion` del módulo `identity/`.
- Al suspender una cuenta (`CuentaSuspendida`), todas sus sesiones se invalidan en la tabla.
- El access token de 15 minutos limita la ventana de exposición ante un token comprometido sin necesidad de una lista negra consultada en cada petición.

---

## Consecuencias

Habilitado por esta decisión:
- Autenticación sin latencia adicional por petición (validación local del JWT).
- Control total del modelo de roles sin depender de la configuración de un tercero.
- Los datos de identidad permanecen en la infraestructura propia del sistema.

Prevenido por esta decisión:
- No hay SSO con sistemas externos de salud (HIS hospitalarios) en esta versión. Mitigación: si se requiere en el futuro, se puede agregar un flujo OAuth2/OIDC sobre la misma base de `identity/` sin reescribir los demás módulos.
- La revocación inmediata de access tokens no es instantánea (ventana de hasta 15 minutos). Mitigación: el refresh token sí es revocable inmediatamente; para casos críticos (cuenta suspendida), se puede añadir una lista negra en Redis con TTL igual al tiempo de expiración del access token.

Riesgos:
| Riesgo | Mitigación |
|--------|------------|
| La clave privada RS256 se compromete | Rotación de claves semestral, almacenamiento en variables de entorno secretas, soporte para múltiples claves activas simultáneamente durante la transición |
| El equipo implementa incorrectamente la validación de tokens | Usar la librería `@nestjs/jwt` + `passport-jwt` para no implementar la validación manualmente |

---

## ADRs Relacionados

- **ADR-001** — El monolito modular permite que `AuthGuard` sea un componente compartido importado desde `modules/identity/index.ts` por todos los módulos, sin necesidad de un servicio de autenticación separado.
