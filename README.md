# Spring Boot & Apache Kafka: Desarrollo de Microservicios y Mensajería en Tiempo Real

Repositorio oficial del curso de **i-Quattro** enfocado en el desarrollo de aplicaciones empresariales con Spring Boot y la integración con Apache Kafka para el manejo de mensajería, streaming de datos y arquitecturas de microservicios escalables.

**[Información del curso](https://www.i-quattro.com/)**

---

## Información General

**Duración:** 20 horas
**Modalidad:** 100% práctico con laboratorios en cada sesión
**Horario:** Del 20 de octubre al 5 de noviembre, 2025 | Lunes, Miércoles y Viernes 19:00-21:30

### Objetivos

Formar a los participantes en el desarrollo de aplicaciones empresariales con Spring Boot y la integración con Apache Kafka para el manejo de mensajería, streaming de datos y arquitecturas de microservicios escalables.

### Valor Diferencial

- Curso 100% práctico con laboratorios en cada sesión
- Ejemplos orientados a casos reales (banca, retail, telecomunicaciones)
- Instructores con experiencia en proyectos cloud-native y DevOps
- Integración completa Spring Boot + Kafka con Docker

---

## Dirigido a

- **Desarrolladores Java** que buscan profundizar en microservicios
- **Ingenieros de software** que desean integrar Kafka para sistemas de mensajería
- **Arquitectos de software** interesados en soluciones event-driven
- **Estudiantes avanzados** de informática/ingeniería en desarrollo backend

---

## Contenido del Curso

### Temario (10 puntos)

1. Introducción a microservicios y Spring Boot
2. Creación de proyectos Spring Boot y configuración básica
3. REST APIs con Spring Web y manejo de datos con Spring Data JPA
4. Introducción a Apache Kafka y fundamentos de mensajería
5. Productores y Consumidores en Kafka con Spring Boot
6. Procesamiento de eventos y streaming con Kafka Streams
7. Seguridad en microservicios y Kafka
8. Patrones de diseño event-driven (CQRS, Event Sourcing)
9. Monitoreo, logging y métricas en aplicaciones Spring + Kafka
10. Proyecto final integrador: microservicio con Kafka para procesamiento en tiempo real

---

## Clases (8 sesiones de 2.5 horas)

### Bloque 1: Spring Boot Fundamentals (Clases 1-3)

- **[Clase 1: Introducción a Microservicios y Spring Boot](bloque-springboot/clase1-introduccion/)**
  - Arquitectura de microservicios vs monolitos
  - Configuración del entorno de desarrollo (Java, Maven, IDE)
  - Primer proyecto Spring Boot con Spring Initializr
  - REST API básica y explorando Spring Boot Actuator

- **[Clase 2: REST APIs con Spring Web y Spring Data JPA](bloque-springboot/clase2-rest-jpa/)**
  - Creación de REST APIs completas (CRUD)
  - Integración con PostgreSQL usando Spring Data JPA
  - Manejo de relaciones entre entidades
  - Pruebas con Postman y validación de datos

- **[Clase 3: Configuración Avanzada y Manejo de Datos](bloque-springboot/clase3-configuracion/)**
  - Profiles de Spring Boot (dev, prod)
  - Manejo de excepciones con @ControllerAdvice
  - Validaciones con Bean Validation
  - Paginación y ordenamiento con Spring Data

### Bloque 2: Apache Kafka (Clases 4-6)

- **[Clase 4: Introducción a Apache Kafka y Fundamentos de Mensajería](bloque-kafka/clase4-kafka-intro/)**
  - Arquitectura de Apache Kafka (brokers, topics, partitions)
  - Instalación de Kafka con Docker Compose
  - Kafka CLI: crear topics, producir y consumir mensajes
  - Conceptos: offset, consumer groups, replication

- **[Clase 5: Productores y Consumidores en Kafka con Spring Boot](bloque-kafka/clase5-producers-consumers/)**
  - Configuración de spring-kafka en Spring Boot
  - Implementación de Kafka Producers
  - Implementación de Kafka Consumers
  - Serialización y deserialización (JSON, Avro)
  - Manejo de errores y reintentos

- **[Clase 6: Kafka Streams y Procesamiento de Eventos](bloque-kafka/clase6-streams/)**
  - Introducción a Kafka Streams
  - Procesamiento stateless y stateful
  - Joins y agregaciones
  - Casos de uso: analytics en tiempo real

### Bloque 3: Advanced Topics & Integration (Clases 7-8)

- **[Clase 7: Seguridad y Patrones Event-Driven](bloque-avanzado/clase7-seguridad-patrones/)**
  - Seguridad en Spring Boot (Spring Security básico)
  - Autenticación y autorización en APIs
  - Patrón CQRS (Command Query Responsibility Segregation)
  - Patrón Event Sourcing
  - Idempotencia en mensajería

- **[Clase 8: Monitoreo, Logging, Métricas y Proyecto Final](bloque-avanzado/clase8-observabilidad/)**
  - Spring Boot Actuator para health checks y métricas
  - Integración con Prometheus y Grafana
  - Logging estructurado con Logback
  - Trazabilidad distribuida (conceptos de tracing)
  - Proyecto final integrador: sistema completo Spring Boot + Kafka

**[Ver bloques completos →](bloque-springboot/)**

---

## Prerrequisitos

### Conocimientos previos

- **Java básico/intermedio** (sintaxis, POO, colecciones)
- **Bases de datos relacionales** (SQL básico)
- **REST APIs** (conceptos básicos de HTTP)
- **Git básico** (clone, commit, push)

### Instalaciones necesarias

**Bloque 1 (Spring Boot):**
- **Java 17+** (OpenJDK o Oracle JDK) - [Guía de instalación](INSTALL_JAVA_MAVEN.md)
- **Maven 3.9+** - [Guía de instalación](INSTALL_JAVA_MAVEN.md)
- **IDE**: [IntelliJ IDEA](https://www.jetbrains.com/idea/download/) (recomendado), [Eclipse](https://www.eclipse.org/downloads/), o [VS Code](https://code.visualstudio.com/)
- **Docker Desktop** (Windows/macOS) o **Docker Engine** (Linux) - [Guía de instalación](INSTALL_DOCKER.md)
- [Git](https://git-scm.com/downloads)
- [Postman](https://www.postman.com/downloads/) o [Insomnia](https://insomnia.rest/download)

**Bloque 2 (Kafka):**
- **Apache Kafka** (vía Docker Compose, no requiere instalación local)
- **Kafka UI** (provectuslabs/kafka-ui - incluido en docker-compose)

### Instalaciones opcionales

- **IntelliJ IDEA Ultimate** (versión de pago con mejor soporte Spring Boot)
- **Spring Tools Suite (STS)** - IDE basado en Eclipse
- **DBeaver** o **pgAdmin** - Cliente de base de datos
- **HTTPie** - Cliente HTTP en terminal
- **kcat (kafkacat)** - Herramienta CLI para Kafka

### Verificación del entorno

```bash
# Java
java -version       # Debe mostrar 17 o superior
javac -version

# Maven
mvn -version        # Debe mostrar 3.9+

# Docker
docker --version
docker compose version

# Git
git --version
```

---

## Cómo usar este repositorio

Cada clase contiene:
- **README.md** con objetivos y referencias a laboratorios
- **labs/** con ejercicios prácticos paso a paso
- **demos/** con ejercicios conceptuales y exploratorios
- **tareas/** con desafíos y tarea para casa
- **cheatsheet.md** con comandos y conceptos de referencia rápida

Recomendamos clonar el repositorio y seguir las clases en orden:

```bash
git clone https://github.com/alefiengo/curso-springboot-kafka.git
cd curso-springboot-kafka/bloque-springboot/clase1-introduccion
```

---

## Proyecto Integrador

El curso incluye un **proyecto integrador** que evoluciona progresivamente clase a clase, simulando un sistema real de procesamiento de eventos para el sector bancario, retail o telecomunicaciones.

**Repositorio:** [proyecto-integrador-springboot-kafka](https://github.com/alefiengo/proyecto-integrador-springboot-kafka)

### Evolución del Proyecto

| Versión | Clase | Funcionalidades |
|---------|-------|-----------------|
| v1.0 | 1 | API REST básica con Spring Boot (in-memory) |
| v1.1 | 2 | + CRUD completo + Spring Data JPA + PostgreSQL |
| v1.2 | 3 | + Validación + Manejo de excepciones + Profiles |
| v2.0 | 4 | + Kafka setup (Docker) + Topics básicos |
| v2.1 | 5 | + Producers/Consumers + Serialización JSON |
| v2.2 | 6 | + Kafka Streams + Procesamiento en tiempo real |
| v3.0 | 7 | + Spring Security + Patrón CQRS |
| v3.1 | 8 | + Actuator + Prometheus + Grafana (Observabilidad completa) |

### Stack Completo (Clase 8)

- **Backend**: Spring Boot 3.x (Java 17)
- **Database**: PostgreSQL 15
- **Messaging**: Apache Kafka 3.x
- **Monitoring**: Actuator + Prometheus + Grafana
- **Containerization**: Docker Compose

**[Ver más detalles →](PROYECTO_INTEGRADOR.md)**

---

## Estructura de un Laboratorio

Cada laboratorio sigue una estructura de **8 secciones** para facilitar el aprendizaje:

1. **Objetivo** - Qué aprenderás en este lab
2. **Comandos a ejecutar** - Pasos con comandos y salidas esperadas
3. **Desglose del comando** - Explicación de cada flag/parámetro
4. **Explicación detallada** - Qué sucede internamente
5. **Conceptos aprendidos** - Resumen de conceptos clave
6. **Troubleshooting** - Errores comunes y soluciones
7. **Desafío adicional** - Ejercicio opcional para practicar
8. **Recursos adicionales** - Links a documentación oficial

---

## Tareas y Evaluación

Cada clase incluye:

### 1. Desafío Rápido (en clase)
- Práctica rápida de 5-10 minutos
- Refuerza conceptos aprendidos en el laboratorio
- Se realiza al final de la sesión

### 2. Tarea para Casa
- Proyecto individual a entregar antes de la siguiente clase
- Debe documentarse en repositorio personal (GitHub/GitLab)
- Incluye criterios de evaluación y rúbrica
- Se entrega vía Moodle con enlace al repositorio

**[Guía de entrega de tareas →](ENTREGA_TAREAS.md)**

---

## Casos de Uso por Industria

El curso enfatiza ejemplos del mundo real:

### Banca (Banking)
- Procesamiento de transacciones en tiempo real
- Detección de fraude con event streams
- Sincronización de cuentas entre sistemas
- Notificaciones de eventos bancarios

### Retail (Comercio)
- Gestión de pedidos (order management)
- Actualizaciones de inventario en tiempo real
- Notificaciones a clientes (shipping, delivery)
- Analytics de ventas en tiempo real

### Telecomunicaciones
- Procesamiento de eventos de facturación (billing)
- Seguimiento de uso de datos/llamadas
- Monitoreo de red en tiempo real
- Alertas y notificaciones a usuarios

---

## Recursos Adicionales

### Documentación Oficial
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring Kafka](https://docs.spring.io/spring-kafka/reference/html/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Spring Security](https://docs.spring.io/spring-security/reference/index.html)

### Herramientas
- [Spring Initializr](https://start.spring.io/) - Generador de proyectos Spring Boot
- [Kafka UI](https://github.com/provectus/kafka-ui) - Interfaz web para Kafka
- [Postman](https://www.postman.com/) - Cliente API
- [DBeaver](https://dbeaver.io/) - Cliente de base de datos universal

### Tutoriales
- [Baeldung Spring Boot](https://www.baeldung.com/spring-boot)
- [Confluent Kafka Tutorials](https://developer.confluent.io/tutorials/)
- [Spring Guides](https://spring.io/guides)

---

## Notas

- Este repositorio es **público** y forma parte del curso oficial de i-Quattro
- El material está diseñado para estudiantes matriculados, pero es de libre acceso para la comunidad
- Si te resulta útil, considera dejar una estrella en GitHub

---

## Uso del Material

Este repositorio es **público** y de libre acceso para la comunidad. Siéntete libre de usarlo para aprender, practicar y compartir conocimiento.

---

**[i-Quattro](https://www.i-quattro.com/)** | Formación en Desarrollo Backend y Arquitecturas Cloud-Native
