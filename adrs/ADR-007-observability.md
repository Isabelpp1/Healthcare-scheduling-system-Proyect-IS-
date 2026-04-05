# ADR-007: Estrategia de Observabilidad — Loki + Prometheus + Grafana + OpenTelemetry

| Campo | Valor |
|---|---|
| Estado | Accepted |
| Fecha | 2026-04-05 |
| Responsables | Isabel y Gabriela  |

---

## Contexto

El sistema combina flujos síncronos y asíncronos entre módulos. Cuando algo falla —una cita no se confirma, un pago queda inconsistente, o un recordatorio nunca llega— el equipo necesita identificar exactamente dónde ocurrió el problema y cuánto tardó cada paso.

Un flujo típico atraviesa varios módulos: el paciente solicita una cita en `scheduling/`, que publica `CitaSolicitada` al bus, lo consume `payments/`, llama a Stripe, y el resultado regresa como `PagoConfirmado` a `scheduling/` y `notifications/`. Sin observabilidad adecuada, diagnosticar un fallo en este flujo requiere revisar logs manualmente módulo por módulo sin una línea de tiempo clara.

El sistema también maneja datos de salud sensibles, por lo que los logs y trazas nunca deben exponer información clínica. El equipo es pequeño (4–8 personas), así que la solución debe ser operativamente viable sin un ingeniero de plataforma dedicado.

---

## Opciones Consideradas

### Opción 1: Stack Open Source — Loki + Prometheus + Grafana + OpenTelemetry

Tres herramientas complementarias que cubren los tres pilares de observabilidad: logs (Loki), métricas (Prometheus) y trazas (OpenTelemetry + Grafana Tempo). Se despliegan como contenedores en la misma VPC.

| Pros | Contras |
|---|---|
| Costo de licencias: $0 (open source) | Requiere configuración inicial y mantenimiento de los contenedores del stack |
| Los tres pilares integrados en un único dashboard de Grafana | El equipo debe definir y mantener alertas y dashboards manualmente |
| OpenTelemetry es estándar abierto sin vendor lock-in | El almacenamiento de métricas y logs escala con el volumen del sistema |
| Loki indexa solo metadatos de logs (eficiente en costo de almacenamiento) | — |
| Correlación directa entre logs, métricas y trazas en Grafana | — |

### Opción 2: Plataforma SaaS — Datadog

Plataforma de observabilidad completamente gestionada que integra logs, métricas, trazas y alertas en un único servicio externo.

| Pros | Contras |
|---|---|
| Sin infraestructura de observabilidad que operar | Costo elevado: desde $15/host/mes más costos adicionales por logs y trazas ingestados |
| Dashboards y alertas predefinidas para Node.js y PostgreSQL | Todos los logs y trazas del sistema salen de la VPC hacia servidores de Datadog |
| Detección de anomalías con ML sin configuración adicional | Para datos de salud, enviar trazas con payloads a un tercero requiere análisis de cumplimiento regulatorio |
| — | Vendor lock-in: migrar requiere reconfigurar toda la instrumentación |

### Opción 3: Solo Logs en Stdout

Únicamente logs estructurados en JSON enviados a stdout, recolectados por el orquestador de contenedores. Sin métricas ni trazas dedicadas.

| Pros | Contras |
|---|---|
| Configuración mínima, funciona desde el primer día | Sin visibilidad de latencia, throughput ni tasa de errores por endpoint |
| Sin herramientas adicionales que operar | Imposible correlacionar una solicitud a través de múltiples módulos |
| — | Diagnosticar fallos en flujos asíncronos requiere búsqueda manual entre logs sin correlación |

---

## Decisión

Se adopta el **stack open source Loki + Prometheus + Grafana + OpenTelemetry** como estrategia de observabilidad.

Razones específicas al proyecto:

