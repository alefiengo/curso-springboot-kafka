# Lab 03 · Relación Product–Category (1:N)

## 1. Objetivo

Introducir la entidad `Category`, establecer una relación `@ManyToOne` desde `Product` y exponer endpoints REST para gestionar categorías y consultar productos por categoría.

---

## 2. Comandos a ejecutar

```bash
# 1. Ubicarse en el proyecto
cd ~/workspace/product-service

# 2. Crear paquete exception si no existe
mkdir -p src/main/java/dev/alefiengo/productservice/exception

# 3. Actualizar DTOs, mapper y servicios en tu IDE
```

---

## 3. Desglose del código

### Entidad Category

**Category.java** (crear en `dev.alefiengo.productservice.model`):
```java
package dev.alefiengo.productservice.model;

import java.util.ArrayList;
import java.util.List;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.FetchType;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;

@Entity
@Table(name = "categories")
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 80, unique = true)
    private String name;

    @Column(length = 255)
    private String description;

    @OneToMany(mappedBy = "category", fetch = FetchType.LAZY)
    private List<Product> products = new ArrayList<>();

    // Getters y setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public List<Product> getProducts() {
        return products;
    }

    public void setProducts(List<Product> products) {
        this.products = products;
    }
}
```

### Ajustes en Product

**Product.java** - Agregar imports y campo `category` en la entidad existente:

```java
// Agregar estos imports al inicio del archivo:
import jakarta.persistence.FetchType;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;

// En la clase Product, agregar este campo después de los campos existentes:

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "category_id", nullable = false)
private Category category;

// Agregar getter y setter:

public Category getCategory() {
    return category;
}

public void setCategory(Category category) {
    this.category = category;
}
```

### Repositorios

**CategoryRepository.java** (crear en `dev.alefiengo.productservice.repository`):
```java
package dev.alefiengo.productservice.repository;

import org.springframework.data.jpa.repository.JpaRepository;

import dev.alefiengo.productservice.model.Category;

public interface CategoryRepository extends JpaRepository<Category, Long> {
    boolean existsByNameIgnoreCase(String name);
}
```

**ProductRepository.java** (actualizar en `dev.alefiengo.productservice.repository`):
```java
package dev.alefiengo.productservice.repository;

import java.util.List;

import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;

import dev.alefiengo.productservice.model.Product;

public interface ProductRepository extends JpaRepository<Product, Long> {

    @EntityGraph(attributePaths = "category")
    @Override
    List<Product> findAll();

    @EntityGraph(attributePaths = "category")
    List<Product> findByNameContainingIgnoreCase(String name);

    @EntityGraph(attributePaths = "category")
    List<Product> findByCategoryId(Long categoryId);
}
```

### DTOs

**CategoryRequest.java** (crear en `dev.alefiengo.productservice.dto`):
```java
package dev.alefiengo.productservice.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record CategoryRequest(
    @NotBlank
    @Size(max = 80)
    String name,

    @Size(max = 255)
    String description
) {}
```

**CategoryResponse.java** (crear en `dev.alefiengo.productservice.dto`):
```java
package dev.alefiengo.productservice.dto;

public record CategoryResponse(
    Long id,
    String name,
    String description
) {}
```

**ProductRequest.java** (actualizar en `dev.alefiengo.productservice.dto`):
```java
package dev.alefiengo.productservice.dto;

import java.math.BigDecimal;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.PositiveOrZero;
import jakarta.validation.constraints.Size;

public record ProductRequest(
    @NotBlank
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
```

**ProductResponse.java** (actualizar en `dev.alefiengo.productservice.dto`):
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
    OffsetDateTime updatedAt,
    Long categoryId,
    String categoryName
) {}
```

### Mapper actualizado

**ProductMapper.java** (actualizar en `dev.alefiengo.productservice.mapper`):
```java
package dev.alefiengo.productservice.mapper;

import dev.alefiengo.productservice.dto.ProductRequest;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.model.Category;
import dev.alefiengo.productservice.model.Product;

public final class ProductMapper {

    private ProductMapper() {
        throw new AssertionError("Utility class, no debe instanciarse");
    }

