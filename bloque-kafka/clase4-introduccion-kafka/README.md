# Clase 4: Introducción a Apache Kafka

Esta clase introduce Apache Kafka como plataforma de mensajería event-driven, cubriendo su arquitectura, infraestructura con Docker y comandos CLI fundamentales.

---

## Objetivos de aprendizaje

- Comprender la arquitectura de Apache Kafka (brokers, topics, partitions, consumer groups)
- Entender el modelo de mensajería publish-subscribe y event-driven
- Desplegar Kafka en modo KRaft con Docker Compose
- Crear y gestionar topics usando Kafka CLI
- Producir y consumir mensajes desde la línea de comandos
- Explorar casos de uso reales de Kafka en diferentes industrias

---

## Requisitos previos

- Clase 3 completada (product-service con arquitectura en capas)
- Docker y Docker Compose instalados y funcionando
- Maven y Java 17 configurados
- Postman o curl para probar endpoints

---

## Hoja de ruta sugerida

Antes de iniciar revisa los **[Conceptos de la clase](CONCEPTOS.md)**.

1. [Lab 00: Conceptos de Kafka](labs/00-conceptos-kafka/)
2. [Lab 01: Docker Compose para Kafka](labs/01-docker-compose-kafka/)
3. [Lab 02: Kafka CLI - Crear Topics](labs/02-kafka-cli-topics/)

---

## Laboratorios de la clase

- [labs/00-conceptos-kafka/](labs/00-conceptos-kafka/README.md) - Arquitectura event-driven
- [labs/01-docker-compose-kafka/](labs/01-docker-compose-kafka/README.md) - Infraestructura Kafka
- [labs/02-kafka-cli-topics/](labs/02-kafka-cli-topics/README.md) - Gestión de topics

**Nota**: Los laboratorios de perfiles, Actuator y variables de entorno se verán en la Clase 8 como parte de la preparación para producción.

---

## Tarea

[Tarea para Casa](tarea/README.md)

---

## Recursos

- [Cheatsheet - Clase 4](cheatsheet.md)
- [FAQ - Clase 4](FAQ.md)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kafka Docker Quick Start](https://developer.confluent.io/quickstart/kafka-docker/)
- [Kafka Intro (Confluent)](https://developer.confluent.io/what-is-apache-kafka/)