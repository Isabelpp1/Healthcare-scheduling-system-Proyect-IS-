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
