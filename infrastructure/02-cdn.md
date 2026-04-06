# Componente: CDN (Content Delivery Network)

## 1. Responsabilidades

- **Distribución de assets estáticos:** Sirve los archivos JavaScript, CSS, imágenes y fuentes de la aplicación web desde nodos edge geográficamente cercanos al usuario, reduciendo la latencia de carga inicial de la interfaz.
- **Protección DDoS en la capa de borde:** Absorbe y filtra tráfico malicioso (ataques volumétricos, floods HTTP) antes de que llegue a la VPC privada, protegiendo al API Gateway y al monolito de ser saturados.
- **Terminación SSL en el edge:** Gestiona los certificados TLS y cifra/descifra el tráfico HTTPS en el nodo más cercano al usuario, reduciendo la latencia de la negociación TLS y descargando ese trabajo del servidor de aplicación.
- **Caché de respuestas HTTP públicas:** Cachea respuestas GET de endpoints públicos sin autenticación (como el catálogo de especialidades médicas o perfiles públicos de médicos) con base en los headers `Cache-Control` que el API devuelve.
- **Ocultamiento de infraestructura:** El usuario nunca conoce la IP real del servidor de producción. Todo el tráfico pasa por los nodos de Cloudflare, lo que impide ataques directos al VPS si la IP queda expuesta.

## 2. Por Qué Este Proyecto lo Necesita

Una plataforma de citas médicas tiene un patrón de tráfico predecible pero con picos pronunciados: los lunes por la mañana, cuando los pacientes buscan citas para la semana, el endpoint de consulta de slots y perfiles de médicos recibe ráfagas de solicitudes en pocos minutos. Sin CDN, cada una de esas solicitudes llega directamente al VPS en Guatemala o la región de despliegue, introduciendo latencia innecesaria para usuarios en otras ciudades o países.

Adicionalmente, el sistema maneja datos de salud sensibles. Exponer la IP real del servidor facilita ataques de denegación de servicio dirigidos que podrían interrumpir el acceso de pacientes a sus citas o de médicos a sus agendas. Cloudflare como capa de borde es la primera línea de defensa sin costo adicional de infraestructura.

## 3. Elección de Tecnología

**Elección: Cloudflare (plan gratuito / Pro)**

| Dimensión | Cloudflare *(Elección)* | AWS CloudFront *(Alternativa A)* | Fastly *(Alternativa B)* |
|---|---|---|---|
| **Managed / Self-hosted** | Managed (Cloudflare) | Managed (AWS) | Managed (Fastly) |
| **Complejidad operativa** | Muy baja: cambio de nameservers DNS, configuración por dashboard | Media: distribuciones CloudFront, políticas de caché en JSON/YAML, integración con ACM para certificados | Media: configuración de servicios en VCL o UI |
| **Costo a nuestra escala** | $0 en plan gratuito (ancho de banda ilimitado); $20/mes plan Pro con WAF | $0.0085/GB de transferencia + $0.0075/10k peticiones HTTP | Desde $50/mes en plan de pago; sin plan gratuito funcional |
| **Feature diferenciador** | Protección DDoS ilimitada en todos los planes, certificados SSL automáticos, DNS integrado, WAF básico gratuito | Integración nativa con S3, Lambda@Edge para lógica en el borde, ideal si todo el stack es AWS | Muy baja latencia de caché, control granular de VCL para lógica avanzada en el edge |

Cloudflare es la elección correcta para este proyecto porque el plan gratuito incluye protección DDoS, SSL automático y caché básica sin costo adicional. AWS CloudFront tendría sentido si el stack completo estuviera en AWS, pero introduciría vendor lock-in y costos variables por transferencia de datos que son difíciles de predecir. Fastly no tiene plan gratuito viable para un proyecto en etapa inicial.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Protección DDoS ilimitada incluida en el plan gratuito | Cloudflare actúa como intermediario del tráfico, lo que implica que Cloudflare ve las IPs de los usuarios |
| Certificados SSL automáticos con renovación sin intervención del equipo | El modo "proxied" de Cloudflare oculta la IP real del servidor pero también limita el uso de ciertos protocolos no-HTTP |
| Caché de assets estáticos reduce carga en el VPS sin configuración compleja | La invalidación de caché en Cloudflare requiere una llamada a su API o acción manual en el dashboard |
| Ancho de banda de salida hacia usuarios ilimitado en todos los planes | En el plan gratuito, el WAF avanzado y las reglas de firewall personalizadas no están disponibles |
| DNS autoritativo incluido con propagación global rápida | Dependencia de Cloudflare: si Cloudflare tiene una interrupción, el sistema queda inaccesible aunque el VPS esté funcionando |

## 5. Integración

En el diagrama de despliegue (Diagrama 4), Cloudflare se posiciona como la primera capa antes de que cualquier tráfico llegue a la VPC privada. El flujo completo es:

```
Usuario (navegador/app móvil)
    │
    ▼ HTTPS
Cloudflare (CDN / DDoS / SSL termination)
    │
    ├── Assets estáticos (.js, .css, imágenes) → servidos desde cache de Cloudflare
    │
    └── Solicitudes de API (/api/*) → proxy hacia Nginx (API Gateway) en el VPS
                                           │
                                           ▼
                                      apps/api (monolito)
```

La configuración de Cloudflare establece reglas de Page Rules o Cache Rules para:
- Cachear rutas como `/api/scheduling/specialties` con `Cache-Control: public, max-age=86400`
- Pasar sin caché todas las rutas autenticadas (header `Authorization` presente)
- Enrutar webhooks de Stripe directamente sin modificar headers

La integración con el componente de Observabilidad (Diagrama 4) se realiza exportando los logs de acceso de Cloudflare hacia Loki mediante el conector de Cloudflare Logpush, disponible en el plan Pro.
