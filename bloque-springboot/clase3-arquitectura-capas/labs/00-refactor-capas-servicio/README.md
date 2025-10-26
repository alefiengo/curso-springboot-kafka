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
import java.util.List;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import dev.alefiengo.productservice.dto.ProductRequest;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.service.ProductService;

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

**ProductRequest.java**:
```java
package dev.alefiengo.productservice.dto;

import java.math.BigDecimal;

public record ProductRequest(
    String name,
    String description,
    BigDecimal price,
    Integer stock
) {}
```

**ProductResponse.java**:
```java
package dev.alefiengo.productservice.dto;

import java.math.BigDecimal;
import java.time.OffsetDateTime;

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

**ProductMapper.java** (ubicar en `dev.alefiengo.productservice.mapper`):
```java
package dev.alefiengo.productservice.mapper;

import dev.alefiengo.productservice.dto.ProductRequest;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.model.Product;

public final class ProductMapper {

    private ProductMapper() {
        throw new AssertionError("Utility class, no debe instanciarse");
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

    public static Product toEntity(ProductRequest request, Product entity) {
        entity.setName(request.name());
        entity.setDescription(request.description());
        entity.setPrice(request.price());
        entity.setStock(request.stock());
        return entity;
    }
}
```

### Service refactorizado

**ProductService.java** (ubicar en `dev.alefiengo.productservice.service`):
```java
package dev.alefiengo.productservice.service;

import java.util.List;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import dev.alefiengo.productservice.dto.ProductRequest;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.exception.ResourceNotFoundException;
import dev.alefiengo.productservice.mapper.ProductMapper;
import dev.alefiengo.productservice.model.Product;
import dev.alefiengo.productservice.repository.ProductRepository;

@Service
@Transactional(readOnly = true)
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
            .toList();
    }

    public ProductResponse findById(Long id) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        return ProductMapper.toResponse(product);
    }

    @Transactional
    public ProductResponse create(ProductRequest request) {
        Product product = new Product();
        Product saved = repository.save(ProductMapper.toEntity(request, product));
        return ProductMapper.toResponse(saved);
    }

    @Transactional
    public ProductResponse update(Long id, ProductRequest request) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        Product updated = repository.save(ProductMapper.toEntity(request, product));
        return ProductMapper.toResponse(updated);
    }

    @Transactional
    public void delete(Long id) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        repository.delete(product);
    }
}
```

### Excepción reutilizable

**ResourceNotFoundException.java** (ubicar en `dev.alefiengo.productservice.exception`):
```java
package dev.alefiengo.productservice.exception;

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
