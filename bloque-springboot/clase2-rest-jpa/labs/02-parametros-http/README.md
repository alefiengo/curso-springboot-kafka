# Lab 02 · Parámetros y métodos HTTP

## 1. Objetivo

Practicar diferentes formas de recibir información en una API REST (ruta, query string y cuerpo JSON) y exponer métodos GET, POST, PUT y DELETE sin persistencia aún.

---

## 2. Comandos a ejecutar

```bash
# 1. Navegar al proyecto
cd ~/workspace/product-service

# 2. Crear el controlador de productos
cat <<'EOF' > src/main/java/dev/alefiengo/productservice/controller/ProductController.java
package dev.alefiengo.productservice.controller;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

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

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final Map<Long, ProductResponse> inMemoryStore = new ConcurrentHashMap<>();
    private final AtomicLong sequence = new AtomicLong(1);

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> findById(@PathVariable Long id,
                                                    @RequestParam(defaultValue = "false") boolean includeStock) {
        ProductResponse product = inMemoryStore.get(id);
        if (product == null) {
            return ResponseEntity.notFound().build();
        }
        if (!includeStock) {
            product = product.withStock(null);
        }
        return ResponseEntity.ok(product);
    }

    @GetMapping
    public ResponseEntity<Map<String, Object>> search(@RequestParam(required = false) String name,
                                                      @RequestParam(defaultValue = "0") int page,
                                                      @RequestParam(defaultValue = "10") int size) {
        Map<String, Object> filters = new HashMap<>();
        filters.put("page", page);
        filters.put("size", size);
        if (name != null && !name.isBlank()) {
            filters.put("name", name);
        }

        Map<String, Object> body = new HashMap<>();
        body.put("filters", filters);
        body.put("results", inMemoryStore.values());

        return ResponseEntity.ok(body);
    }

    @PostMapping
    public ResponseEntity<ProductResponse> create(@RequestBody ProductRequest request) {
        long id = sequence.getAndIncrement();
        ProductResponse created = ProductResponse.fromRequest(id, request);
        inMemoryStore.put(id, created);
        return ResponseEntity.ok(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> update(@PathVariable Long id, @RequestBody ProductRequest request) {
        if (!inMemoryStore.containsKey(id)) {
            return ResponseEntity.notFound().build();
        }
        ProductResponse updated = ProductResponse.fromRequest(id, request);
        inMemoryStore.put(id, updated);
        return ResponseEntity.ok(updated);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        inMemoryStore.remove(id);
        return ResponseEntity.noContent().build();
    }

    public record ProductRequest(String name, String description, Double price, Integer stock) { }

    public record ProductResponse(Long id, String name, String description, Double price, Integer stock, Instant updatedAt) {
        static ProductResponse fromRequest(Long id, ProductRequest request) {
            return new ProductResponse(
                id,
                request.name(),
                request.description(),
                request.price(),
                request.stock(),
                Instant.now()
            );
        }

        ProductResponse withStock(Integer newStock) {
            return new ProductResponse(id, name, description, price, newStock, updatedAt);
        }
    }
}
EOF

# 3. Ejecutar pruebas manuales
mvn spring-boot:run

# 4. Desde otra terminal, probar los endpoints
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","description":"Modelo 2025","price":1299.99,"stock":5}'

curl "http://localhost:8080/api/products/1?includeStock=true"

curl -X PUT http://localhost:8080/api/products/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","description":"Actualizada","price":1199.99,"stock":3}'

curl -X DELETE http://localhost:8080/api/products/1
```

---

## 3. Desglose del comando

| Comando | Descripción |
|---------|-------------|
| `ProductController` | Define endpoints `GET`, `POST`, `PUT` y `DELETE` sobre `/api/products`, utilizando una estructura en memoria para simular persistencia. |
| `@GetMapping("/{id}")` | Combina un parámetro de ruta con un `@RequestParam` opcional (`includeStock`). |
| `@GetMapping` | Usa query parameters (`name`, `page`, `size`) para realizar búsquedas filtradas. |
| `@PostMapping` / `@PutMapping` | Reciben un JSON y lo mapean al `record ProductRequest` a través de `@RequestBody`. |
| `curl -X POST ...` | Envía un cuerpo JSON que Spring deserializa automáticamente. |
| `curl -X PUT ...` y `curl -X DELETE ...` | Ejemplifican la actualización y eliminación de recursos con distintos métodos HTTP. |

---

## 4. Explicación detallada

1. **Uso de `@RequestParam`**  
   Los parámetros de query permiten personalizar la petición sin modificar la ruta. El atributo `defaultValue` evita `null` cuando el cliente no envía el parámetro y, para construir la respuesta, usamos un `HashMap` para tolerar valores opcionales (evitamos `Map.of(...)` porque no acepta `null`).
2. **`@RequestBody` y deserialización**  
   Spring Boot usa Jackson para convertir JSON en objetos Java. El `record ProductRequest` evita escribir getters/setters.
3. **Respuesta controlada con `ResponseEntity`**  
   Permite devolver códigos de estado apropiados (`200 OK`, `204 No Content`, `404 Not Found`).
4. **Almacenamiento temporal**  
   El `ConcurrentHashMap` y `AtomicLong` simulan una capa de datos hasta que integremos JPA en los siguientes labs. Esto se reemplazará después por repositorios reales.

---

## 5. Conceptos aprendidos

- Diferencias entre parámetros de ruta, query y cuerpo en una API REST.
- Declaración de múltiples métodos HTTP en un mismo controlador.
- Uso de `ResponseEntity` para codificar respuestas HTTP adecuadas.
- Ventajas de trabajar con `record` para requests/responses temporales.

---

## 6. Troubleshooting

- **Error 415 Unsupported Media Type**  
  Asegúrate de enviar `Content-Type: application/json` en peticiones POST/PUT.
- **Los parámetros de query llegan vacíos**  
  Revisa que la URL incluya `?name=valor` y que en el método el parámetro tenga el mismo nombre.
- **`NullPointerException` al listar sin filtros**  
  Si usaste `Map.of(...)` para construir la respuesta y `name` es `null`, se producirá el error porque `Map.of` no admite valores nulos. Cambia a `HashMap` como se muestra en el código del lab.
- **`ResponseEntity` retorna 404 siempre**  
  Confirma que el `id` exista en el mapa. Si reiniciaste la aplicación se pierde el estado.
- **Jackson no puede deserializar**  
  Verifica la sintaxis JSON (comillas dobles, coma final). Usa `jq` o Postman para validar el cuerpo.

---

## 7. Desafío adicional/final

Agrega un endpoint `PATCH /api/products/{id}/stock` que reciba un cuerpo con el nuevo valor de stock y devuelva la entidad actualizada. Aprovecha `ResponseEntity` para devolver `404` si el producto no existe.

---

## 8. Recursos adicionales

- [Documentación de `@RequestParam`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/RequestParam.html)
- [Documentación de `@RequestBody`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/RequestBody.html)
- [HTTP Methods – RFC 7231](https://www.rfc-editor.org/rfc/rfc7231)