    public static ProductResponse toResponse(Product product) {
        Category category = product.getCategory();
        return new ProductResponse(
            product.getId(),
            product.getName(),
            product.getDescription(),
            product.getPrice(),
            product.getStock(),
            product.getCreatedAt(),
            product.getUpdatedAt(),
            category != null ? category.getId() : null,
            category != null ? category.getName() : null
        );
    }

    public static Product toEntity(ProductRequest request, Product entity, Category category) {
        entity.setName(request.name());
        entity.setDescription(request.description());
        entity.setPrice(request.price());
        entity.setStock(request.stock());
        entity.setCategory(category);
        return entity;
    }
}
```

### Actualización de ProductService

**ProductService.java** (actualizar en `dev.alefiengo.productservice.service`):
```java
package dev.alefiengo.productservice.service;

import java.util.List;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import dev.alefiengo.productservice.dto.ProductRequest;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.exception.ResourceNotFoundException;
import dev.alefiengo.productservice.mapper.ProductMapper;
import dev.alefiengo.productservice.model.Category;
import dev.alefiengo.productservice.model.Product;
import dev.alefiengo.productservice.repository.CategoryRepository;
import dev.alefiengo.productservice.repository.ProductRepository;

@Service
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;

    public ProductService(ProductRepository productRepository, CategoryRepository categoryRepository) {
        this.productRepository = productRepository;
        this.categoryRepository = categoryRepository;
    }

    public List<ProductResponse> findAll(String name) {
        List<Product> products = (name == null || name.isBlank())
            ? productRepository.findAll()
            : productRepository.findByNameContainingIgnoreCase(name);
        return products.stream()
            .map(ProductMapper::toResponse)
            .toList();
    }

    public ProductResponse findById(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        return ProductMapper.toResponse(product);
    }

    @Transactional
    public ProductResponse create(ProductRequest request) {
        Category category = categoryRepository.findById(request.categoryId())
            .orElseThrow(() -> new ResourceNotFoundException("Categoría " + request.categoryId() + " no encontrada"));
        Product product = new Product();
        Product saved = productRepository.save(ProductMapper.toEntity(request, product, category));
        return ProductMapper.toResponse(saved);
    }

    @Transactional
    public ProductResponse update(Long id, ProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        Category category = categoryRepository.findById(request.categoryId())
            .orElseThrow(() -> new ResourceNotFoundException("Categoría " + request.categoryId() + " no encontrada"));
        Product updated = productRepository.save(ProductMapper.toEntity(request, product, category));
        return ProductMapper.toResponse(updated);
    }

    @Transactional
    public void delete(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Producto " + id + " no encontrado"));
        productRepository.delete(product);
    }
}
```

### CategoryController

**CategoryController.java** (crear en `dev.alefiengo.productservice.controller`):
```java
package dev.alefiengo.productservice.controller;

import java.util.List;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import dev.alefiengo.productservice.dto.CategoryRequest;
import dev.alefiengo.productservice.dto.CategoryResponse;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.service.CategoryService;
import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/categories")
public class CategoryController {

    private final CategoryService service;

    public CategoryController(CategoryService service) {
        this.service = service;
    }

    @GetMapping
    public ResponseEntity<List<CategoryResponse>> findAll() {
        return ResponseEntity.ok(service.findAll());
    }

    @PostMapping
    public ResponseEntity<CategoryResponse> create(@Valid @RequestBody CategoryRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(request));
    }

    @GetMapping("/{id}/products")
    public ResponseEntity<List<ProductResponse>> findProducts(@PathVariable Long id) {
        return ResponseEntity.ok(service.findProducts(id));
    }
}
```

### CategoryService

**CategoryService.java** (crear en `dev.alefiengo.productservice.service`):
```java
package dev.alefiengo.productservice.service;

import java.util.List;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import dev.alefiengo.productservice.dto.CategoryRequest;
import dev.alefiengo.productservice.dto.CategoryResponse;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.exception.CategoryAlreadyExistsException;
import dev.alefiengo.productservice.exception.ResourceNotFoundException;
import dev.alefiengo.productservice.mapper.ProductMapper;
import dev.alefiengo.productservice.model.Category;
import dev.alefiengo.productservice.repository.CategoryRepository;
import dev.alefiengo.productservice.repository.ProductRepository;

