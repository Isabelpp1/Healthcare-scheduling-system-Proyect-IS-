# Componente: Proveedor de Autenticación (JWT Auto-gestionado)

## 1. Responsabilidades

- **Emisión de tokens JWT con RS256:** El módulo `identity/` emite access tokens (TTL: 15 minutos) firmados con clave privada RSA, y refresh tokens opacos (TTL: 7 días) almacenados en Redis con capacidad de revocación inmediata.
- **Validación sin consulta a base de datos:** El `AuthGuard` exportado desde `modules/identity/index.ts` valida el access token verificando la firma con la clave pública, sin realizar ninguna consulta a PostgreSQL en cada petición autenticada.
- **Control de acceso basado en roles (RBAC):** El payload del JWT incluye el campo `rol: Enum(PACIENTE, MÉDICO, RECEPCIONISTA, ADMIN)`. Los decoradores `@Roles()` en los controllers verifican el rol requerido para cada endpoint antes de ejecutar la lógica de negocio.
- **Revocación de sesiones:** Al suspender una cuenta (`CuentaSuspendida`), todas las entradas de refresh token del usuario se eliminan de Redis. El access token activo expira en máximo 15 minutos — ventana de exposición acotada sin necesidad de lista negra consultada por petición.
- **Verificación de email y recuperación de contraseña:** Gestiona el ciclo de vida de tokens de un solo uso almacenados en PostgreSQL bajo el esquema `identity`, con TTL de 24 horas y verificación de uso previo.

## 2. Por Qué Este Proyecto lo Necesita

El sistema maneja datos de salud de pacientes lo que implica información de las más sensibles en términos de privacidad y regulación. La decisión de gestionar la autenticación internamente está motivada por dos razones específicas del dominio: los datos de identidad del sistema están vinculados a historiales clínicos y registros de pago, y mantenerlos en infraestructura de terceros aumenta la superficie de exposición sin que el beneficio operacional justifique el riesgo en esta etapa.

El modelo RBAC de cuatro roles es suficientemente simple para implementarse con claims en el JWT, sin necesidad de un sistema de permisos dinámicos que justifique la complejidad de un proveedor externo. El beneficio clave de Auth0 (gestión de flujos OAuth/OIDC, social login, MFA) no es un requisito de la primera versión del sistema.

## 3. Elección de Tecnología

**Elección: JWT RS256 auto-gestionado con `@nestjs/jwt` + `passport-jwt`**

| Dimensión | JWT auto-gestionado *(Elección)* | Auth0 *(Alternativa)* | AWS Cognito *(Alternativa)* |
|---|---|---|---|
| **Managed / Self-hosted** | Self-hosted (lógica en `identity/`) | Managed (SaaS) | Managed (AWS) |
| **Complejidad operativa** | Media: el equipo implementa y mantiene el flujo completo | Muy baja: flujos de auth listos, SDK, dashboard | Baja:gestionado pero con curva de configuración |
| **Costo a nuestra escala** | $0 adicional | Gratuito hasta 7,500 MAU; $23/mes hasta 10K MAU | Gratuito hasta 50K MAU; luego $0.0055/MAU |
| **Feature diferenciador** | Control total del modelo de datos, sin dependencia externa en critical path, datos en infraestructura propia | MFA, social login, anomaly detection, compliance integrado | Integración nativa con AWS (API GW, Lambda, IAM), SAML/OIDC |

Para el volumen inicial y los requisitos actuales, el JWT auto-gestionado es la elección correcta. Auth0 y Cognito se justifican cuando la complejidad de los flujos de identidad supera lo que el equipo puede mantener razonablemente, ese umbral no se alcanza con cuatro roles y autenticación básica por email/password.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Sin dependencia externa en el critical path de autenticación: si el proveedor tiene downtime, las sesiones activas siguen funcionando | El equipo es responsable de la seguridad de la implementación (almacenamiento seguro de claves, rotación, validación correcta) |
| Datos de identidad en infraestructura propia — relevante para compliance en sistemas de salud | La revocación de access tokens no es instantánea (ventana de hasta 15 min) sin lista negra adicional en Redis |
| Costo cero  | Sin MFA nativo, debe implementarse manualmente si se requiere en futuras versiones |
| Validación de JWT es local (~1ms): sin latencia de red por petición autenticada | Rotación de claves RS256 requiere proceso cuidadoso para no invalidar tokens activos durante la transición |
| Total control sobre el modelo de roles y claims del token |  |

## 5. Integración

En el diagrama de despliegue, la lógica de autenticación vive dentro del proceso `apps/api` como parte del módulo `identity/`. No es un proceso separado. Las conexiones son:

```
apps/api (identity/) -> PostgreSQL :5432  (Usuarios, Sesiones, VerificaciónEmail)
apps/api (identity/) -> Redis :6379       (Refresh tokens con TTL, lista negra opcional)
apps/api (AuthGuard) -> [memoria proceso] (validación de firma JWT con clave pública RS256)
```

El `AuthGuard` se registra como middleware global en `apps/api/src/app.module.ts`, excepto para las rutas públicas (`POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/verify-email`, `POST /api/payments/webhook`). Todos los demás módulos importan únicamente `{ AuthGuard, CurrentUser, UserRole }` desde `modules/identity/index.ts`, nunca importan servicios internos del módulo de identidad, respetando la frontera de contexto del Documento 2.
