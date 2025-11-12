# Clase 8: Producción, Seguridad y Consolidación

Esta clase final prepara los microservicios para entornos de producción implementando perfiles, monitoreo con Actuator, variables de entorno, seguridad con JWT, y logging estructurado. Culmina con una revisión completa del proyecto y preparación para el proyecto integrador.

---

## Objetivos de aprendizaje

- Configurar perfiles de Spring Boot para diferentes entornos (dev, prod)
- Implementar health checks y métricas con Spring Boot Actuator
- Externalizar configuración usando variables de entorno
- Proteger endpoints REST con JWT (JSON Web Tokens)
- Implementar logging estructurado para debugging en producción
- Consolidar el conocimiento de 4 microservicios con arquitectura event-driven
- Preparar propuesta para el proyecto integrador final

---

## Contexto de la clase

### Estado previo

Después de completar las Clases 1-7, tienes:

- 4 microservicios operativos comunicándose vía Kafka
- 5 topics activos con flujo de eventos completo
- 3 bases de datos PostgreSQL independientes
- Arquitectura event-driven con CQRS implementada

### Arquitectura completa

```
product-service (8080)
  ├── PostgreSQL: ecommerce
  ├── Publica: ecommerce.products.created
  └── JWT Security

order-service (8081)
  ├── PostgreSQL: ecommerce_orders
  ├── Publica: ecommerce.orders.placed
  └── Consume: ecommerce.inventory.updated

inventory-service (8082)
  ├── PostgreSQL: ecommerce_inventory
  ├── Consume: ecommerce.orders.placed
  └── Publica: ecommerce.inventory.updated

analytics-service (8083)
  ├── Kafka Streams (sin BD propia)
  ├── Consume: todos los topics
  └── CQRS read model
```

### Nota importante

Los laboratorios 00-02 de esta clase (Profiles, Actuator, Variables de Entorno) fueron originalmente planificados para Clase 4, pero se movieron aquí para enfocarnos en la infraestructura Kafka primero y en la configuración de producción al final del curso, justo antes del proyecto integrador.

---

## Requisitos previos

- **Clases 1-7 completadas**: 4 microservicios funcionando con Kafka
- **Docker Compose ejecutándose**: Kafka y PostgreSQL activos
- **Java 17 y Maven** configurados
- **Postman o curl** para probar endpoints
- **Comprensión de**: REST, JPA, Kafka, Streams

---

## Hoja de ruta sugerida

Antes de iniciar revisa los **[Conceptos de la clase](CONCEPTOS.md)**.

1. [Lab 00: Spring Boot Profiles](labs/00-spring-profiles/)
2. [Lab 01: Spring Boot Actuator](labs/01-actuator/)
3. [Lab 02: Variables de Entorno](labs/02-variables-entorno/)
4. [Lab 03: JWT Security](labs/03-jwt-security/)
5. [Lab 04: Logging Estructurado](labs/04-logging/)
6. [Lab 05: Consolidación Final](labs/05-consolidacion/)

---

## Laboratorios de la clase

- [labs/00-spring-profiles/](labs/00-spring-profiles/README.md) - Configurar perfiles dev/prod
- [labs/01-actuator/](labs/01-actuator/README.md) - Health checks y métricas
- [labs/02-variables-entorno/](labs/02-variables-entorno/README.md) - Externalizar configuración
- [labs/03-jwt-security/](labs/03-jwt-security/README.md) - Proteger con JWT
- [labs/04-logging/](labs/04-logging/README.md) - Logs estructurados
- [labs/05-consolidacion/](labs/05-consolidacion/README.md) - Revisión final

---

## Contenido de producción

### 12-Factor App

Esta clase implementa varios principios de [12-Factor App](https://12factor.net/):

- **III. Config**: Externalizar configuración en variables de entorno
- **VI. Processes**: Servicios stateless (Kafka Streams es la excepción)
- **XI. Logs**: Logging estructurado a stdout

### Spring Boot Profiles

Aprenderás a:

- Crear perfiles `dev` y `prod`
- Configurar logging diferente por perfil
- Activar perfiles desde línea de comandos
- Usar perfiles para diferentes entornos

### Spring Boot Actuator

Implementarás:

- Health checks para Kubernetes/Docker
- Métricas de la aplicación
- Endpoints de información
- Monitoreo de Kafka y PostgreSQL

### JWT Security

Protegerás product-service con:

- Autenticación basada en tokens
- Login endpoint sin base de datos (demo)
- Filtro de autenticación
- Configuración de seguridad

---

## Tarea

[Tarea para Casa - Preparar Proyecto Integrador](tarea/README.md)

---

## Recursos

- [Cheatsheet - Clase 8](cheatsheet.md)
- [FAQ - Clase 8](FAQ.md)
- [Spring Boot Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [12-Factor App](https://12factor.net/)
- [JWT.io](https://jwt.io/)
- [PROYECTO_INTEGRADOR.md](../../PROYECTO_INTEGRADOR.md)

---

## Próximos pasos

Después de completar esta clase:

- Tendrás 4 microservicios listos para producción
- Conocerás prácticas de configuración y monitoreo
- Implementarás seguridad básica con JWT
- Estarás preparado para el proyecto integrador final

El proyecto integrador te permitirá aplicar todos los conocimientos del curso en un dominio propio, diseñando e implementando tu propia arquitectura de microservicios con Apache Kafka.
