# 01 - Arquitectura de Alto Nivel

## Sistema de Programación de Citas Médicas


## Introducción

El Sistema basado en Programación de Citas Médicas sera una plataforma digital que conecta a pacientes con profesionales de la salud, permitiendo a los pacientes poder agendar, gestionar y y poder dar seguimiento de sus citas médicas. El sistema involucra múltiples roles entre ellos tenemos: paciente, médico, administrador de clínica, recepcionista, flujos de pago, notificaciones en tiempo real e integraciones con sistemas externos como calendarios, pasarelas de pago y servicios de mensajería.

A continuación se comparan dos enfoques arquitectónicos aplicables a este proyecto.



## Enfoque A: Monolito Modular

### Descripción

En este enfoque, todo el sistema llegaria a caer en una sola base de código desplegada como un solo artefacto. Sin embargo, internamente está organizado en módulos bien definidos con límites previamente especificados: `pacientes`, `médicos`, `citas`, `pagos`, `notificaciones`, `autenticación`, entre otros.

Los módulos se estarian comunicando por medio de interfaces internas explícitas y compartirian una base de datos principal con esquemas separados por contexto. Las reglas de negocio de cada dominio quedan encapsuladas dentro de su módulo correspondiente.

**Aplicado a este proyecto:** Un equipo pequeño de entre 4 a 8 desarrolladores, puede construir rápidamente la funcionalidad core lo cual seria: registro de usuarios, disponibilidad de médicos, agendamiento de citas y cobro. La separación modular nos podria garantizar que los límites del dominio se respeten sin la complejidad operativa de múltiples servicios. El despliegue en un servidor o en un PaaS llega a ser mas directo y económico.



## Enfoque B: Microservicios

### Descripción

Cada capacidad de negocio se llegaria a implementar como un servicio independiente con su propia base de datos, con sus procesos de despliegue y ciclos de vida. Los servicios se comunican por medio de APIs REST/gRPC y por medio de un bus de eventos como Kafka o RabbitMQ.

Los servicios que se estarian implementando son: `auth-service`, `patient-service`, `doctor-service`, `scheduling-service`, `payment-service`, `notification-service`, `analytics-service`, entre otros. Cada uno puede escalar, desplegarse y fallar de forma independiente.

**Aplicado a este proyecto:** Permite que el módulo de agendamiento el cual representa el mas critico y con mayor carga llegue a escalar horizontalmente de forma independiente durante horas de alta demanda. El equipo de pagos puede iterar sin afectar el módulo de notificaciones. Sin embargo, requiere infraestructura adicional: orquestador de contenedores, service mesh, trazabilidad distribuida y gestión de consistencia eventual.

---

## Tabla Comparativa

| Dimensión                                 | Monolito Modular | Microservicios |
|-------------------------------------------|---|---|
| **Complejidad de despliegue**             | Baja — un artefacto, un pipeline CI/CD | Alta — múltiples servicios, pipelines independientes, orquestación |
| **Tamaño de equipo ideal**                | 4–10 desarrolladores | 15+ desarrolladores (equipos por servicio) |
| **Escalabilidad horizontal**              | Limitada — escala todo el monolito | Alta — cada servicio escala de forma independiente |
| **Aislamiento de fallos**                 | Bajo — un fallo puede afectar todo el proceso | Alto — un servicio caído no derrumba el sistema completo |
| **Velocidad de desarrollo inicial**       | Alta — sin overhead de red, contratos, ni infra distribuida | Baja — requiere definir contratos, infraestructura y comunicación entre servicios desde el inicio |
| **Velocidad de desarrollo a largo plazo** | Media — riesgo de acoplamiento si no se mantiene disciplina modular | Alta — equipos autónomos, deploys independientes |
| **Costo de infraestructura**              | Bajo — uno o dos servidores, DB única | Alto — múltiples contenedores, brokers, balanceadores, bases de datos por servicio |
| **Complejidad operativa**                 | Baja — logs centralizados, un proceso a monitorear | Alta — trazabilidad distribuida, health checks por servicio, gestión de consistencia eventual |

---

## Recomendación: Monolito Modular

**Se recomienda el Monolito Modular** como arquitectura inicial para este sistema.

### Justificación

1. **Tamaño del equipo:** El proyecto podria ser desarrollado por un equipo académico/startup de tamaño reducido de almenos de 8 personas. Los microservicios exigen equipos dedicados por servicio; con un equipo pequeño, la sobrecarga de coordinación y operación distribuida supera los beneficios.

2. **Complejidad del dominio en etapa temprana:** Aunque el sistema tiene múltiples bounded contexts como: pacientes, médicos, citas, pagos, notificaciones, las fronteras exactas entre ellos no están completamente estabilizadas. El monolito modular permite refactorizar y ajustar esas fronteras sin el costo de modificar contratos de API entre servicios independientes.

3. **Carga esperada:** Una plataforma de citas médicas no enfrenta el volumen de transacciones de Amazon o Netflix. El pico de carga más predecible ocurre en las mañanas de días laborales. Un monolito bien desplegado impplementado con réplicas de lectura en la DB y caché que puede manejar decenas de miles de citas diarias sin problemas.

4. **Costo operativo:** En etapa inicial, el costo de operar Kubernetes, un message broker dedicado, bases de datos por servicio y trazabilidad distribuida sería desproporcionado respecto al valor entregado. El monolito modular puede correr en un PaaS asequible con una base de datos gestionada.

5. **Camino de evolución:** La disciplina modular interna como módulos con interfaces explícitas, sin importaciones cruzadas directas, eventos internos para comunicación asíncrona lo cual facilita la extracción futura de servicios independientes si el sistema crece y el equipo lo requiere. 

> **Conclusión:** El Monolito Modular ofrece el equilibrio correcto entre velocidad de entrega, mantenibilidad y costo operativo para las condiciones actuales del proyecto. La arquitectura interna respetará estrictamente los límites de los bounded contexts definidos en el Documento 2, lo que garantiza que una migración futura a microservicios sea incremental y controlada.