# Propuestas de Arquitectura

Esta carpeta contiene los cuatro documentos que describen el sistema desde el nivel más alto hasta los límites concretos de módulos y flujos de datos. En conjunto, permiten que un ingeniero nuevo pueda entender el sistema completo sin necesidad de revisar código.

---

| Documento | Descripción |
|---|---|
| [01-high-level-architecture.md](01-high-level-architecture.md) | Comparación de Monolito Modular vs Microservicios con tabla de dimensiones y recomendación justificada para este proyecto |
| [02-bounded-contexts.md](02-bounded-contexts.md) | Descomposición DDD en 8 bounded contexts con entidades tipadas, eventos de dominio y mapa de contextos con patrones de integración |
| [03-service-module-decomposition.md](03-service-module-decomposition.md) | Árbol de directorios del monolito, descripción de cada módulo, qué expone y cómo se refuerzan los límites de contexto |
| [04-data-flow-and-interactions.md](04-data-flow-and-interactions.md) | 4 flujos end-to-end con diagramas de secuencia Mermaid, happy path y caminos de fallo y compensación |
