# Componente: API Gateway

## 1. Responsabilidades

- **Punto de entrada único:** Recibe todas las solicitudes HTTP/HTTPS entrantes desde clientes web y móviles, y las enruta al proceso `apps/api` del monolito modular.
- **Rate limiting:** Aplica límites de peticiones por IP (para tráfico anónimo) y por `userId` extraído del JWT (para tráfico autenticado), protegiéndose de abuso y scraping de datos médicos.
- **Terminación TLS:** Gestiona los certificados SSL y descifra el tráfico HTTPS antes de pasarlo al monolito en HTTP interno, evitando que el proceso de aplicación cargue con la negociación TLS.
- **Validación de cabeceras:** Verifica la presencia del header `Authorization: Bearer <jwt>` en rutas protegidas antes de que la solicitud llegue al módulo `identity/`, rechazando tempranamente peticiones malformadas.
- **Enrutamiento de webhooks:** Dirige las solicitudes POST de Stripe al endpoint `/api/payments/webhook` de forma aislada, sin pasar por el middleware de autenticación estándar.

## 2. Por Qué Este Proyecto lo Necesita

Un sistema de salud expone datos clínicos y financieros sensibles. Sin un API Gateway, el proceso Node.js estaría directamente expuesto a internet, lo que significa que cada ataque de denegación de servicio, intento de scraping de perfiles médicos o solicitud malformada consumiría recursos del proceso de aplicación.

El flujo de webhooks de Stripe introduce una complejidad adicional: es el único punto de entrada que no usa JWT sino firma HMAC. Sin un gateway que enrute este tráfico de forma diferenciada, la lógica de autenticación del monolito tendría que manejar dos esquemas completamente distintos en el mismo middleware, acoplando la infraestructura con la lógica de dominio.

## 3. Elección de Tecnología

**Elección: Nginx (self-hosted en el VPS)**

| Dimensión | Nginx *(Elección)* | AWS API Gateway *(Alternativa)* | Kong *(Alternativa)*|
|---|---|---|---|
| **Managed / Self-hosted** | Self-hosted | Managed (AWS) | Self-hosted o Cloud |
| **Complejidad operativa** | Baja: un archivo de configuración declarativo | Media: configuración por consola AWS o IaC | Alta: base de datos propia, plugins, admin API |
| **Costo a nuestra escala** | $0 adicional (corre en el mismo VPS) | ~$3.50/millón de peticiones y data transfer | Gratuito (OSS) pero requiere infraestructura adicional |
| **Feature diferenciador** | Rate limiting nativo, reverse proxy maduro, mínimo overhead (~1ms) | Integración nativa con IAM/Lambda/Cognito | Ecosistema de plugins extenso, ideal para microservicios |

Nginx es la elección correcta porque el sistema ya corre en un VPS y Nginx puede coexistir en el mismo servidor sin costo adicional. AWS API Gateway introduce vendor lock-in innecesario y su modelo de precios por petición es desproporcionado para el volumen esperado. Kong añade complejidad operativa (requiere su propia base de datos) que no se justifica para un monolito con dos procesos.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Costo cero adicional: corre en el mismo VPS que el monolito                   | El equipo es responsable de mantener la configuración y actualizaciones de seguridad |
| Latencia mínima al monolito: comunicación en localhost                        | Sin dashboard de monitoreo nativo — las métricas de gateway se exportan vía módulo `nginx-prometheus-exporter` |
| Configuración declarativa y versionable en el repositorio                     | Rate limiting basado en memoria del proceso |
| Maduro y battle-tested: maneja decenas de miles de conexiones concurrentes     | Requiere recarga manual (`nginx -s reload`) para aplicar cambios de configuración |

## 5. Integración

En el diagrama de despliegue (Diagrama 4), Nginx se ubica en la API Layer dentro de la VPC privada, entre Cloudflare (CDN) y el proceso `apps/api`. El tráfico fluye así:

```
Cloudflare -> Nginx (puerto 443/80) -> apps/api (puerto 3000, HTTP interno)
Stripe webhooks -> Nginx -> /api/payments/webhook (sin AuthGuard)
```

Nginx conoce la dirección interna del contenedor `apps/api` vía variable de entorno o DNS interno de Docker. Si se agregan réplicas del proceso `apps/api`, Nginx actúa también como load balancer con `upstream` round-robin sin configuración adicional, conectándose directamente con el componente de Load Balancer descrito en su propio archivo.
