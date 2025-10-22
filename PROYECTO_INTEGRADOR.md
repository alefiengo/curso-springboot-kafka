# Proyecto Integrador · Spring Boot & Apache Kafka

Referencia pública del proyecto integrador que se muestra como hilo conductor del curso. El código completo se publicará en un repositorio separado para que los estudiantes lo consulten cuando necesiten ejemplos o configuraciones avanzadas.

---

## Objetivo

Mostrar un escenario de comercio electrónico que evoluciona desde un servicio monolítico hasta una arquitectura de microservicios conectados mediante Kafka, reforzando los temas de cada bloque.

---

## Roadmap de versiones

| Versión | Bloque / Clase | Enfoque principal |
|---------|----------------|-------------------|
| v1.0 | Bloque 1 (Clases 1-3) | REST + JPA + PostgreSQL, validaciones y manejo de errores. |
| v2.0 | Bloque 2 (Clases 4-6) | Separación en microservicios, mensajería con Kafka, consumidores y streams. |
| v3.0 | Bloque 3 (Clases 7-8) | Seguridad básica (JWT), observabilidad y demo final. |

Cada versión incorpora los componentes trabajados en clase (información detallada se entregará en el repositorio dedicado).

---

## Componentes previstos

- **Servicios**: order-service, inventory-service, analytics-service (más servicios de apoyo si el tiempo lo permite).
- **Kafka**: topics para órdenes creadas/confirmadas, actualizaciones de inventario y flujo de analítica.
- **Persistencia**: PostgreSQL por servicio para favorecer el encapsulamiento.
- **Seguridad**: autenticación con JWT en endpoints críticos (bloque 3).
- **Observabilidad**: Spring Boot Actuator, métricas y logging estructurado.

---

## Recursos

- Repositorio de referencia: se comunicará al finalizar cada bloque.
- Colecciones Postman y docker-compose específicos acompañarán a cada versión publicada.
- La rúbrica y requisitos del proyecto final se anunciarán junto con el material de la Clase 8.

---

Este documento resume el avance del proyecto integrador visible para los estudiantes; detalles operativos y planificación interna se mantienen en archivos privados.
