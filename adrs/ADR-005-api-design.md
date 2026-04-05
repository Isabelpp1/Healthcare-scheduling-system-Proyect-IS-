# ADR-006: Diseño de API — REST para API Externa, Comunicación Directa Interna

| Campo        | Valor                                     |
|--------------|-------------------------------------------|
| Estado       | Accepted                                  |
| Fecha        | 2026-03-31                                |
| Responsables | Isabel y Gabriela                         |

---

## Contexto

El sistema expone funcionalidad a dos tipos de consumidores distintos:

1. **Consumidores externos:** La aplicación web (SPA) y la aplicación móvil consumen la API para todas las operaciones de usuario: registro, búsqueda de médicos, agendamiento, pagos, visualización de historial.
2. **Comunicación interna entre módulos:** Dentro del monolito, los módulos se comunican entre sí. Esta comunicación no cruza la red.

La elección del estilo de API afecta la experiencia de desarrollo del frontend, el rendimiento percibido por el usuario, la documentación y el costo de mantenimiento del contrato API. Para la comunicación interna, la elección afecta el acoplamiento entre módulos y la facilidad de refactorización.

---

## Opciones Consideradas

### Opción 1: GraphQL para API Externa

Un único endpoint GraphQL que permite a los clientes solicitar exactamente los campos que necesitan. El frontend define sus propias queries y mutations.

| Pros | Contras |
|---|---|
| El cliente solicita exactamente los datos que necesita, sin over-fetching ni under-fetching   | Mayor complejidad inicial: schema definition, resolvers, DataLoader para evitar N+1 |
| Un solo endpoint para todos los módulos                                                       | Caching HTTP estándar no aplica (todas las queries van por POST) |
| Ideal para UIs con necesidades de datos heterogéneas y cambiantes                             | Curva de aprendizaje adicional para un equipo pequeño |
| Introspección del schema como documentación viva                                              | Los errores se devuelven con HTTP 200, lo que complica el monitoreo estándar |
|                                                                                               | Overkill para un sistema con patrones de acceso predecibles y bien definidos |

### Opción 2: gRPC para API Externa e Interna

Protocolo binario sobre HTTP/2 con contratos definidos en Protocol Buffers. Alta performance y tipado fuerte en ambos lados.

| Pros | Contras |
|---|---|
| Altísimo rendimiento: binario, multiplexado, streaming bidireccional                                  | Los navegadores no soportan gRPC directamente |
| Contratos fuertemente tipados (`.proto` files) en donde el cliente y servidor comparten el mismo tipo | Genera una capa de complejidad significativa para una API consumida desde un browser |
| Ideal para comunicación entre microservicios internos                                                 | El equipo de frontend necesita aprender a trabajar con Protocol Buffers |
|                                                                                                       | No aplica al contexto de un monolito: la comunicación interna no cruza la red |

### Opción 3: REST para API Externa, Llamadas Directas Internas

La API externa sigue principios REST con recursos nombrados, verbos HTTP semánticos, códigos de estado estándar y documentación OpenAPI. La comunicación interna entre módulos del monolito es directa (llamadas entre Application Services), sin cruzar la red.

| Pros | Contras |
|---|---|
| REST es el estándar más conocido por equipos frontend                     | Puede haber over-fetching en algunos endpoints |
| HTTP caching estándar aplicable a endpoints de solo lectura               | Múltiples endpoints para operaciones distintas |
| Códigos de estado HTTP semánticos facilitan el monitoreo y alertas        |  |
| OpenAPI/Swagger generado automáticamente con decoradores de NestJS        |  |
| La comunicación interna sin red es más rápida y no requiere serialización |  |

---

## Decisión

Se decidió adoptar REST para la API externa e invocación directa entre módulos para la comunicación interna.

Razones específicas al proyecto:

1. **Patrones de acceso predecibles:** Las operaciones del sistema son bien definidas (agendar cita, procesar pago, ver historial). No hay necesidad de queries ad-hoc que justifiquen GraphQL. Los endpoints REST reflejan directamente los casos de uso del Documento 4.

2. **Consumidores conocidos:** El frontend y la app móvil son desarrollados por el mismo equipo. La documentación OpenAPI generada automáticamente por NestJS es suficiente para la coordinación — no se necesita la introspección de GraphQL.

3. **Caching HTTP:** Los endpoints de solo lectura se benefician del caching HTTP estándar, algo no disponible con GraphQL.

4. **Sin overhead de red interno:** En el monolito, llamar a `schedulingService.getAvailableSlots()` directamente es más eficiente y más simple que serializar/deserializar una petición HTTP o gRPC interna.

Convenciones REST adoptadas:

| Aspecto | Convención |
|---|---|
| Recursos en plural y kebab-case   | `/api/clinical-records`, `/api/doctor-profiles` |
| Verbos HTTP semánticos            | `GET` leer, `POST` crear, `PUT/PATCH` actualizar, `DELETE` eliminar |
| Códigos de estado explícitos      | `201` creado, `202` aceptado (async), `409` conflicto, `422` validación |
| Versionamiento                    | Prefijo `/api/v1/`, sin versionamiento en la URL inicial, se añade `v1` solo si hay un breaking change |
| Paginación                        | `?page=1&limit=20` con envelope `{ data: [], meta: { total, page, limit } }` |
| Errores                           | `{ error: string, code: string, field?: string }` con código de error interno para el frontend |
| Autenticación                     | `Authorization: Bearer <jwt>` en todos los endpoints protegidos |

Documentación:
- Swagger UI disponible en `/api/docs` en entornos de desarrollo.
- Decoradores `@ApiOperation`, `@ApiResponse` en cada controller obligatorios en code review.

---

## Consecuencias

Habilitado por esta decisión:
- El frontend puede consumir la API con cualquier librería HTTP estándar (`fetch`, `axios`) sin dependencias adicionales.
- Los endpoints de solo lectura pueden cachearse en el API Gateway o en el cliente con cabeceras HTTP estándar.
- OpenAPI generado automáticamente elimina la necesidad de documentación manual de contratos.
- La comunicación interna entre módulos no incurre en latencia de red ni serialización.

Prevenido por esta decisión:
- Si en el futuro el frontend necesita queries altamente personalizadas (p. ej., un dashboard con datos de múltiples recursos en una sola petición), REST puede resultar en múltiples roundtrips. Mitigación: considerar endpoints de composición específicos para el dashboard (`GET /api/dashboard/doctor/:id`) antes de adoptar GraphQL.

Riesgos:
| Riesgo | Mitigación |
|--------|------------|
| El contrato REST evoluciona y el frontend queda fuera de sincronía    | El contrato OpenAPI se versiona en el repositorio y los cambios breaking requieren aprobación explícita del equipo de frontend en el PR.
| Over-fetching en endpoints que devuelven entidades grandes            | Parámetro `?fields=id,name,specialty` en endpoints de listado para proyección parcial cuando sea necesario.

---

## ADRs Relacionados

- **ADR-004** — La autenticación JWT se aplica en todos los endpoints REST mediante el `AuthGuard` exportado desde `identity/`.
- **ADR-002** — Los flujos muestran la mezcla: llamadas REST síncronas para operaciones directas del usuario, eventos asíncronos para efectos secundarios.
