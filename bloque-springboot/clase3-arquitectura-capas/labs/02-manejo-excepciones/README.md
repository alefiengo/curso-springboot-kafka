# Lab 02 · Manejo global de excepciones

## 1. Objetivo

Crear un manejador global de excepciones (`@RestControllerAdvice
public class GlobalExceptionHandler {

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

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.generic("Ocurrió un error inesperado", request.getRequestURI()));
    }
}
```bash
# 1. Ubicarse en el proyecto
cd ~/workspace/product-service

# 2. Crear paquete exception
mkdir -p src/main/java/dev/alefiengo/productservice/exception
```

---

## 3. Desglose del código

### ErrorResponse
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

### Handler global
```java
package dev.alefiengo.productservice.exception;

import java.util.stream.Collectors;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import dev.alefiengo.productservice.exception.ResourceNotFoundException;
import jakarta.servlet.http.HttpServletRequest;

@RestControllerAdvice
public class GlobalExceptionHandler {

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

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.generic("Ocurrió un error inesperado", request.getRequestURI()));
    }
}
```

---

## 4. Explicación detallada

1. **`ErrorResponse`** define el formato de error: timestamp, status HTTP, código, mensaje y path.
2. **Handlers específicos**: distinguen entre recursos no encontrados, errores de validación y errores genéricos.
3. **`@RestControllerAdvice`** hace que el handler aplique a todos los controladores REST del proyecto.

---

## 5. Conceptos aprendidos

- Uniformar la respuesta de errores mejora la observabilidad y las integraciones front-end.
- `MethodArgumentNotValidException` captura los errores de Bean Validation.
- Manejo proactivo de `Exception` permite devolver mensajes controlados incluso cuando algo inesperado ocurre.

---

## 6. Troubleshooting

- **El handler no se ejecuta**: verifica el paquete (debe ser escaneado por Spring) y la anotación `@RestControllerAdvice`.
- **Mensaje vacío en validaciones**: comprueba que la lógica de `Collectors.joining` recorra errores con `getDefaultMessage` y que existan mensajes definidos.
- **Stacktrace expuesto**: evita devolver `ex.toString()` al usuario; usa mensajes resumidos.

---

## 7. Desafío adicional/final

Agrega un handler adicional para `DataIntegrityViolationException` de Spring (violaciones de claves únicas) y responde con `409 Conflict`.

---

## 8. Recursos adicionales

- [Spring @ControllerAdvice](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-controller-advice)
- [Error Handling for REST](https://www.baeldung.com/exception-handling-for-rest-with-spring)
