# Clase 3: Arquitectura en Capas y Validación

Esta sesión refactoriza el `product-service` construido en la clase anterior para adoptar una arquitectura en capas profesional, incorporar DTOs, validaciones con Bean Validation y un manejo uniforme de errores. También sentará las bases para relaciones 1:N que se profundizarán antes de pasar al bloque de Kafka.

---

## Objetivos de aprendizaje

- Refactorizar el servicio para separar responsabilidades entre Controller, Service, Repository y Entity.
- Implementar DTOs y mapeos manuales para evitar filtrar entidades directamente en las respuestas REST.
- Aplicar validaciones con JSR-380 (`@Valid`, `@NotBlank`, `@Size`, etc.) y mensajes personalizados.
- Centralizar el manejo de excepciones con `@ControllerAdvice` y `ErrorResponse` consistente.
- Preparar una relación `Product` ↔ `Category` (1:N) respetando buenas prácticas de JPA.

---

## Requisitos previos

- Clase 2 completada con el `product-service` funcional y persistiendo en PostgreSQL.
- Docker Compose activo para PostgreSQL (`docker compose up -d`).
- Herramientas instaladas: Postman/Bruno, cliente SQL (DBeaver, TablePlus, pgAdmin) y Git.
- Repositorio personal sincronizado con los cambios de Clase 2.

---

## Hoja de ruta sugerida

Antes de iniciar las prácticas revisa los **[Conceptos de la clase](CONCEPTOS.md)** para repasar arquitectura en capas, DI/IoC, Bean Validation y manejo de excepciones.

1. **Presentación teórica**  
   Arquitectura por capas, DTO pattern, DI/IoC y nociones de Bean Validation.
2. **[Lab 00 · Refactor a arquitectura en capas](labs/00-refactor-capas-servicio/)**  
   Organizar controller, service y mapper manual con DTOs.
3. **[Lab 01 · Validaciones con Bean Validation](labs/01-validaciones-bean-validation/)**  
   Añadir anotaciones de validación y mensajes personalizados.
4. **[Lab 02 · Manejo global de excepciones](labs/02-manejo-excepciones/)**  
   Crear `@RestControllerAdvice` y unificar respuestas de error.
5. **[Lab 03 · Relación Product–Category (1:N)](labs/03-relacion-product-category/)**  
   Añadir entidad `Category`, relación 1:N y endpoints asociados.
---

## Laboratorios de la clase

- [labs/00-refactor-capas-servicio/](labs/00-refactor-capas-servicio/README.md)
- [labs/01-validaciones-bean-validation/](labs/01-validaciones-bean-validation/README.md)
- [labs/02-manejo-excepciones/](labs/02-manejo-excepciones/README.md)
- [labs/03-relacion-product-category/](labs/03-relacion-product-category/README.md)

---

## Tarea para casa

- **[Refuerzo de arquitectura, validaciones y relación 1:N](tarea/README.md)**  
  Consolida el refactor, añade validaciones obligatorias, documenta el manejo de errores y entrega la colección Postman actualizada.

---

## Recursos de apoyo

- **Conceptos de la clase:** [CONCEPTOS.md](CONCEPTOS.md)
- **Cheatsheet:** [cheatsheet.md](cheatsheet.md)
- **FAQ:** [FAQ.md](FAQ.md)
- **Colección Postman:** [recursos/postman/clase3-product-service.postman_collection.json](recursos/postman/clase3-product-service.postman_collection.json)
- **Documentación oficial:**
  - [Spring MVC Controllers](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-controller)
  - [Bean Validation 3.0 (JSR 380)](https://jakarta.ee/specifications/bean-validation/3.0/)
  - [Spring Exception Handling](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-controller-advice)

---

## Próximos pasos

- Completar los labs y reflejar los cambios en el repositorio personal.
- Preparar dudas sobre validaciones y manejo de errores para la sesión de Q&A.
- Mantener actualizado el README de tu proyecto con instrucciones para ejecutar el servicio y probar errores.

La Clase 4 introducirá Apache Kafka y arquitectura event-driven. Más adelante (Clase 8) veremos perfiles y configuración externa, por lo que es importante dejar el servicio organizado y validado antes de continuar.
