# Lab 02 · Manejo global de excepciones

## 1. Objetivo

Crear un manejador global de excepciones (`@RestControllerAdvice`) que capture errores del sistema y devuelva respuestas JSON consistentes con códigos HTTP apropiados.

---

## 2. Comandos a ejecutar

```bash
# 1. Ubicarse en el proyecto
cd ~/workspace/product-service

# 2. Crear paquete exception
mkdir -p src/main/java/dev/alefiengo/productservice/exception

# 3. Crear ErrorResponse
code src/main/java/dev/alefiengo/productservice/exception/ErrorResponse.java

# 4. Crear ResourceNotFoundException
code src/main/java/dev/alefiengo/productservice/exception/ResourceNotFoundException.java

# 5. Crear GlobalExceptionHandler
code src/main/java/dev/alefiengo/productservice/exception/GlobalExceptionHandler.java

# 6. Actualizar ProductService para lanzar excepciones
code src/main/java/dev/alefiengo/productservice/service/ProductService.java

# 7. Recompilar y ejecutar
mvn clean spring-boot:run
```

---

## 3. Desglose del comando

| Componente | Descripción |
|------------|-------------|
| `@RestControllerAdvice` | Anotación que marca la clase como manejador global de excepciones para controladores REST |
| `@ExceptionHandler` | Define qué excepción captura cada método del handler |
| `ResourceNotFoundException` | Excepción personalizada para recursos no encontrados (404) |
| `MethodArgumentNotValidException` | Excepción de Spring cuando falla Bean Validation |
| `ErrorResponse` | DTO que define el formato uniforme de errores |
| `HttpServletRequest` | Permite acceder a información de la petición (URI, headers) |
| `ResponseEntity.status()` | Construye respuesta HTTP con código de estado específico |
| `getBindingResult()` | Obtiene los errores de validación de campos |

---

## 4. Explicación detallada

### Paso 1: Crear ErrorResponse

```java
package dev.alefiengo.productservice.exception;

import java.time.Instant;

import org.springframework.http.HttpStatus;

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

    public static ErrorResponse generic(String message, String path) {
        return new ErrorResponse(Instant.now(), HttpStatus.INTERNAL_SERVER_ERROR.value(), "INTERNAL_ERROR", message, path);
    }
}
```

### Paso 2: Crear ResourceNotFoundException

```java
package dev.alefiengo.productservice.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

### Paso 3: Crear GlobalExceptionHandler

```java
package dev.alefiengo.productservice.exception;

import java.time.Instant;
import java.util.stream.Collectors;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import jakarta.servlet.http.HttpServletRequest;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.notFound(ex.getMessage(), request.getRequestURI()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        String message = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ResponseEntity.badRequest()
            .body(ErrorResponse.validation(message, request.getRequestURI()));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrity(DataIntegrityViolationException ex, HttpServletRequest request) {
        log.error("Error de integridad de datos: {}", ex.getMessage());
        String message = "Error de integridad de datos. Verifica que todos los campos requeridos estén presentes.";

        if (ex.getMessage() != null && ex.getMessage().contains("category_id")) {
            message = "La categoría es obligatoria";
        }

        return ResponseEntity.badRequest()
            .body(new ErrorResponse(
                Instant.now(),
                HttpStatus.BAD_REQUEST.value(),
                "DATA_INTEGRITY_ERROR",
                message,
                request.getRequestURI()
            ));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex, HttpServletRequest request) {
        log.error("Error inesperado: ", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.generic("Ocurrió un error inesperado", request.getRequestURI()));
    }
}
```

### Paso 4: Actualizar ProductService

Reemplaza `ResponseStatusException` por `ResourceNotFoundException`:

```java
public ProductResponse findById(Long id) {
    Product product = repository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Producto no encontrado con ID: " + id));
    return toResponse(product);
}
```

**Resultado**: Todas las excepciones retornan JSON consistente con timestamp, status, código, mensaje y path.

---

## 5. Conceptos aprendidos

- Manejo centralizado de excepciones con `@RestControllerAdvice`.
- Formato uniforme de errores mejora la observabilidad y las integraciones front-end.
- `MethodArgumentNotValidException` captura los errores de Bean Validation automáticamente.
- Manejo proactivo de `Exception` permite devolver mensajes controlados incluso cuando algo inesperado ocurre.
- Códigos HTTP apropiados: 404 (Not Found), 400 (Bad Request), 500 (Internal Server Error).

---

## 6. Troubleshooting

- **El handler no se ejecuta**: verifica el paquete (debe ser escaneado por Spring) y la anotación `@RestControllerAdvice`.
- **Mensaje vacío en validaciones**: comprueba que la lógica de `Collectors.joining` recorra errores con `getDefaultMessage` y que existan mensajes definidos.
- **Stacktrace expuesto**: evita devolver `ex.toString()` al usuario; usa mensajes resumidos.
- **ResourceNotFoundException no se resuelve**: verifica imports y que la clase esté en el paquete correcto.

---

## 7. Desafío adicional/final

Agrega un handler adicional para `DataIntegrityViolationException` de Spring (violaciones de claves únicas) y responde con `409 Conflict`.

---

## 8. Recursos adicionales

- [Spring @ControllerAdvice](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-controller-advice)
- [Error Handling for REST](https://www.baeldung.com/exception-handling-for-rest-with-spring)
- [HTTP Status Codes](https://developer.mozilla.org/es/docs/Web/HTTP/Status)
