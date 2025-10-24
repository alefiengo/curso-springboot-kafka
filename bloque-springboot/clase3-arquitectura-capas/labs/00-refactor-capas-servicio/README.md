# Lab 00 · Refactor a arquitectura en capas

## 1. Objetivo

Reorganizar el `product-service` para separar responsabilidades entre controller, service, repository y DTOs. Eliminaremos lógica de negocio del controlador y centralizaremos las operaciones CRUD en `ProductService` usando un mapper manual.

---

## 2. Comandos a ejecutar

```bash
# 1. Ubicarse en el proyecto
cd ~/workspace/product-service

# 2. Crear paquetes si no existen
dirs=("src/main/java/dev/alefiengo/productservice/dto" \
      "src/main/java/dev/alefiengo/productservice/mapper" \
      "src/main/java/dev/alefiengo/productservice/exception")
for dir in "${dirs[@]}"; do mkdir -p "$dir"; done

# 3. Abrir los archivos indicados en tu IDE para actualizarlos
```

---

## 3. Desglose del código

### Controller mínimo
```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService service;

    public ProductController(ProductService service) {
        this.service = service;
    }

    @GetMapping
    public ResponseEntity<List<ProductResponse>> findAll(@RequestParam(required = false) String name) {
        return ResponseEntity.ok(service.findAll(name));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> findById(@PathVariable Long id) {
        return ResponseEntity.ok(service.findById(id));
    }

    @PostMapping
    public ResponseEntity<ProductResponse> create(@RequestBody ProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(request));
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> update(@PathVariable Long id, @RequestBody ProductRequest request) {
        return ResponseEntity.ok(service.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### DTO inicial (ubicados en `dev.alefiengo.productservice.dto`)
```java
import java.math.BigDecimal;
import java.time.OffsetDateTime;

public record ProductRequest(
    String name,
    String description,
    BigDecimal price,
    Integer stock
) {}

public record ProductResponse(
    Long id,
    String name,
    String description,
    BigDecimal price,
    Integer stock,
    OffsetDateTime createdAt,
    OffsetDateTime updatedAt
) {}
```

### Mapper manual
```java
public final class ProductMapper {

    private ProductMapper() {
    }

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

### Service refactorizado
```java
import java.util.List;
import java.util.stream.Collectors;

import org.springframework.stereotype.Service;

import dev.alefiengo.productservice.exception.ResourceNotFoundException;
import dev.alefiengo.productservice.model.Product;
import dev.alefiengo.productservice.repository.ProductRepository;

@Service
public class ProductService {

    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    public List<ProductResponse> findAll(String name) {
        List<Product> products = (name == null || name.isBlank())
            ? repository.findAll()
            : repository.findByNameContainingIgnoreCase(name);
        return products.stream()
            .map(ProductMapper::toResponse)
            .collect(Collectors.toList());
    }

    public ProductResponse findById(Long id) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        return ProductMapper.toResponse(product);
    }

    public ProductResponse create(ProductRequest request) {
        Product product = new Product();
        ProductMapper.updateEntity(request, product);
        Product saved = repository.save(product);
        return ProductMapper.toResponse(saved);
    }

    public ProductResponse update(Long id, ProductRequest request) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        ProductMapper.updateEntity(request, product);
        return ProductMapper.toResponse(product);
    }

    public void delete(Long id) {
        if (!repository.existsById(id)) {
            throw new ResourceNotFoundException("Producto " + id + " no encontrado");
        }
        repository.deleteById(id);
    }
}
```

### Excepción reutilizable
```java
// Ubicar en dev.alefiengo.productservice.exception
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

---

## 4. Explicación detallada

1. **Controller sin lógica de negocio**: recibe la petición, delega todo al service y retorna `ProductResponse`.
2. **Service como orquestador**: centraliza validaciones, búsquedas y conversiones a DTO.
3. **Mapper estático**: evita repetición y facilita futuros cambios (por ejemplo, añadir campos).
4. **Excepción compartida**: define un mensaje claro y permite un manejo uniforme en el handler global.

---

## 5. Conceptos aprendidos

- Ventajas de la arquitectura en capas para mantenimiento y pruebas.
- Patrón DTO + mapper manual como contrato estable de la API.
- Uso de excepciones personalizadas para desacoplar el manejo de errores del controlador.

---

## 6. Troubleshooting

- **`ResourceNotFoundException` no encontrada**: revisa el paquete e import (`dev.alefiengo.productservice.exception`).
- **Faltan imports**: agrega `import java.util.List;` y `import java.util.stream.Collectors;` en el service.
- **Jackson serializa entidades con relaciones LAZY**: asegúrate de retornar DTOs (`ProductResponse`).
- **El controlador devuelve 500**: revisa que el service lance `ResourceNotFoundException` cuando el ID no exista y que el handler global la capture (se agrega en el Lab 02).

---

## 7. Desafío adicional/final

Crear un DTO `ProductSummaryResponse` con los campos `id`, `name` y `price` y exponer `GET /api/products/summaries`. Reutiliza el mapper para construir la lista.

---

## 8. Recursos adicionales

- [Spring Service Layer – Best Practices](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory)
- [DTO Pattern – Baeldung](https://www.baeldung.com/java-dto-pattern)
