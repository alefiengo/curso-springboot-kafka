# Cheatsheet · Clase 3

Apoyo rápido para la sesión de arquitectura en capas, validaciones y manejo de errores.

---

## Maven / Spring Boot

```bash
# Ejecutar con reinicio rápido
mvn spring-boot:run

# Ejecutar tests (útil para validar excepciones custom)
mvn test

# Ver dependencias activas
mvn dependency:tree
```

---

## Bean Validation

```java
// DTO de entrada
public record ProductRequest(
    @NotBlank(message = "El nombre es obligatorio")
    @Size(max = 120)
    String name,

    @Size(max = 255)
    String description,

    @DecimalMin(value = "0.0", inclusive = false)
    BigDecimal price,

    @PositiveOrZero
    Integer stock
) {}
```

```java
@PostMapping
public ResponseEntity<ProductResponse> create(@Valid @RequestBody ProductRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(service.create(request));
}
```

---

## Manejo de excepciones

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.notFound(ex.getMessage(), request.getRequestURI()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ResponseEntity.badRequest()
            .body(ErrorResponse.validation(message, request.getRequestURI()));
    }
}
```

---

## psql rápido (dentro del contenedor)

```bash
docker compose exec postgres psql -U product_user -d product_db

-- Ver categorías creadas
SELECT id, name, description FROM categories;

-- Ver productos con categoría
SELECT p.id, p.name, c.name AS category
FROM products p
JOIN categories c ON p.category_id = c.id;
```

---

## cURL útiles

```bash
# Crear categoría
curl -X POST http://localhost:8080/api/categories \
  -H "Content-Type: application/json" \
  -d '{"name":"Electrónica","description":"Dispositivos"}'

# Crear producto asignado a categoría ID 1
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","description":"Ultrabook","price":1299.99,"stock":5,"categoryId":1}'

# Probar validaciones (precio negativo)
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","description":"Ultrabook","price":-1,"stock":5,"categoryId":1}'
```

---

## Repositorios Spring Data

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    @EntityGraph(attributePaths = "category")
    List<Product> findByCategoryId(Long categoryId);
}

public interface CategoryRepository extends JpaRepository<Category, Long> {
    boolean existsByNameIgnoreCase(String name);
}
```

---

## ErrorResponse sugerido

```java
public record ErrorResponse(
    Instant timestamp,
    int status,
    String code,
    String message,
    String path
) {
    public static ErrorResponse notFound(String message, String path) {
        return new ErrorResponse(Instant.now(), HttpStatus.NOT_FOUND.value(), "NOT_FOUND", message, path);
    }

    public static ErrorResponse validation(String message, String path) {
        return new ErrorResponse(Instant.now(), HttpStatus.BAD_REQUEST.value(), "VALIDATION_ERROR", message, path);
    }
}
```

---

## Recordatorios rápidos

- Usa `@Transactional` en los métodos del service que modifican datos.
- Evita exponer entidades directamente en los controladores.
- Cuando uses `FetchType.LAZY`, accede a la relación dentro del service para hidratar los DTOs.
- Documenta en el README del proyecto cómo reproducir errores y validar las respuestas.

---

## DTOs tras la relación 1:N

```java
public record ProductRequest(
    @NotBlank(message = "{product.name.notblank}")
    @Size(max = 120)
    String name,
    @Size(max = 255)
    String description,
    @NotNull
    @DecimalMin(value = "0.0", inclusive = false)
    BigDecimal price,
    @NotNull
    @PositiveOrZero
    Integer stock,
    @NotNull
    Long categoryId
) {}

public record ProductResponse(
    Long id,
    String name,
    String description,
    BigDecimal price,
    Integer stock,
    OffsetDateTime createdAt,
    OffsetDateTime updatedAt,
    Long categoryId,
    String categoryName
) {}
```
