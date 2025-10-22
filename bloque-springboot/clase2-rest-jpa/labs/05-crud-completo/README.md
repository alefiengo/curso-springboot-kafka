# Lab 05 · CRUD REST completo con Spring Data JPA

## 1. Objetivo

Exponer un CRUD completo de productos utilizando Spring Data JPA, servicios con transacciones y DTOs para separar la capa web de la entidad persistente.

---

## 2. Comandos a ejecutar

```bash
# 1. Ubicarse en el proyecto
cd ~/workspace/product-service

# 2. Eliminar el controlador provisional del Lab 02
rm -f src/main/java/dev/alefiengo/productservice/controller/ProductController.java

# 3. Crear los DTOs
mkdir -p src/main/java/dev/alefiengo/productservice/dto
cat <<'EOF' > src/main/java/dev/alefiengo/productservice/dto/ProductRequest.java
package dev.alefiengo.productservice.dto;

import java.math.BigDecimal;

public record ProductRequest(
        String name,
        String description,
        BigDecimal price,
        Integer stock
) { }
EOF

cat <<'EOF' > src/main/java/dev/alefiengo/productservice/dto/ProductResponse.java
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
) { }
EOF

# 4. Crear la capa de servicio
mkdir -p src/main/java/dev/alefiengo/productservice/service
cat <<'EOF' > src/main/java/dev/alefiengo/productservice/service/ProductService.java
package dev.alefiengo.productservice.service;

import java.util.List;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

import dev.alefiengo.productservice.dto.ProductRequest;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.model.Product;
import dev.alefiengo.productservice.repository.ProductRepository;

@Service
public class ProductService {

    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public List<ProductResponse> findAll(String name) {
        List<Product> products = (name == null || name.isBlank())
            ? repository.findAll()
            : repository.findByNameContainingIgnoreCase(name);
        return products.stream()
            .map(this::toResponse)
            .toList();
    }

    @Transactional(readOnly = true)
    public ProductResponse findById(Long id) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Producto no encontrado"));
        return toResponse(product);
    }

    @Transactional
    public ProductResponse create(ProductRequest request) {
        Product product = new Product();
        applyRequest(request, product);
        Product saved = repository.save(product);
        return toResponse(saved);
    }

    @Transactional
    public ProductResponse update(Long id, ProductRequest request) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Producto no encontrado"));
        applyRequest(request, product);
        return toResponse(product);
    }

    @Transactional
    public void delete(Long id) {
        if (!repository.existsById(id)) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Producto no encontrado");
        }
        repository.deleteById(id);
    }

    private void applyRequest(ProductRequest request, Product product) {
        product.setName(request.name());
        product.setDescription(request.description());
        product.setPrice(request.price());
        product.setStock(request.stock());
    }

    private ProductResponse toResponse(Product product) {
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
}
EOF

# 5. Crear el controlador definitivo
cat <<'EOF' > src/main/java/dev/alefiengo/productservice/controller/ProductController.java
package dev.alefiengo.productservice.controller;

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
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

import dev.alefiengo.productservice.dto.ProductRequest;
import dev.alefiengo.productservice.dto.ProductResponse;
import dev.alefiengo.productservice.service.ProductService;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public List<ProductResponse> list(@RequestParam(required = false) String name) {
        return productService.findAll(name);
    }

    @GetMapping("/{id}")
    public ProductResponse getById(@PathVariable Long id) {
        return productService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@RequestBody ProductRequest request) {
        return productService.create(request);
    }

    @PutMapping("/{id}")
    public ProductResponse update(@PathVariable Long id, @RequestBody ProductRequest request) {
        return productService.update(id, request);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
EOF

# 6. Ejecutar la aplicación y probar el flujo completo
mvn spring-boot:run
```

Desde otra terminal:

```bash
curl -X POST http://localhost:8080/api/products -H "Content-Type: application/json" \
  -d '{"name":"Cámara","description":"Mirrorless","price":899.90,"stock":7}'

curl http://localhost:8080/api/products
curl http://localhost:8080/api/products/1
curl -X PUT http://localhost:8080/api/products/1 -H "Content-Type: application/json" \
  -d '{"name":"Cámara","description":"Mirrorless 2025","price":949.90,"stock":5}'
curl -X DELETE http://localhost:8080/api/products/1
```

---

## 3. Desglose del comando

| Elemento | Descripción |
|----------|-------------|
| DTOs `ProductRequest` y `ProductResponse` | Separan la API pública de la entidad JPA, evitando exponer campos internos. |
| `ProductService` | Centraliza la lógica de negocio, aplica mapeos y maneja transacciones (`@Transactional`). |
| `ProductController` | Expone endpoints CRUD (`GET`, `POST`, `PUT`, `DELETE`) apoyándose en la capa de servicio. |
| `ResponseStatusException` | Permite lanzar errores semánticos y traducirlos a códigos HTTP apropiados. |
| Pruebas con `curl` | Validan el ciclo completo: creación, consulta, actualización y eliminación. |

---

## 4. Explicación detallada

1. **DTOs dedicados**  
   Evitan exponer `Product` directamente. `ProductRequest` representa la carga útil de entrada; `ProductResponse` es la salida enviada al cliente.
2. **Servicio y transacciones**  
   Métodos de lectura se marcan `readOnly=true` para optimizar acceso. Las operaciones de escritura usan el `EntityManager` subyacente de Hibernate al modificar la entidad gestionada.
3. **Controlador**  
   Mantiene responsabilidades de orquestación: delega en el servicio, controla rutas, códigos de estado y conversión de parámetros.
4. **Códigos HTTP**  
   - `POST` devuelve `201 Created`.  
   - `GET` retorna `200 OK` con contenido.  
   - `DELETE` responde `204 No Content`.  
   - Errores de recurso inexistente producen `404 Not Found` gracias a `ResponseStatusException`.
5. **Verificación**  
   Usa `docker compose exec postgres psql -U product_user -d product_db -c "select * from products"` para comprobar que las operaciones modifican la base.

---

## 5. Conceptos aprendidos

- Diseño de capas Controller → Service → Repository.
- Creación de DTOs para desacoplar la API de la entidad.
- Manejo de transacciones con anotaciones de Spring.
- Búsqueda condicional con métodos derivados (`findByNameContainingIgnoreCase`).
- Uso de `ResponseEntity` y `ResponseStatusException` para respuestas HTTP claras.

---

## 6. Troubleshooting

- **`HTTP 400` al invocar POST**  
  Revisa el cuerpo JSON (tipos numéricos y nombres de campos). `BigDecimal` requiere números sin comillas.
- **`HTTP 404` en GET/PUT/DELETE**  
  Confirma que el `id` exista. Revisa en la base con un `SELECT` o en la respuesta de la creación.
- **`could not execute statement`**  
  Suele indicar violaciones de restricciones (campos nulos, longitud). Verifica la entidad y la entrada.
- **`No serializer found for class`**  
  Asegúrate de devolver `ProductResponse` en todas las respuestas y no la entidad `Product`.

---

## 7. Desafío adicional/final

Implementa un endpoint `GET /api/products/search` que acepte `minPrice` y `maxPrice` como parámetros de query y filtre los resultados delegando en un método adicional del repositorio (`findByPriceBetween`). Documenta los nuevos parámetros en el README del proyecto.

---

## 8. Recursos adicionales

- [Documentación de `@Transactional`](https://docs.spring.io/spring-framework/reference/data-access/transaction/annotation-driven.html)
- [Best Practices for DTOs](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#projections)
- [ResponseEntity Reference](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseEntity.html)
