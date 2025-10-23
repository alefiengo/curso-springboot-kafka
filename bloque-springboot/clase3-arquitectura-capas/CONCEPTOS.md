# Conceptos Fundamentales - Clase 3

Esta clase consolida patrones de arquitectura en capas dentro de `product-service`, introduce DTOs formales, valida datos con Bean Validation y estandariza el manejo de errores. También prepara una relación 1:N entre `Product` y `Category`, clave para continuar hacia entornos event-driven.

---

## 1. Arquitectura en Capas

```
Cliente HTTP → Controller → Service → Repository → Database
```

- **Controller (`@RestController`)**: Expone endpoints, valida input al recibir la petición y convierte las respuestas en DTOs.
- **Service (`@Service`)**: Encapsula la lógica de negocio, orquesta transacciones, delega en repositorios y decide qué DTO retornar.
- **Repository (`@Repository`)**: Encapsula la persistencia con JPA, expone métodos declarativos (`findByNameContainingIgnoreCase`).
- **Entity (`@Entity`)**: Representa tablas de la BD. Nunca se expone directamente a la capa REST.
- **DTO (Data Transfer Object)**: Define el contrato de la API. Evita que cambios en las entidades rompan clientes.

**Beneficios**: separación de responsabilidades, facilidad para probar componentes y posibilidad de evolucionar cada capa de forma independiente.

---

## 2. DTOs y Mapeo Manual

- `record ProductRequest(String name, String description, BigDecimal price, Integer stock)` para entradas.
- `record ProductResponse(Long id, String name, String description, BigDecimal price, Integer stock, OffsetDateTime createdAt, OffsetDateTime updatedAt)` para salidas.

Creación de un mapper manual:

```java
public final class ProductMapper {
    private ProductMapper() {}

    public static ProductResponse toResponse(Product product) {
        return new ProductResponse(
            product.getId(),
            product.getName(),
            product.getDescription(),
            product.getPrice(),
            product.getStock(),
            product.getCreatedAt(),
            product.getUpdatedAt()
        );
    }

    public static void updateEntity(ProductRequest request, Product entity) {
        entity.setName(request.name());
        entity.setDescription(request.description());
        entity.setPrice(request.price());
        entity.setStock(request.stock());
    }
}
```

> Frameworks como MapStruct o ModelMapper pueden incorporarse más adelante, pero primero es importante dominar el proceso manual.

---

## 3. Inversión de Control e Inyección de Dependencias (IoC/DI)

- Spring crea los beans y los inyecta usando el constructor.
- Evita `new` dentro de los controladores/servicios.
- Para pruebas, se puede mockear el repositorio sin cambiar el código de producción.

```java
@Service
public class ProductService {
    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }
}
```

---

## 4. Bean Validation (JSR 380)

Dependencia recomendada:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Anotaciones frecuentes:

| Anotación | Uso |
|-----------|-----|
| `@NotBlank` | Strings obligatorios (ignora espacios) |
| `@Size(min, max)` | Limitar longitud de cadenas/Listas |
| `@Min`, `@Max`, `@DecimalMin` | Límite numérico |
| `@Positive` / `@PositiveOrZero` | Números positivos |
| `@Email` | Formato de correo |

**Uso en controllers**:

```java
@PostMapping
public ResponseEntity<ProductResponse> create(@Valid @RequestBody ProductRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(service.create(request));
}
```

**Mensajes personalizados**: `@NotBlank(message = "El nombre es obligatorio")`

Para agrupar mensajes globales utiliza `ValidationMessages.properties` en `src/main/resources`.

---

## 5. Manejo Global de Excepciones

```
┌─────────────────┐
│ @ControllerAdvice│
├─────────────────┤
│ @ExceptionHandler│─────► ErrorResponse (timestamp, status, code, message, path)
└─────────────────┘
```

Ejemplo:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            Instant.now(),
            HttpStatus.NOT_FOUND.value(),
            "NOT_FOUND",
            ex.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        String message = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.joining("; "));

        ErrorResponse error = new ErrorResponse(
            Instant.now(),
            HttpStatus.BAD_REQUEST.value(),
            "VALIDATION_ERROR",
            message,
            request.getRequestURI()
        );
        return ResponseEntity.badRequest().body(error);
    }
}
```

---

## 6. Relación 1:N con JPA (Product ↔ Category)

```java
@Entity
@Table(name = "categories")
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 80)
    private String name;

    @Column(length = 255)
    private String description;

    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL, orphanRemoval = false, fetch = FetchType.LAZY)
    private List<Product> products = new ArrayList<>();
}
```

```java
@Entity
@Table(name = "products")
public class Product {
    // ...
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;
}
```

Buenas prácticas:
- Usa `FetchType.LAZY` en relaciones `@ManyToOne` para evitar cargas innecesarias.
- Controla las cascadas (`CascadeType.PERSIST` y `CascadeType.MERGE` suelen ser suficientes).
- Ajusta tus DTOs (`ProductRequest` con `categoryId`, `ProductResponse` con `categoryName`) y actualiza el mapper/service para cargar la categoría desde el repositorio.
- Para evitar N+1 Queries utiliza `@EntityGraph` o `JOIN FETCH` cuando necesites traer productos asociados.

---

## 7. Resumen de decisiones

- **DTOs**: contracto claro con clientes, evita exponer entidades.
- **Validación**: se dispara en el límite de entrada (controller), evitando lógica defensiva repetida.
- **Errores uniformes**: facilita observabilidad y troubleshooting.
- **Relación 1:N**: prepara el dominio para futuras integraciones (por ejemplo, mensajes Kafka sobre categorías).

---

## 8. Referencias recomendadas

- [Spring Boot Official Guide – Validating Form Input](https://spring.io/guides/gs/validating-form-input/)
- [Baeldung – Spring REST Exception Handling](https://www.baeldung.com/exception-handling-for-rest-with-spring)
- [Baeldung – DTO Pattern](https://www.baeldung.com/java-dto-pattern)
- [Hibernate/Microservices – Best practices for Many-to-One](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#associations-many-to-one)
