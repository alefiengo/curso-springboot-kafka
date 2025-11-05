# Clase 4: Perfiles, Configuración y Apache Kafka

Esta clase prepara `product-service` para producción con perfiles y observabilidad, e introduce Apache Kafka como plataforma de mensajería event-driven.

---

## Objetivos de aprendizaje

- Configurar perfiles de Spring Boot para diferentes entornos (dev, prod)
- Implementar observabilidad con Spring Boot Actuator
- Externalizar configuración con variables de entorno
- Comprender la arquitectura de Apache Kafka (brokers, topics, partitions, consumer groups)
- Desplegar Kafka y Zookeeper con Docker Compose
- Crear y gestionar topics usando Kafka CLI

---

## Requisitos previos

- Clase 3 completada (product-service con arquitectura en capas)
- Docker y Docker Compose instalados y funcionando
- Maven y Java 17 configurados
- Postman o curl para probar endpoints

---

## Hoja de ruta sugerida

Antes de iniciar revisa los **[Conceptos de la clase](CONCEPTOS.md)**.

### Parte 1: Production-Ready Spring Boot

1. [Lab 00: Perfiles de Spring Boot](labs/00-perfiles-spring-boot/)
2. [Lab 01: Spring Boot Actuator](labs/01-actuator/)
3. [Lab 02: Variables de Entorno](labs/02-variables-entorno/)

### Parte 2: Introducción a Apache Kafka

4. [Lab 03: Conceptos de Kafka](labs/03-conceptos-kafka/)
5. [Lab 04: Docker Compose para Kafka](labs/04-docker-compose-kafka/)
6. [Lab 05: Kafka CLI - Crear Topics](labs/05-kafka-cli-topics/)

---

## Laboratorios de la clase

- [labs/00-perfiles-spring-boot/](labs/00-perfiles-spring-boot/README.md) - Configuración por entorno
- [labs/01-actuator/](labs/01-actuator/README.md) - Health checks y métricas
- [labs/02-variables-entorno/](labs/02-variables-entorno/README.md) - Externalización de config
- [labs/03-conceptos-kafka/](labs/03-conceptos-kafka/README.md) - Arquitectura event-driven
- [labs/04-docker-compose-kafka/](labs/04-docker-compose-kafka/README.md) - Infraestructura Kafka
- [labs/05-kafka-cli-topics/](labs/05-kafka-cli-topics/README.md) - Gestión de topics

---

## Tarea

[Tarea para Casa](tarea/README.md)

---

## Recursos

- [Cheatsheet - Clase 4](cheatsheet.md)
- [FAQ - Clase 4](FAQ.md)
- [Spring Boot Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kafka Docker Quick Start](https://developer.confluent.io/quickstart/kafka-docker/)