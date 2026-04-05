# ADR-009: Despliegue y CI/CD — Docker + GitHub Actions + Railway

| Campo | Valor |
|---|---|
| Estado | Accepted |
| Fecha | 2026-04-05 |
| Responsables | Isabel y Gabriela |

---

## Contexto

El sistema necesita una estrategia de despliegue que permita al equipo entregar cambios a producción de forma confiable, rápida y con capacidad de revertir si algo falla. El entorno de producción debe ser reproducible y consistente entre los ambientes de desarrollo, staging y producción.

El sistema tiene dos procesos desplegables: `apps/api` (el monolito HTTP) y `apps/worker` (el proceso de jobs de fondo). Ambos comparten el mismo repositorio y código base, pero tienen ciclos de vida independientes: es posible reiniciar el worker sin interrumpir el API, y viceversa.

El equipo es pequeño (4–8 personas) con experiencia en desarrollo pero sin un DevOps dedicado. La estrategia de CI/CD debe ser lo suficientemente simple para que cualquier desarrollador del equipo pueda operar y depurar el pipeline, sin requerir conocimiento especializado de Kubernetes u orquestación compleja.

---

## Opciones Consideradas

### Opción 1: Docker + GitHub Actions + Railway

Contenedores Docker para cada proceso desplegable, pipeline de CI/CD en GitHub Actions, y Railway como plataforma de despliegue gestionada (similar a Heroku pero con soporte nativo de Docker).

| Pros | Contras |
|---|---|
| Railway gestiona el orquestador, el balanceador y los certificados SSL automáticamente | Railway tiene límites en el plan gratuito; el plan de producción cuesta ~$20/mes por servicio |
| GitHub Actions está integrado en el repositorio sin herramientas adicionales | Railway no es tan flexible como Kubernetes para configuraciones avanzadas de red |
| Rollback con un clic en el dashboard de Railway o con `railway rollback` | Dependencia de Railway como plataforma (vendor lock-in moderado) |
| Variables de entorno gestionadas de forma segura en Railway | — |
| Despliegue automático en cada merge a `main` con zero-downtime por defecto | — |

### Opción 2: Docker + GitHub Actions + Kubernetes (EKS/GKE)

Contenedores Docker orquestados con Kubernetes en un proveedor cloud gestionado.

| Pros | Contras |
|---|---|
| Control total sobre el entorno de ejecución, red y escalado | Curva de aprendizaje alta: deployments, services, ingress, namespaces, RBAC de K8s |
| Escala horizontal automática con Horizontal Pod Autoscaler | Un equipo sin DevOps dedicado pasaría semanas configurando el clúster antes de desplegar código |
| Estándar de la industria para sistemas de gran escala | Costo mínimo de un clúster EKS: ~$150/mes solo en nodos de control |
| — | Overkill para un monolito con carga moderada y predecible |

### Opción 3: VPS con despliegue manual (SSH + scripts)

Un servidor virtual privado (DigitalOcean Droplet, AWS EC2) donde el equipo sube el código manualmente vía SSH y ejecuta scripts de despliegue.

| Pros | Contras |
|---|---|
| Costo bajo: desde $6/mes por un Droplet básico | Sin automatización: cada despliegue es manual y propenso a errores humanos |
| Control total sobre el servidor | Sin rollback automático: revertir requiere acceso SSH y ejecución manual de comandos |
| Sin dependencias de plataformas externas | Sin ambientes de staging reproducibles |
| — | Los secretos de producción deben gestionarse manualmente en el servidor |

---

## Decisión

Se adopta **Docker + GitHub Actions + Railway** como estrategia de despliegue y CI/CD.

Razones específicas al proyecto:

1. **Sin DevOps dedicado:** Railway abstrae la complejidad de orquestación (balanceador, SSL, variables de entorno, logs) sin requerir conocimiento de Kubernetes. El equipo puede enfocarse en funcionalidad de negocio desde el primer sprint.

