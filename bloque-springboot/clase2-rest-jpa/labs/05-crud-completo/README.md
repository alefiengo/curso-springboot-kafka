# Lab 05 · CRUD básico con Spring Data JPA

## 1. Objetivo

Conectar el controlador REST directamente con `ProductRepository` para lograr un CRUD funcional sobre la entidad `Product` persistida en PostgreSQL.

---

## 2. Comandos a ejecutar

```bash
# 1. Ubicarse en el proyecto
cd ~/workspace/product-service

# 2. Abrir el controlador existente
code src/main/java/dev/alefiengo/productservice/controller/ProductController.java
```

Actualiza el contenido del controlador con la siguiente versión simplificada:

```java
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
import org.springframework.web.server.ResponseStatusException;

import dev.alefiengo.productservice.model.Product;
import dev.alefiengo.productservice.repository.ProductRepository;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductRepository repository;

    public ProductController(ProductRepository repository) {
        this.repository = repository;
    }

    @GetMapping
    public List<Product> list(@RequestParam(required = false) String name) {
        return (name == null || name.isBlank())
            ? repository.findAll()
            : repository.findByNameContainingIgnoreCase(name);
    }

    @GetMapping("/{id}")
    public Product getById(@PathVariable Long id) {
        return repository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Producto no encontrado"));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product create(@RequestBody Product request) {
        return repository.save(request);
    }

    @PutMapping("/{id}")
    public Product update(@PathVariable Long id, @RequestBody Product request) {
        Product current = repository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Producto no encontrado"));

        current.setName(request.getName());
        current.setDescription(request.getDescription());
        current.setPrice(request.getPrice());
        current.setStock(request.getStock());
        return repository.save(current);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        if (!repository.existsById(id)) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Producto no encontrado");
        }
        repository.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## 3. Desglose del comando

| Componente | Descripción |
|------------|-------------|
| `cd ~/workspace/product-service` | Navega al directorio del proyecto |
| `code src/main/.../ProductController.java` | Abre el controlador en el editor |
| `@RestController` | Marca la clase como controlador REST |
| `@RequestMapping("/api/products")` | Prefijo de ruta para todos los endpoints |
| `ProductRepository repository` | Inyección de dependencia del repositorio |
| `repository.findAll()` | Método de Spring Data JPA para obtener todos los registros |
| `repository.findById(id)` | Busca un registro por ID, retorna `Optional<Product>` |
| `repository.save(entity)` | Guarda o actualiza una entidad |
| `repository.deleteById(id)` | Elimina un registro por ID |
| `ResponseStatusException` | Lanza excepciones HTTP con códigos de estado |

---

## 4. Explicación detallada

1. **Uso directo de `ProductRepository`**: el controlador delega en Spring Data JPA para todas las operaciones.
2. **Búsqueda filtrada**: `findByNameContainingIgnoreCase` permite filtrar por nombre sin implementar lógica adicional.
3. **Validación mínima**: si el producto no existe, se lanza `ResponseStatusException` con estado `404`.
4. **Actualización con setters**: se reutiliza la entidad existente y luego se guarda.

---

## 5. Conceptos aprendidos

- Cómo exponer un CRUD completo sin capas adicionales.
- Uso directo de repositorios Spring Data en un controlador REST.
- Manejo básico de errores HTTP (`404`) con `ResponseStatusException`.
- Operaciones CRUD básicas: Create, Read, Update, Delete.
- Búsqueda con query methods de Spring Data JPA.

---

## 6. Troubleshooting

- **`ResponseStatusException` no se resuelve**: importa `org.springframework.web.server.ResponseStatusException`.
- **Los datos no se actualizan**: asegúrate de que el contenedor de PostgreSQL esté en ejecución (`docker compose ps`).
- **Error al crear producto (`price` nulo)**: recuerda enviar todos los campos requeridos en el JSON (name, description, price, stock).

---

## 7. Desafío adicional/final

- Añade `GET /api/products/count` que devuelva el total de registros usando `repository.count()`.
- Implementa `DELETE /api/products` para borrar todos los productos (`repository.deleteAll()`).

---

## 8. Recursos adicionales

- [Spring Data Repositories](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.repositories)
- [ResponseStatusException](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/server/ResponseStatusException.html)
