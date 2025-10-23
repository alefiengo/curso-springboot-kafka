# Spring Boot & Apache Kafka: Desarrollo de Microservicios y Mensajería en Tiempo Real

Repositorio oficial del curso de **i-Quattro** centrado en el desarrollo de microservicios con Spring Boot e integración con Apache Kafka para mensajería y procesamiento de eventos en tiempo real.

---

## Sobre el programa

- Serie de ocho sesiones presenciales, con enfoque práctico y laboratorios guiados.
- Progresión gradual: fundamentos de Spring Boot → persistencia → mensajería con Kafka → integración y seguridad.
- Material completamente en español, sin notas privadas del instructor y orientado a estudiantes que ya completaron la Clase 1.

---

## Público objetivo

- Desarrolladores Java que desean profesionalizar su trabajo con Spring Boot.
- Ingenieros de software que necesitan integrar una plataforma de mensajería basada en Kafka.
- Arquitectos interesados en soluciones event-driven y microservicios escalables.
- Estudiantes avanzados de ingeniería informática con interés en backend.

---

## Resultados de aprendizaje

Al finalizar el curso, el estudiante será capaz de:

- Diseñar e implementar microservicios con Spring Boot 3.x y Java 17.
- Exponer APIs REST robustas con control de validaciones, manejo de errores y persistencia en PostgreSQL.
- Integrar aplicaciones Spring con Apache Kafka para publicar y consumir eventos.
- Introducir Kafka Streams en escenarios de analítica y transformación en tiempo real.
- Incorporar prácticas básicas de seguridad y observabilidad en microservicios event-driven.

---

## Organización por bloques

### Bloque 1 · Fundamentos de Spring Boot

- **[Clase 1: Introducción a Microservicios y Spring Boot](bloque-springboot/clase1-introduccion/)**  
  Entorno de desarrollo listo, teoría de microservicios y primer proyecto Spring Boot.
- **[Clase 2: REST APIs y Persistencia con Spring Data JPA](bloque-springboot/clase2-rest-jpa/)**  
  CRUD básico desde el controlador consumiendo directamente `ProductRepository` y persistiendo en PostgreSQL.
- **[Clase 3: Arquitectura en capas y validación](bloque-springboot/clase3-arquitectura-capas/)**  
  Refactor a capas con DTOs y servicios, Bean Validation, manejo global de excepciones y relación Product–Category.

### Bloque 2 · Apache Kafka y mensajería

- **Clase 4: Kafka desde Spring**  
  Arquitectura de Kafka, despliegue con Docker Compose, tooling CLI y primeros topics/productores manuales.
- **Clase 5: Productores y consumidores con Spring Kafka**  
  Configuración de `spring-kafka`, serialización JSON, publicación y consumo confiable desde microservicios.
- **Clase 6: Integración entre microservicios y procesamiento**  
  Caso e-commerce con dos microservicios comunicados vía Kafka, consumo concurrente y patrones de resiliencia básicos.

### Bloque 3 · Streams, seguridad y cierre

- **Clase 7: Kafka Streams y analítica en tiempo real**  
  Topologías, operaciones stateless, agregaciones y tercer microservicio orientado a analytics.
- **Clase 8: Seguridad y proyecto integrador**  
  Introducción a JWT en Spring Security, hardening mínimo de productores/consumidores y entrega del proyecto final.

---

## Prerrequisitos

- Conocimientos de Java orientado a objetos, colecciones y buenas prácticas básicas.
- Familiaridad con HTTP y conceptos REST.
- Manejo elemental de Git y uso de línea de comandos.
- Herramientas instaladas: Java 17, Maven 3.9+, Docker, Git e IDE (IntelliJ IDEA recomendado).  
  Para soporte adicional revisa `bloque-springboot/clase1-introduccion/FAQ.md`, `cheatsheet.md`, [INSTALL_JAVA_MAVEN.md](INSTALL_JAVA_MAVEN.md) y [INSTALL_DOCKER.md](INSTALL_DOCKER.md).

---

## Estructura del repositorio

```
bloque-springboot/
├── clase1-introduccion/
│   ├── README.md
│   ├── CONCEPTOS.md
│   ├── cheatsheet.md
│   ├── FAQ.md
│   └── tarea/
│       └── README.md
├── clase2-rest-jpa/
│   ├── README.md
│   ├── CONCEPTOS.md
│   ├── cheatsheet.md
│   ├── FAQ.md
│   ├── labs/
│   ├── recursos/
│   └── tarea/
├── clase3-arquitectura-capas/
│   ├── README.md
│   ├── CONCEPTOS.md
│   ├── cheatsheet.md
│   ├── FAQ.md
│   ├── labs/
│   ├── recursos/
│   └── tarea/
└── ... (clases siguientes)
```

Cada clase incluye:
- `README.md` con objetivos, contexto y enlaces a laboratorios.
- `CONCEPTOS.md` con la teoría previa (cuando aplique).
- Carpeta `labs/` con prácticas guiadas siguiendo la plantilla de ocho secciones.
- Carpeta `tarea/` con la actividad para casa y criterios de evaluación.
- Recursos auxiliares cuando aplique (Postman collections, guías, diagramas).

---

## Cómo aprovechar el material

1. Sigue las clases en orden y completa los laboratorios antes de avanzar.
2. Usa los labs como guía paso a paso; cada uno detalla comandos, explicaciones y troubleshooting.
3. Documenta tus resultados en un repositorio propio para las tareas y el proyecto final.
4. Consulta los cheatsheets y FAQ de cada clase cuando surjan dudas.
5. Comparte experiencias y bloqueos en el foro oficial del curso.

---

## Recursos recomendados

- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/html/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Spring Guides](https://spring.io/guides)
- [Baeldung – Spring Boot](https://www.baeldung.com/spring-boot)
- [Confluent Kafka Tutorials](https://developer.confluent.io/tutorials/)

---

## Roadmap de clases en preparación

El material público se irá liberando progresivamente. Próximos contenidos:

1. **Clase 4:** Perfiles, configuración externa y preparación para despliegues.
2. **Clase 5:** Productores y consumidores Kafka con Spring Boot.
3. **Clase 6:** Integración entre microservicios y procesamiento de eventos.
4. **Clase 7:** Kafka Streams, analítica en tiempo real y seguridad básica.
5. **Clase 8:** Observabilidad, logging y proyecto integrador.

Cada clase se publicará con README, laboratorios, tarea y recursos adicionales en cuanto estén listos.

---

Este repositorio es público. Úsalo para aprender, adaptar ejemplos a tus proyectos y compartir conocimiento con la comunidad.

**i-Quattro** · Formación en desarrollo backend y arquitecturas cloud-native
