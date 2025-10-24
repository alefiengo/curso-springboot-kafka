# Guion del Instructor · Clase 2

## Propósito de la sesión
- Pasar del starter vacío a una API REST funcional con persistencia real.
- Reforzar diferencias Spring Framework vs Spring Boot antes de programar.
- Dejar el proyecto `product-service` listo para refactorizar en Clase 3.

## Preparación previa
- Verificar que todos tengan Java 17, Maven 3.9+ y Docker funcionando (pedir captura/lista).
- Clonar el repo oficial y abrir `bloque-springboot/clase2-rest-jpa`.
- Descargar con anticipación la imagen `postgres:15-alpine` para evitar esperas.
- Tener Postman (o alternativa) con la colección incluida (`recursos/postman/product-service.postman_collection.json`).
- Tener GitHub/Drive listo para compartir snippets de código si alguien se retrasa.

## Apertura sugerida
1. Recordar que Clase 1 fue teoría + setup; hoy se arranca con código.
2. Preguntar quién logró completar la tarea (entorno listo) y resolver bloqueos críticos.
3. Mostrar breve slide/diagrama: diferencias Spring Framework vs Spring Boot (autoconfiguración, starters, servidor embebido).

## Secuencia de actividades
1. **Lab 00 · Crear proyecto**  
   - Guiar la generación del proyecto `product-service`.  
   - Recalcar el `groupId dev.alefiengo` y `Spring Web` único.
   - Validar ejecución con `mvn spring-boot:run` (esperar “Tomcat started…”).
2. **Lab 01 · Hola Mundo REST**  
   - Crear `HelloController`.  
   - Introducir `@RestController`, `@GetMapping`, `@PathVariable`.  
   - Mostrar prueba rápida con `curl` y Postman.
3. **Lab 02 · Parámetros y métodos HTTP**  
   - Ejemplo in-memory con mapa y `AtomicLong`.  
   - Cubrir GET/POST/PUT/DELETE, manejo de query params.  
   - Subrayar importancia de `ResponseEntity`.
4. **Lab 03 · PostgreSQL con Docker Compose**  
   - Revisar `docker-compose.yml`.  
   - Arrancar contenedor, comprobar con `docker compose ps` y `psql`.  
   - Recordar apagarlo al final (`docker compose down`).
5. **Lab 04 · Entidad + Spring Data JPA**  
   - Agregar dependencias JPA/PostgreSQL.  
   - Crear entidad `Product` y repositorio, configurar `application.yml`.  
   - Recordar conceptos de `@Entity`, `@Table`, `@Id`, `@GeneratedValue`.  
   - Mostrar logs de Hibernate creando tablas.
6. **Lab 05 · CRUD básico**  
   - Actualizar `ProductController` para consumir directamente `ProductRepository`.  
   - Repasar métodos clave (`findAll`, `findById`, `save`, `deleteById`).  
   - Manejar errores simples con `ResponseStatusException`.  
   - Probar cada endpoint con Postman/cURL y revisar la base de datos.

## Puntos de énfasis / frases clave
- “Spring Boot nos ahorra configuración manual y nos deja concentrarnos en la ruta de negocio.”
- “Persistimos porque necesitamos datos reales para validar la API antes de Kafka.”
- “Cada lab construye sobre el anterior; si alguien se queda atrás comparte el repositorio listo al final de cada bloque.”
- “En esta clase consumimos `ProductRepository` directo; en la siguiente refactorizamos a capas.”

## Anticipar preguntas frecuentes
- Error `Cannot resolve symbol RestController`: falta dependencia `spring-boot-starter-web`.
- `Failed to configure a DataSource`: credenciales incorrectas o contenedor PostgreSQL no levantado.
- `Table 'products' doesn't exist`: revisar `ddl-auto=update` y que la entidad esté en el paquete correcto.
- Docker en Windows con WSL: recordar ejecutar Docker Desktop.

## Pausas recomendadas
- Después de Lab 02: revisar que todos puedan consumir endpoints con curl o Postman.
- Tras levantar PostgreSQL: confirmar que `docker compose ps` está “Up”.
- Antes del CRUD final: comprobar que la base tiene la tabla `products`.

## Cierre sugerido
- Mostrar la colección Postman con todas las operaciones funcionando.
- Recordar la tarea (CRUD básico con repositorio) y enfatizar entrega en repositorio propio.
- Adelantar que la próxima clase se centra en arquitectura en capas, validaciones y manejo de errores.

## Material de apoyo
- Diagramas internos (clase2) para explicar capas, flujo HTTP y lifecycle de Spring Boot.
- Momento para cada diagrama:  
  - `01-arquitectura-capas.drawio` durante la introducción teórica de anotaciones y capas.  
  - `02-flujo-peticion-http.drawio` al presentar el flujo DispatcherServlet → Controller → Service → Repository.  
  - `03-springboot-startup.drawio` antes de ejecutar `mvn spring-boot:run` (explica el startup y beans principales).
- Colección Postman: `recursos/postman/product-service.postman_collection.json`.
- FAQ y cheatsheet publicados en la carpeta de la clase (para recomendar a estudiantes).
