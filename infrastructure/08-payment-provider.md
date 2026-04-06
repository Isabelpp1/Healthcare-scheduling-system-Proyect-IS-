# Componente: Proveedor de Pago (Stripe)

## 1. Responsabilidades

- **Procesamiento de cobros:** El módulo `payments/` envía solicitudes de cargo vía Stripe API (`POST /v1/charges` o `POST /v1/payment_intents`) por el monto de la consulta al momento en que el paciente confirma el pago.
- **Tokenización de métodos de pago:** Los datos de tarjeta del paciente nunca llegan al servidor — Stripe.js en el frontend los tokeniza directamente con Stripe, y el servidor solo recibe el `paymentMethodToken` para realizar el cargo. Esto elimina la obligación de cumplir PCI DSS nivel 1.
- **Procesamiento de reembolsos:** Al cancelar una cita con derecho a reembolso (> 24 horas de anticipación), el módulo `payments/` invoca `POST /v1/refunds` con el `chargeId` original y el monto a devolver.
- **Webhooks de confirmación asíncrona:** Stripe notifica el resultado final de operaciones asíncronas (especialmente pagos 3D Secure y disputas) via `POST /api/payments/webhook`. El endpoint valida la firma `Stripe-Signature` antes de procesar el evento.
- **Gestión de disputas y chargebacks:** El módulo `payments/` registra los eventos de disputa recibidos via webhook y marca las órdenes afectadas para revisión manual del equipo de operaciones.

## 2. Por Qué Este Proyecto lo Necesita

El procesamiento de pagos con tarjeta de crédito requiere cumplimiento PCI DSS, certificaciones de seguridad y relaciones con redes de pago que ningún equipo de desarrollo puede implementar de forma razonable. Stripe abstrae toda esta complejidad: el sistema solo necesita integrar una API REST bien documentada y delegar la responsabilidad de seguridad de los datos de tarjeta al proveedor.

El flujo de pago del sistema tiene una característica particular: la confirmación de la cita depende del resultado del pago (patrón Partnership entre `scheduling/` y `payments/`). Stripe soporta esta coordinación via webhooks: cuando Stripe confirma el cobro, el webhook activa el evento `PagoConfirmado` que a su vez confirma la cita. El `StripeGateway` en `infrastructure/gateways/` actúa como la ACL que traduce el modelo de Stripe al modelo interno.

## 3. Elección de Tecnología

**Elección: Stripe**

| Dimensión | Stripe *(Elección)* | PayPal / Braintree *(Alternativa)* | Wompi (regional) *(Alternativa)* |
|---|---|---|---|
| **Managed / Self-hosted** | Managed (SaaS) | Managed (SaaS) | Managed (SaaS) |
| **Complejidad operativa** | Baja: SDK oficial para Node.js, webhooks, dashboard completo | Media: dos productos distintos (PayPal + Braintree), integración más verbosa | Baja: API simple, enfocada en mercado latinoamericano |
| **Costo a nuestra escala** | 2.9% + $0.30 por transacción exitosa | 3.49% + $0.49 (tarjeta de crédito estándar) | Variable según país; ~2.95% en Colombia y Guatemala |
| **Feature diferenciador** | Mejor DX de la industria, SDK tipado, testing con tarjetas de prueba, webhooks robustos, soporte para 135+ monedas | Reconocimiento de marca para pacientes, Braintree como pasarela flexible | Cobertura de métodos de pago locales relevantes en LATAM |

Stripe es la elección por su developer experience superior, documentación exhaustiva y SDK oficial para Node.js con tipado TypeScript completo. Para el mercado guatemalteco (la plataforma opera en GTQ), Wompi podría ser considerado si los métodos de pago locales son prioritarios en una segunda versión. Si el sistema expande a múltiples países latinoamericanos, una ACL que soporte múltiples gateways detrás de `PaymentGatewayPort` permitiría agregar Wompi o MercadoPago sin cambiar el dominio.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| El sistema nunca toca datos de tarjeta: cumplimiento PCI DSS simplificado (SAQ A) | Dependencia crítica: si Stripe tiene downtime, los pagos del sistema no funcionan |
| SDK Node.js con TypeScript completo, integración sin boilerplate manual           | Costo por transacción (2.9% + $0.30) es significativo para consultas de bajo monto |
| Webhooks confiables con reintentos automáticos y firma HMAC para validación       | Los webhooks pueden llegar fuera de orden o duplicados en donde el handler debe ser idempotente |
| Entorno de prueba (`sk_test_*`) idéntico al de producción para desarrollo y CI    | El dashboard de Stripe es el único lugar para gestionar disputas, no integrado en el sistema |
| Soporte para 3D Secure y autenticación adicional de forma transparente            |  |

## 5. Integración

En el diagrama de despliegue, Stripe es un Servicio Externo Gestionado fuera de la VPC. Las conexiones son bidireccionales:

```
apps/api (payments/StripeGateway) -> Stripe API HTTPS   (POST /charges, POST /refunds)
Stripe                            -> apps/api HTTPS      (POST /api/payments/webhook)
```

El `StripeGateway` en `modules/payments/infrastructure/gateways/stripe.gateway.ts` implementa el puerto `PaymentGatewayPort` definido en el dominio. Esta ACL traduce las respuestas de Stripe al modelo interno (`ChargeResult`, `RefundResult`), aislando al dominio de `payments/` de los cambios en la API de Stripe. El endpoint de webhook tiene su propio controller (`stripe-webhook.controller.ts`) con middleware de validación de firma (`Stripe-Signature`) que corre antes que el `AuthGuard` global, ya que Stripe no envía JWTs.
