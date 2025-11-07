# Clase 5: Productores y Consumidores con Spring Kafka

Esta clase integra Apache Kafka con Spring Boot usando spring-kafka, implementando productores de eventos en product-service y creando order-service como segundo microservicio con capacidades de publicación de eventos.

---

## Objetivos de aprendizaje

- Integrar spring-kafka en un proyecto Spring Boot existente
- Configurar productores de Kafka con KafkaTemplate
- Crear eventos de dominio (ProductCreated, OrderPlaced)
- Publicar mensajes a topics de Kafka desde microservicios
- Crear un segundo microservicio (order-service) con Spring Boot
- Comprender la serialización JSON en Kafka
- Implementar logging de eventos publicados

---

## Requisitos previos

- Clase 4 completada (Kafka funcionando con Docker Compose)
- product-service con arquitectura en capas (de Clase 3)
- Docker Compose con Kafka y Zookeeper en ejecución
- Maven y Java 17 configurados
- Postman o curl para probar endpoints

---

## Hoja de ruta sugerida

Antes de iniciar revisa los **[Conceptos de la clase](CONCEPTOS.md)**.

1. [Lab 00: Configurar spring-kafka en product-service](labs/00-spring-kafka-config/)
2. [Lab 01: Producer en product-service](labs/01-producer-product-service/)
3. [Lab 02: Crear order-service](labs/02-crear-order-service/)
4. [Lab 03: Producer en order-service](labs/03-producer-order-service/)

---

## Laboratorios de la clase

- [labs/00-spring-kafka-config/](labs/00-spring-kafka-config/README.md) - Dependencias y configuración spring-kafka
- [labs/01-producer-product-service/](labs/01-producer-product-service/README.md) - Eventos ProductCreated
- [labs/02-crear-order-service/](labs/02-crear-order-service/README.md) - Segundo microservicio básico
- [labs/03-producer-order-service/](labs/03-producer-order-service/README.md) - Eventos OrderPlaced

---

## Tarea

[Tarea para Casa](tarea/README.md)

---

## Recursos

- [Cheatsheet - Clase 5](cheatsheet.md)
- [FAQ - Clase 5](FAQ.md)
- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/html/)
- [KafkaTemplate Reference](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/core/KafkaTemplate.html)
- [Spring Kafka Getting Started](https://spring.io/guides/gs/messaging-kafka/)