@Service
@Transactional(readOnly = true)
public class CategoryService {

    private final CategoryRepository categoryRepository;
    private final ProductRepository productRepository;

    public CategoryService(CategoryRepository categoryRepository, ProductRepository productRepository) {
        this.categoryRepository = categoryRepository;
        this.productRepository = productRepository;
    }

    public List<CategoryResponse> findAll() {
        return categoryRepository.findAll().stream()
            .map(category -> new CategoryResponse(category.getId(), category.getName(), category.getDescription()))
            .toList();
    }

    @Transactional
    public CategoryResponse create(CategoryRequest request) {
        if (categoryRepository.existsByNameIgnoreCase(request.name())) {
            throw new CategoryAlreadyExistsException(request.name());
        }
        Category category = new Category();
        category.setName(request.name());
        category.setDescription(request.description());
        Category saved = categoryRepository.save(category);
        return new CategoryResponse(saved.getId(), saved.getName(), saved.getDescription());
    }

    public List<ProductResponse> findProducts(Long id) {
        Category category = categoryRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Categoría " + id + " no encontrada"));
        return productRepository.findByCategoryId(category.getId()).stream()
            .map(ProductMapper::toResponse)
            .toList();
    }
}

```

**CategoryAlreadyExistsException.java** (crear en `dev.alefiengo.productservice.exception`):
```java
package dev.alefiengo.productservice.exception;

public class CategoryAlreadyExistsException extends RuntimeException {
    public CategoryAlreadyExistsException(String name) {
        super("La categoría " + name + " ya existe");
    }
}
```

### Actualización del handler global

**GlobalExceptionHandler.java** - Agregar este método a la clase existente en `dev.alefiengo.productservice.exception`:

```java
// Agregar este import si no existe:
import dev.alefiengo.productservice.exception.CategoryAlreadyExistsException;

// Agregar este método dentro de la clase GlobalExceptionHandler:

@ExceptionHandler(CategoryAlreadyExistsException.class)
public ResponseEntity<ErrorResponse> handleCategoryExists(CategoryAlreadyExistsException ex, HttpServletRequest request) {
    return ResponseEntity.status(HttpStatus.CONFLICT)
        .body(new ErrorResponse(Instant.now(), HttpStatus.CONFLICT.value(), "CATEGORY_EXISTS", ex.getMessage(), request.getRequestURI()));
}
```


---

## 4. Explicación detallada

1. **Entidad Category**: recibe anotaciones estándar y mantiene la colección de productos (solo para navegación interna).
2. **Relación en Product**: la columna `category_id` es obligatoria; se maneja con fetch LAZY para evitar cargas innecesarias.
3. **Endpoints nuevos**: exponen la creación de categorías y la consulta de productos por categoría.
4. **Validaciones extra**: previenen duplicados (`existsByNameIgnoreCase`) y responden con error usando el handler global (añade un handler para `CategoryAlreadyExistsException`).

---

## 5. Conceptos aprendidos

- Configuración de relaciones `@ManyToOne`/`@OneToMany`.
- Uso de `@EntityGraph` para reducir N+1 queries.
- Organización de servicios y controladores para entidades adicionales.

---

## 6. Troubleshooting

- **`could not execute statement` al crear tabla**: revisa si `category_id` falta o si la tabla `categories` no existe; reinicia Docker con `docker compose down` y `up`.
- **`DataIntegrityViolationException` al crear categoría**: probablemente el nombre ya existe; añade manejo en `CategoryService` y en el `@ControllerAdvice`.
- **`LazyInitializationException`**: no retornes entidades; utiliza DTOs y resuelve la relación en el service.

---

## 7. Desafío adicional/final

- Exponer `PUT /api/categories/{id}` y `DELETE /api/categories/{id}` respetando las reglas de negocio.
- Añadir filtro `GET /api/products?categoryId=1` combinando validaciones y `findByCategoryId`.

---

## 8. Recursos adicionales

- [Spring Data JPA – Relationships](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.entity-persistence)
- [Hibernate Entity Graphs](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#fetching-strategies-fetchgraphs)