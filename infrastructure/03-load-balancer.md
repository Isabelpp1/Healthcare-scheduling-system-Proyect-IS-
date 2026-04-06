# Componente: Load Balancer

## 1. Responsabilidades

- **Distribución de tráfico entre réplicas del monolito:** Reparte las solicitudes HTTP entrantes entre las instancias activas del proceso `apps/api`, evitando que una sola instancia acumule toda la carga durante los picos de mañana.
- **Health checking de instancias:** Verifica periódicamente que cada réplica del monolito responde correctamente en su endpoint `/health`. Si una instancia falla el health check, el load balancer deja de enviarle tráfico automáticamente hasta que se recupere.
- **Conexiones persistentes hacia el upstream:** Mantiene un pool de conexiones HTTP keep-alive hacia cada instancia del monolito, evitando el costo de abrir y cerrar conexiones TCP en cada solicitud.
- **Routing diferenciado por ruta:** Dirige el tráfico de la ruta `/api/payments/webhook` exclusivamente a instancias que tienen habilitado el handler de webhooks de Stripe, aislando ese tráfico del resto de las solicitudes.
- **Terminación de conexiones lentas:** Cierra conexiones de clientes que no envían datos dentro del timeout configurado, liberando recursos del monolito de conexiones zombie.

## 2. Por Qué Este Proyecto lo Necesita

El sistema tiene un patrón de carga predecible con picos pronunciados en horarios de mañana los días laborables. Una única instancia del monolito puede manejar la carga media, pero durante los picos (cuando múltiples pacientes consultan slots y confirman citas simultáneamente) se necesita la capacidad de agregar réplicas horizontalmente sin tiempo de inactividad.

Sin un load balancer, escalar horizontalmente el monolito requeriría cambiar la configuración del DNS o del CDN para apuntar a la nueva instancia, lo que introduce tiempo de propagación y complejidad operativa. Con Nginx como load balancer, agregar una réplica es tan simple como actualizar el bloque `upstream` en la configuración y recargar Nginx, sin interrumpir el tráfico existente.

## 3. Elección de Tecnología

**Elección: Nginx (en el mismo VPS, como reverse proxy con upstream)**

| Dimensión | Nginx *(Elección)* | AWS ALB *(Alternativa A)* | HAProxy *(Alternativa B)* |
|---|---|---|---|
| **Managed / Self-hosted** | Self-hosted | Managed (AWS) | Self-hosted |
| **Complejidad operativa** | Muy baja: bloque `upstream` en el mismo archivo de configuración del API Gateway | Baja: configuración por consola o Terraform, pero requiere cuenta AWS y VPC configurada | Baja-Media: archivo de configuración propio (`haproxy.cfg`), herramienta dedicada exclusivamente al balanceo |
| **Costo a nuestra escala** | $0 adicional: corre en el mismo proceso Nginx del API Gateway | ~$0.008/hora por ALB + $0.008/LCU hora, aproximadamente $16/mes mínimo | $0 en versión Community Edition |
| **Feature diferenciador** | Doble rol: API Gateway y Load Balancer en el mismo proceso, sin salto de red adicional | Health checks avanzados, sticky sessions, integración nativa con Auto Scaling Groups de AWS | Estadísticas detalladas en tiempo real, algoritmos de balanceo más avanzados (least connections, source hashing) |

Nginx es la elección correcta porque ya cumple el rol de API Gateway en la arquitectura. Añadir balanceo de carga es una extensión natural de su configuración sin costo ni complejidad adicional. AWS ALB tiene sentido si el despliegue es completamente en AWS con Auto Scaling Groups, lo que no es el caso en Railway. HAProxy es una excelente herramienta de balanceo pero añadir un proceso dedicado solo para esta función es innecesario cuando Nginx ya lo cubre.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Costo cero: el mismo proceso Nginx actúa como gateway y load balancer | Nginx mismo se convierte en punto único de fallo si no se despliega en alta disponibilidad |
| Configuración declarativa y versionada en el repositorio junto al resto del código | El algoritmo de balanceo de Nginx (round-robin por defecto) no considera la carga actual de cada instancia |
| Recarga de configuración sin downtime con `nginx -s reload` | Las métricas de balanceo (solicitudes por instancia, tiempo de respuesta por upstream) requieren el módulo `nginx-module-vts` adicional |
| Health checks nativos con `proxy_next_upstream` para reintentar en instancia sana | Sticky sessions requieren módulo adicional o configuración de `ip_hash`, lo que puede desbalancear el tráfico |

## 5. Integración

En el diagrama de despliegue (Diagrama 4), Nginx actúa simultáneamente como API Gateway (capa de entrada) y Load Balancer (distribución hacia réplicas). La configuración Nginx relevante sigue este esquema:

```nginx
upstream api_servers {
    server api_1:3000;   # Réplica 1 de apps/api
    server api_2:3000;   # Réplica 2 de apps/api (en picos)
    keepalive 32;        # Pool de conexiones persistentes
}

server {
    listen 443 ssl;

    location /api/ {
        proxy_pass http://api_servers;
        proxy_next_upstream error timeout http_500;  # Reintenta en otra instancia si falla
    }

    location /health {
        proxy_pass http://api_servers;
        access_log off;  # No contaminar logs con health checks
    }
}
```

El health check hacia cada instancia verifica el endpoint `GET /health` que el monolito expone, el cual valida conectividad con PostgreSQL y Redis antes de responder `200 OK`. Si una instancia responde con error o no responde en 2 segundos, Nginx la marca como `down` y redirige el tráfico a las instancias restantes, conectándose directamente con el comportamiento de failover descrito en el diagrama de despliegue.
