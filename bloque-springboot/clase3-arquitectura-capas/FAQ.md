# FAQ · Clase 3

Preguntas frecuentes y soluciones rápidas relacionadas con la refactorización en capas, validaciones y manejo de excepciones.

---

## Arquitectura en capas

### El controller sigue llamando a `ProductRepository` directamente
Asegúrate de que `ProductController` use únicamente `ProductService`. Inyecta el service por constructor y mueve toda la lógica CRUD al service.

### `Circular view path` o errores al reorganizar paquetes
Revisa que el paquete raíz (`dev.alefiengo.productservice`) contenga `ProductServiceApplication`. Si moviste clases fuera de este paquete, actualiza `@ComponentScan` o regresa la estructura original.

---

## Bean Validation

### Las validaciones no se disparan
Añade `@Valid` en el parámetro del controller y agrega la dependencia `spring-boot-starter-validation` en el `pom.xml`. Sin `@Valid`, Spring no ejecuta Bean Validation.

### Los mensajes de error aparecen en inglés
Declara `ValidationMessages.properties` en `src/main/resources` y utiliza `message = "..."` en las anotaciones. También puedes configurar `spring.messages.basename=ValidationMessages` en `application.yml`.

### Error `Failed to instantiate [javax.validation.Validator]`
Ocurre cuando falta la dependencia de validación o hay versiones incompatibles. Usa la BOM de Spring Boot y agrega `spring-boot-starter-validation`.

---

## Manejo de excepciones

### `ResponseStatusException` funciona pero no mi `@ControllerAdvice`
Verifica que la clase esté anotada con `@RestControllerAdvice` y que se encuentre bajo el paquete raíz para que el `ComponentScan` la detecte.

### `LazyInitializationException` cuando serializo `Product`
Si expones directamente la entidad, Jackson intenta acceder a relaciones LAZY fuera de la transacción. Usa DTOs e hidrata los datos en el service.

### Los errores de validación devuelven un cuerpo vacío
Asegúrate de capturar `MethodArgumentNotValidException` en el handler global y construir correctamente el `ErrorResponse`. También verifica que el controller devuelva `ResponseEntity`.

---

## Relación Product–Category

### `NullPointerException` al crear un producto sin categoría
Asegúrate de que tras el Lab 03 el `ProductRequest` incluye `categoryId` y el service carga la categoría con `categoryRepository.findById`. Si no se encuentra, lanza `ResourceNotFoundException`.

### Violación de clave foránea al eliminar categoría
Define la regla de negocio: prohibir eliminar categorías con productos asociados o habilitar `cascade = CascadeType.REMOVE` (no recomendado en escenarios reales). En la práctica, lanza una excepción con un mensaje claro y maneja el error con el handler global.

### `CategoryAlreadyExistsException` / nombre duplicado
Verifica que exista un handler en `GlobalExceptionHandler` que responda `409 Conflict` cuando la categoría ya esté registrada.

### `Unique constraint violation` al crear categorías
Puedes utilizar un constraint de negocio verificando `categoryRepository.existsByNameIgnoreCase(name)` antes de persistir. Responde con `400 Bad Request` si la categoría ya existe.

---

## Docker / PostgreSQL

### Cambié el modelo y la tabla no se actualiza
`ddl-auto=update` cubre la mayoría de escenarios, pero cambios drásticos pueden requerir eliminar el volumen (`docker compose down -v`) y recrear la BD. Alternativamente, usa migraciones con Flyway/Liquibase.

### Errores de conexión después de reiniciar Docker
Verifica `docker compose ps`. Si el contenedor se reinicia en bucle, revisa los logs (`docker compose logs postgres`) y comprueba credenciales en `application.yml`.

---

Mantén esta FAQ actualizada con dudas surgidas durante las sesiones para agilizar la resolución de problemas.
