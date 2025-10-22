# Proyecto Integrador · Spring Boot & Apache Kafka

El proyecto integrador se desarrolla durante las últimas sesiones del curso y consolida los conceptos aprendidos en los bloques de Spring Boot y Kafka.

---

## Objetivo general

Construir un sistema de microservicios orientado a comercio electrónico que procese pedidos, administre inventario y genere analítica en tiempo real usando Apache Kafka.

---

## Alcance tentativo

- **Servicios**:
  - `order-service`: expone APIs REST para gestionar órdenes y publica eventos en Kafka.
  - `inventory-service`: consume eventos de órdenes y actualiza existencias.
  - `analytics-service`: procesa flujos con Kafka Streams y genera métricas agregadas.
- **Kafka**:
  - Topics para órdenes creadas, órdenes confirmadas y actualizaciones de inventario.
  - Consumer groups diferenciados por servicio.
- **Persistencia**:
  - PostgreSQL por servicio (database-per-service) o enfoque en memoria si el tiempo es limitado.
- **Seguridad (Clase 7/8)**:
  - Autenticación básica con JWT para endpoints sensibles.
  - Configuración mínima de CORS y manejo de tokens.
- **Observabilidad**:
  - Uso de Spring Boot Actuator y logs estructurados.

> El detalle final se ajustará según el ritmo del grupo. Esta guía sirve como referencia para planificar entregables y rúbricas.

---

## Entregables sugeridos

1. **Repositorio Git central** con los tres microservicios y una carpeta `infra/` para `docker-compose.yml`.  
2. **Documentación**:
   - README principal con instrucciones de ejecución.
   - Diagramas de arquitectura (opcional) en `docs/`.
   - Colecciones Postman para endpoints REST.
3. **Docker Compose**:
   - Brokers de Kafka + Zookeeper.
   - Kafka UI o herramienta similar para monitoreo.
   - Servicios Spring Boot listos para levantarse en conjunto.
4. **Presentación final** (Clase 8):
   - Resumen de arquitectura.
   - Flujo de eventos y demo breve de endpoints + mensajes.

---

## Evaluación preliminar (propuesta)

| Criterio | Descripción | Peso |
|----------|-------------|------|
| Diseño e implementación de microservicios | Calidad del código, separación de responsabilidades, buenas prácticas | 35% |
| Integración con Kafka | Productores, consumidores, manejo de eventos y resiliencia básica | 30% |
| Observabilidad y seguridad mínima | Uso de Actuator, logs, manejo de tokens o credenciales | 15% |
| Documentación y despliegue | README, scripts de arranque, claridad en la explicación | 20% |

---

## Próximos pasos

- Completar la documentación de Clases 3-7 antes de concretar el repositorio definitivo del proyecto.  
- Definir si el proyecto será entregado individual o en grupos y comunicarlo en la Clase 7.  
- Preparar una rúbrica final alineada con esta tabla y publicarla en la carpeta de la Clase 8.