1. **Los flujos asíncronos requieren trazabilidad end-to-end:** Un log de error en `notifications/` no basta para entender por qué una notificación no llegó. Se necesita trazar el evento desde `scheduling/` hasta `notifications/` con tiempos de cada paso. OpenTelemetry con trace IDs propagados a través del bus de eventos hace esto posible sin cambiar los módulos de dominio.

2. **Costo proporcional al volumen:** Con ~8,000 eventos/día y un equipo de 4–8 personas, Datadog costaría entre $150–300/mes solo en observabilidad, antes de los costos de ingestión de logs y trazas. Loki + Prometheus en contenedores tiene costo marginal cero en licencias.

3. **Datos de salud dentro de la VPC:** Aunque los logs se configuran para no incluir datos clínicos, las trazas pueden contener IDs de citas en los payloads de los spans. Mantener el stack de observabilidad dentro de la VPC propia elimina el riesgo de exposición a un proveedor externo.

4. **Un único panel de control:** Grafana integra los tres pilares: desde una métrica de latencia alta se puede hacer drill-down a la traza que causó el spike y al log de error correspondiente, reduciendo el tiempo de diagnóstico.

---

## SLOs Monitoreados

| Métrica | Objetivo | Alerta si supera |
|---|---|---|
| Latencia P95 de `POST /api/scheduling/appointments` | < 500 ms | > 800 ms durante 5 min |
| Latencia P95 de `POST /api/payments/orders/:id/pay` | < 2,000 ms | > 3,000 ms durante 5 min |
| Tasa de errores HTTP 5xx | < 0.5% | > 1% durante 5 min |
| Mensajes acumulados en Dead-Letter Queue | 0 | > 5 mensajes |
| Disponibilidad del proceso API | > 99.5% | Proceso caído > 1 min |
| Duración del job de recordatorios (Worker) | < 2 min | > 5 min |

---

## Política de Privacidad en Logs

Los logs siguen formato JSON estructurado con campos fijos: `timestamp`, `level`, `traceId`, `module`, `event`, `durationMs`. Los siguientes datos **nunca** aparecen en logs o trazas:

- Contenido de notas clínicas, diagnósticos o prescripciones
- Datos personales del paciente (nombre, fecha de nacimiento, condiciones médicas)
- Tokens de pago o últimos cuatro dígitos de tarjeta

---

## Consecuencias

Habilitado por esta decisión:
- Es posible correlacionar una solicitud HTTP con todos los eventos asíncronos que generó, usando el `traceId` propagado a través del bus de eventos.
- Las alertas en Grafana notifican al equipo cuando los SLOs se degradan antes de que los usuarios reporten problemas.
- Los mensajes en la DLQ de RabbitMQ generan alertas inmediatas, evitando que eventos críticos queden sin procesar silenciosamente.

Prevenido por esta decisión:
- Los datos clínicos nunca aparecen en logs ni trazas. Esto limita el nivel de detalle para depuración, pero es un trade-off necesario por privacidad y cumplimiento regulatorio.

Riesgos:

| Riesgo | Mitigación |
|---|---|
| El stack de observabilidad consume recursos significativos | Loki con retención de 15 días, Prometheus con 30 días. Almacenamiento estimado < 10 GB/mes para el volumen esperado |
| Los dashboards quedan obsoletos y dejan de reflejar el sistema real | Dashboards definidos como código JSON en el repositorio y revisados en cada sprint de infraestructura |
| Un desarrollador agrega un dato clínico en un log por error | Linter en el pipeline CI/CD que detecte campos prohibidos en los objetos de log antes del merge |

---

## ADRs Relacionados

- **ADR-002** — La comunicación asíncrona requiere propagación de trace IDs entre eventos para mantener la correlación de flujos completos.
- **ADR-006** — El tamaño de las colas de RabbitMQ y las DLQ son métricas clave monitoreadas en Prometheus con alertas configuradas.
- **ADR-009** — El pipeline de CI/CD incluye validación del formato de logs antes del despliegue a producción.