2. **Dos servicios independientes del mismo repo:** Railway soporta el despliegue de múltiples servicios desde un monorepo, cada uno con su propio Dockerfile. `apps/api` y `apps/worker` se despliegan y escalan de forma independiente sin configuración compleja.

3. **Rollback confiable:** Cada despliegue exitoso en Railway crea un snapshot inmutable. Si un deploy rompe algo en producción, el rollback al estado anterior es inmediato desde el dashboard o la CLI, sin necesidad de revertir commits en Git.

4. **GitHub Actions integrado:** El pipeline CI/CD vive en el mismo repositorio como código (`.github/workflows/`), versionado junto con el código de la aplicación. No requiere configurar un servidor de CI externo (Jenkins, TeamCity).

---

## Pipeline de CI/CD

```
Rama feature → PR → main
       │
       ▼
┌─────────────────────────────────────────┐
│          GitHub Actions CI              │
│                                         │
│  1. Lint (ESLint + reglas de módulos)   │
│  2. Type check (tsc --noEmit)           │
│  3. Tests unitarios (Jest)              │
│  4. Tests de integración (DB en Docker) │
│  5. Validación de formato de logs       │
│  6. Build de imagen Docker              │
│  7. Push a Container Registry           │
└─────────────────────────────────────────┘
       │ (solo si todos los pasos pasan)
       ▼
┌─────────────────────────────────────────┐
│         Railway CD — Staging            │
│  Deploy automático en ambiente staging  │
│  Smoke tests automáticos post-deploy    │
└─────────────────────────────────────────┘
       │ (aprobación manual para producción)
       ▼
┌─────────────────────────────────────────┐
│       Railway CD — Production           │
│  Deploy con zero-downtime (rolling)     │
│  Health check automático post-deploy    │
│  Rollback automático si health falla    │
└─────────────────────────────────────────┘
```

### Estrategia de Branches

| Branch | Ambiente | Despliegue |
|---|---|---|
| `feature/*` | Local | Manual con Docker Compose |
| `main` | Staging | Automático en cada merge |
| `main` (tag `v*`) | Producción | Manual con aprobación |

---

## Estructura de Dockerfiles

```dockerfile
# apps/api/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/apps/api/main.js"]
```

El `apps/worker/Dockerfile` sigue la misma estructura pero ejecuta `dist/apps/worker/main.js`.

---

## Consecuencias

Habilitado por esta decisión:
- Cualquier merge a `main` desencadena el pipeline completo automáticamente: lint, tests, build y deploy a staging.
- El equipo puede verificar cambios en staging antes de aprobar el despliegue a producción.
- El rollback a cualquier versión anterior tarda menos de 2 minutos desde el dashboard de Railway.
- Los secretos de producción (clave JWT, credenciales de DB, API key de Stripe) nunca están en el repositorio; se gestionan como variables de entorno en Railway.

Prevenido por esta decisión:
- No es posible escalar el servicio de API a más de las instancias que Railway permite en el plan contratado sin cambiar de plan o migrar a Kubernetes. Para el volumen esperado, esto no es una limitación real.

Riesgos:

| Riesgo | Mitigación |
|---|---|
| Railway sufre una interrupción y el sistema no se puede desplegar | El Dockerfile garantiza que el sistema puede desplegarse en cualquier VPS con Docker si Railway no está disponible |
| Un merge a `main` con tests insuficientes rompe staging | Los tests de integración corren en un contenedor de PostgreSQL limpio en cada ejecución del pipeline, evitando dependencias de estado externo |
| Las migraciones de base de datos fallan en producción | Las migraciones se ejecutan como un paso separado del pipeline antes del deploy, con validación de idempotencia |

---

## ADRs Relacionados

- **ADR-001** — El monolito modular simplifica el pipeline: dos Dockerfiles en lugar de 8+ pipelines independientes por microservicio.
- **ADR-003** — Las migraciones de base de datos por esquema se ejecutan en el pipeline antes del despliegue de la aplicación.
- **ADR-007** — El pipeline incluye validación de formato de logs como paso obligatorio antes del merge a main.
