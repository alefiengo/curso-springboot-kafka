# Lab 01 · Validaciones con Bean Validation

## 1. Objetivo

Agregar validaciones a los DTOs de `product-service` usando Bean Validation, asegurando que las peticiones inválidas respondan con código `400 Bad Request`.

---

## 2. Pasos

1. **Agregar dependencia (si no se añadió antes)**  
   Abre `pom.xml` en tu IDE y dentro del bloque `<dependencies>` añade:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-validation</artifactId>
   </dependency>
   ```
2. **Crear (o actualizar) `ValidationMessages.properties`**  
   Ubicado en `src/main/resources/`:
   ```properties
   product.name.notblank=El nombre es obligatorio
   product.price.min=El precio debe ser mayor que cero
   product.stock.min=El stock no puede ser negativo
   ```
3. **Configurar `application.yml` para usar mensajes personalizados**  
   ```yaml
   spring:
     messages:
       basename: ValidationMessages
   ```
4. **Actualizar el DTO `ProductRequest`** (mismo paquete que el lab anterior):
   ```java
   public record ProductRequest(
           @NotBlank(message = "{product.name.notblank}")
           @Size(max = 120)
           String name,

           @Size(max = 255)
           String description,

           @NotNull
           @DecimalMin(value = "0.0", inclusive = false, message = "{product.price.min}")
           BigDecimal price,

           @NotNull
           @PositiveOrZero(message = "{product.stock.min}")
           Integer stock
   ) {}
   ```
5. **Aplicar `@Valid` en el controller**:
   ```java
   @PostMapping
   public ResponseEntity<ProductResponse> create(@Valid @RequestBody ProductRequest request) {
       return ResponseEntity.status(HttpStatus.CREATED).body(service.create(request));
   }

   @PutMapping("/{id}")
   public ResponseEntity<ProductResponse> update(@PathVariable Long id,
                                                 @Valid @RequestBody ProductRequest request) {
       return ResponseEntity.ok(service.update(id, request));
   }
   ```

---

## 3. Conceptos aprendidos

- Cómo activar Bean Validation en los endpoints REST.
- Uso de archivos de propiedades para mensajes reutilizables.
- Integración natural entre Spring MVC y `@Valid` / `jakarta.validation`.

---

## 4. Troubleshooting

- **Las validaciones no se ejecutan**: confirma que agregaste `@Valid` en el controller y que la app se reinició tras editar el `pom.xml`.
- **Los mensajes aparecen en inglés**: verifica `spring.messages.basename` y que el archivo se llame `ValidationMessages.properties`.
- **Errores de compilación**: usa imports de `jakarta.validation.*` (Spring Boot 3).

---

## 5. Desafío adicional/final

Añade reglas para validar nombres sin números (`@Pattern`) o combinar `@Positive` con controles adicionales en el service.

---

## 6. Recursos adicionales

- [Jakarta Bean Validation 3.0](https://jakarta.ee/specifications/bean-validation/3.0/)
- [Spring Boot Validation Guide](https://spring.io/guides/gs/validating-form-input/)
