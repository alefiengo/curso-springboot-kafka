# Clase 2: REST APIs y Persistencia con Spring Data JPA

Clase dedicada a pasar del proyecto vacío creado en la sesión anterior a un microservicio funcional con endpoints REST, persistencia en PostgreSQL y pruebas con herramientas de cliente HTTP.

---

## Objetivos de aprendizaje

- Diferenciar las responsabilidades de Spring Framework y Spring Boot dentro de una aplicación moderna.
- Construir endpoints REST con distintas firmas (`@PathVariable`, `@RequestParam`, `@RequestBody`) y métodos HTTP.
- Preparar un entorno de base de datos con PostgreSQL mediante Docker Compose.
- Modelar entidades JPA y repositorios para persistir información.
- Implementar un CRUD completo sobre productos y verificarlo con Postman o herramientas equivalentes.

---

## Requisitos previos

- Clase 1 completada y entorno Java 17 + Maven configurado.
- Proyecto inicial `product-service` generado (Spring Web) o disposición para crearlo en el Lab 00.
- Docker Desktop (Windows/macOS) o Docker Engine (Linux) en funcionamiento.
- Cliente de base de datos (DBeaver, TablePlus, pgAdmin o similar).
- Cliente HTTP como Postman, Bruno o Thunder Client.

---

## Hoja de ruta sugerida

Antes de iniciar los laboratorios revisa los **[Conceptos de la clase](CONCEPTOS.md)** para repasar anotaciones, IoC/DI y flujo de peticiones en Spring Boot.

1. **Presentación teórica breve**  
   Spring Framework vs Spring Boot, comparación con otros frameworks modernos y anatomía de una aplicación Spring Boot.
2. **Lab 00 · Generar el proyecto base**  
   Crear `product-service`, revisar la estructura Maven y ejecutar la aplicación.
3. **Lab 01 · Hola Mundo REST**  
   Primer controlador REST con dos endpoints GET.
4. **Lab 02 · Parámetros y métodos HTTP**  
   Endpoints con parámetros de ruta, query string y cuerpo JSON.
5. **Lab 03 · PostgreSQL con Docker Compose**  
   Levantar la base de datos sin instalación local y comprobar conexión.
6. **Lab 04 · Entidad y repositorio JPA**  
   Añadir dependencias, crear entidad `Product` y repositorio Spring Data.
7. **Lab 05 · CRUD completo**  
   Construir endpoints `GET/POST/PUT/DELETE`, probarlos y validar persistencia.

Cada laboratorio sigue la plantilla oficial de ocho secciones para asegurar consistencia pedagógica.

---

## Laboratorios de la clase

- **[Lab 00 · Generar el proyecto product-service](labs/00-crear-proyecto/)**  
  Obtén el proyecto base con Spring Initializr y verifica su ejecución.
- **[Lab 01 · Hola Mundo REST](labs/01-hola-mundo-rest/)**  
  Crea el primer controlador REST y entiende el flujo de una petición.
- **[Lab 02 · Parámetros y métodos HTTP](labs/02-parametros-http/)**  
  Practica diferentes formas de recibir datos en tus endpoints.
- **[Lab 03 · PostgreSQL con Docker Compose](labs/03-postgresql-docker/)**  
  Configura el contenedor de base de datos, variables de entorno y cliente SQL.
- **[Lab 04 · Entidad y repositorio JPA](labs/04-entity-jpa/)**  
  Introduce Spring Data JPA, mapea la entidad `Product` y prueba el repositorio.
- **[Lab 05 · CRUD REST completo](labs/05-crud-completo/)**  
  Expone operaciones completas sobre productos y valida la persistencia end to end.

---

## Tarea para casa

- **[CRUD con relación Producto-Categoría](tarea/README.md)**  
  Amplía el servicio para gestionar categorías y vincular productos (relación 1:N). Incluye criterios de evaluación, checklist y formato de entrega.

---

## Recursos de apoyo

- **Guía rápida:** [cheatsheet.md](cheatsheet.md)  
  Comandos clave de Maven, Docker Compose, Postman y consultas SQL básicas.
- **FAQ:** [FAQ.md](FAQ.md)  
  Errores frecuentes en dependencias, configuración de base de datos y conexión Docker.
- **Colección Postman:** [recursos/postman/product-service.postman_collection.json](recursos/postman/product-service.postman_collection.json)

---

## Próximos pasos

Antes de la siguiente clase asegúrate de:

- Completar los cinco laboratorios con código ejecutable.
- Documentar en tu repositorio personal cómo ejecutar el servicio y la base de datos.
- Resolver la tarea de relación producto-categoría.
- Registrar dudas o bloqueos para el espacio de Q&A inicial de la Clase 3.

La Clase 3 se enfoca en arquitectura en capas, validaciones y manejo de excepciones para profesionalizar el servicio construido aquí.
