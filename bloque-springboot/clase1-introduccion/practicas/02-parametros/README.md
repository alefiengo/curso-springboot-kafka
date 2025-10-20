# Práctica 2: REST con Parámetros Dinámicos

**Nivel:** Principiante

---

## 1. Objetivo

Aprender a crear endpoints REST que acepten parámetros dinámicos, tanto en la ruta (path variables) como en query strings, para construir APIs más flexibles e interactivas.

**Al finalizar esta práctica podrás:**
- Capturar parámetros de la URL con `@PathVariable`
- Capturar parámetros de query string con `@RequestParam`
- Manejar parámetros opcionales con valores por defecto
- Combinar múltiples parámetros en un mismo endpoint
- Validar y usar parámetros en la lógica del controlador

---

## 2. Comandos a Ejecutar

### Paso 1: Crear nuevo proyecto

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,devtools \
  -d groupId=dev.alefiengo \
  -d artifactId=parametros-demo \
  -d name=ParametrosDemo \
  -d packageName=dev.alefiengo.parametrosdemo \
  -d javaVersion=17 \
  -o parametros-demo.zip

unzip parametros-demo.zip -d parametros-demo
cd parametros-demo
```

### Paso 2: Crear controlador con parámetros

**Archivo:** `src/main/java/dev/alefiengo/parametrosdemo/controller/GreetingController.java`

```java
package dev.alefiengo.parametrosdemo.controller;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class GreetingController {

    // 1. Path Variable - Parámetro en la ruta
    @GetMapping("/hello/{name}")
    public String helloName(@PathVariable String name) {
        return "Hola, " + name + "!";
    }

    // 2. Query Parameter - Parámetro opcional
    @GetMapping("/greet")
    public String greet(@RequestParam(defaultValue = "Invitado") String name) {
        return "Bienvenido, " + name;
    }

    // 3. Múltiples Path Variables
    @GetMapping("/user/{id}/profile/{section}")
    public String userProfile(@PathVariable Long id, @PathVariable String section) {
        return "Usuario ID: " + id + " | Sección: " + section;
    }

    // 4. Múltiples Query Parameters
    @GetMapping("/search")
    public String search(
            @RequestParam String query,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int size) {
        return "Buscando: '" + query + "' | Página: " + page + " | Tamaño: " + size;
    }

    // 5. Combinación de Path Variable y Query Params
    @GetMapping("/products/{category}")
    public String productsByCategory(
            @PathVariable String category,
            @RequestParam(defaultValue = "price") String sortBy,
            @RequestParam(defaultValue = "asc") String order) {
        return "Categoría: " + category + " | Ordenar por: " + sortBy + " " + order;
    }

    // 6. Parámetro opcional (puede ser null)
    @GetMapping("/welcome")
    public String welcome(@RequestParam(required = false) String name) {
        if (name == null) {
            return "Bienvenido, visitante anónimo";
        }
        return "Bienvenido, " + name;
    }
}
```

### Paso 3: Ejecutar la aplicación

```bash
mvn spring-boot:run
```

### Paso 4: Probar los endpoints

```bash
# 1. Path Variable
curl http://localhost:8080/api/hello/Juan

# 2. Query Parameter con valor
curl http://localhost:8080/api/greet?name=Maria

# 3. Query Parameter sin valor (usa default)
curl http://localhost:8080/api/greet

# 4. Múltiples Path Variables
curl http://localhost:8080/api/user/123/profile/settings

# 5. Múltiples Query Parameters
curl "http://localhost:8080/api/search?query=spring+boot&page=2&size=20"

# 6. Combinación
curl "http://localhost:8080/api/products/electronics?sortBy=name&order=desc"

# 7. Parámetro opcional presente
curl http://localhost:8080/api/welcome?name=Carlos

# 8. Parámetro opcional ausente
curl http://localhost:8080/api/welcome
```

**Respuestas esperadas:**
```
Hola, Juan!
Bienvenido, Maria
Bienvenido, Invitado
Usuario ID: 123 | Sección: settings
Buscando: 'spring boot' | Página: 2 | Tamaño: 20
Categoría: electronics | Ordenar por: name desc
Bienvenido, Carlos
Bienvenido, visitante anónimo
```

---

## 3. Desglose del Comando

### Anotaciones de Parámetros

| Anotación | Uso | Ejemplo |
|-----------|-----|---------|
| `@PathVariable` | Captura variable de la URL | `/hello/{name}` → `@PathVariable String name` |
| `@RequestParam` | Captura query parameter | `/greet?name=Juan` → `@RequestParam String name` |
| `defaultValue` | Valor por defecto si falta | `@RequestParam(defaultValue = "10") int size` |
| `required` | Si el parámetro es obligatorio | `@RequestParam(required = false) String name` |
| `name` | Nombre del parámetro si difiere | `@RequestParam(name = "q") String query` |

### Diferencias clave

| Aspecto | @PathVariable | @RequestParam |
|---------|---------------|---------------|
| **Ubicación** | En la ruta `/user/{id}` | Después de `?` en query string |
| **Sintaxis URL** | `/user/123` | `/search?q=spring` |
| **Obligatorio** | Sí (por defecto) | Configurable con `required` |
| **Valor default** | No aplica | `defaultValue = "valor"` |
| **Uso típico** | Identificadores de recursos | Filtros, paginación, ordenamiento |

---

## 4. Explicación Detallada

### ¿Qué son los parámetros dinámicos?

Los parámetros dinámicos permiten que tus endpoints REST sean flexibles y reutilizables. En lugar de crear un endpoint específico para cada caso, puedes capturar valores variables que los clientes envían en la petición.

### Path Variables (@PathVariable)

**Uso:** Identificar recursos específicos en la jerarquía de la URL.

```java
@GetMapping("/users/{userId}/orders/{orderId}")
public String getOrder(@PathVariable Long userId, @PathVariable Long orderId) {
    return "Pedido " + orderId + " del usuario " + userId;
}
```

**URL:** `GET /users/42/orders/1001`

**Características:**
- Parte integral de la ruta
- Siempre obligatorios
- Ideal para identificadores de recursos (IDs)
- Contribuyen a URLs RESTful semánticas

### Query Parameters (@RequestParam)

**Uso:** Filtrar, ordenar, paginar o configurar la respuesta.

```java
@GetMapping("/products")
public String listProducts(
    @RequestParam(required = false) String category,
    @RequestParam(defaultValue = "name") String sortBy) {
    return "Listando productos...";
}
```

**URLs válidas:**
- `GET /products` (usa defaults)
- `GET /products?category=electronics`
- `GET /products?category=books&sortBy=price`

**Características:**
- Opcionales con `required = false`
- Pueden tener valores por defecto
- Múltiples parámetros con `&`
- Ideal para opciones de búsqueda/filtrado

### Nombres de parámetros personalizados

```java
@GetMapping("/search")
public String search(@RequestParam(name = "q") String query) {
    return "Buscando: " + query;
}
```

**URL:** `GET /search?q=spring+boot`

El parámetro en la URL es `q`, pero en Java se llama `query`.

### Tipos de datos automáticos

Spring Boot convierte automáticamente los parámetros:

```java
@GetMapping("/calculate")
public String calculate(
    @RequestParam int x,
    @RequestParam int y,
    @RequestParam(defaultValue = "true") boolean showSteps) {
    int result = x + y;
    return "Resultado: " + result + " | Mostrar pasos: " + showSteps;
}
```

**URL:** `GET /calculate?x=10&y=20&showSteps=false`

**Tipos soportados:**
- Primitivos: `int`, `long`, `boolean`, `double`
- Wrappers: `Integer`, `Long`, `Boolean`, `Double`
- Strings
- Enums
- Arrays y Lists

---

## 5. Conceptos Aprendidos

### REST API Design

- **Path Variables**: Recursos jerárquicos (`/users/{id}/posts/{postId}`)
- **Query Parameters**: Modificadores de la respuesta (`?sort=date&filter=active`)
- **URLs semánticas**: Fáciles de leer y entender

### Spring MVC

- **@PathVariable**: Extrae valores de la URL path
- **@RequestParam**: Extrae valores del query string
- **Type conversion**: Spring convierte strings a tipos Java automáticamente
- **Default values**: Valores por defecto para parámetros opcionales
- **Required vs Optional**: Control sobre parámetros obligatorios

### Buenas Prácticas

- Path variables para **identificadores** de recursos
- Query params para **filtros, ordenamiento, paginación**
- Usar valores por defecto razonables
- Validar parámetros en la lógica de negocio
- Nombres de parámetros descriptivos

### Convenciones REST

```
GET    /users              - Listar todos los usuarios
GET    /users/{id}         - Obtener usuario específico
GET    /users/{id}/posts   - Posts del usuario
GET    /posts?author={id}  - Posts filtrados por autor
```

---

## 6. Troubleshooting

### Problema 1: "Required request parameter 'name' is not present"

**Error:**
```
Required request parameter 'name' for method parameter type String is not present
```

**Causa:** El parámetro es obligatorio pero no se envió.

**Solución 1:** Agregar valor por defecto
```java
@RequestParam(defaultValue = "Invitado") String name
```

**Solución 2:** Hacerlo opcional
```java
@RequestParam(required = false) String name
```

---

### Problema 2: "Failed to convert value of type 'java.lang.String' to required type 'int'"

**Error:**
```
Failed to convert value of type 'java.lang.String' to required type 'int'
```

**Causa:** Enviaste un texto cuando se esperaba un número.

**URL problemática:** `GET /calculate?x=diez&y=20`

**Solución:** Enviar valores numéricos válidos
```bash
curl "http://localhost:8080/calculate?x=10&y=20"
```

**Mejor práctica:** Validar y manejar errores (verás esto en Clase 3)

---

### Problema 3: Path variable no captura correctamente

**Problema:** `GET /hello/Juan Carlos` retorna 404

**Causa:** El espacio en la URL no está codificado.

**Solución:** Codificar espacios como `%20` o `+`
```bash
curl http://localhost:8080/api/hello/Juan%20Carlos
```

---

### Problema 4: Múltiples valores para el mismo parámetro

**Pregunta:** ¿Cómo envío múltiples valores? Ej: `?tags=java&tags=spring&tags=boot`

**Solución:** Usar una lista

```java
@GetMapping("/filter")
public String filter(@RequestParam List<String> tags) {
    return "Tags: " + String.join(", ", tags);
}
```

**URL:** `GET /filter?tags=java&tags=spring&tags=boot`

---

### Problema 5: Nombre de parámetro con guión

**Problema:** `@RequestParam String sort-by` causa error de sintaxis

**Causa:** Los guiones no son válidos en nombres de variables Java.

**Solución:** Usar el atributo `name`
```java
@GetMapping("/list")
public String list(@RequestParam(name = "sort-by") String sortBy) {
    return "Ordenar por: " + sortBy;
}
```

**URL:** `GET /list?sort-by=price`

---

## 7. Desafío Adicional

### Desafío 1: Calculadora REST

Crea un endpoint que funcione como calculadora:

```java
@GetMapping("/calc/{operation}")
public String calculate(
    @PathVariable String operation,
    @RequestParam double a,
    @RequestParam double b) {

    double result = switch (operation) {
        case "sum" -> a + b;
        case "subtract" -> a - b;
        case "multiply" -> a * b;
        case "divide" -> b != 0 ? a / b : Double.NaN;
        default -> 0;
    };

    return operation + "(" + a + ", " + b + ") = " + result;
}
```

**Pruebas:**
```bash
curl "http://localhost:8080/api/calc/sum?a=10&b=5"
curl "http://localhost:8080/api/calc/multiply?a=7&b=8"
curl "http://localhost:8080/api/calc/divide?a=100&b=4"
```

---

### Desafío 2: Buscador de libros

Crea un endpoint que simule búsqueda de libros con múltiples filtros:

```java
@GetMapping("/books")
public String searchBooks(
    @RequestParam(required = false) String title,
    @RequestParam(required = false) String author,
    @RequestParam(required = false) Integer year,
    @RequestParam(defaultValue = "title") String sortBy,
    @RequestParam(defaultValue = "1") int page) {

    StringBuilder result = new StringBuilder("Buscando libros: ");

    if (title != null) result.append("Título='" + title + "' ");
    if (author != null) result.append("Autor='" + author + "' ");
    if (year != null) result.append("Año=" + year + " ");

    result.append("| Orden: " + sortBy + " | Página: " + page);

    return result.toString();
}
```

**Pruebas:**
```bash
curl "http://localhost:8080/api/books?title=Spring&author=Craig"
curl "http://localhost:8080/api/books?year=2023&sortBy=author&page=2"
curl "http://localhost:8080/api/books"
```

---

### Desafío 3: Validación de edad

Crea un endpoint que valide si alguien es mayor de edad:

```java
@GetMapping("/validate/age/{age}")
public String validateAge(
    @PathVariable int age,
    @RequestParam(defaultValue = "18") int minimumAge) {

    if (age >= minimumAge) {
        return "Edad válida: " + age + " años (mínimo: " + minimumAge + ")";
    }
    return "Edad insuficiente: " + age + " años (mínimo requerido: " + minimumAge + ")";
}
```

**Pruebas:**
```bash
curl http://localhost:8080/api/validate/age/20
curl http://localhost:8080/api/validate/age/15
curl "http://localhost:8080/api/validate/age/16?minimumAge=16"
```

---

## 8. Recursos Adicionales

### Documentación Oficial

- [Request Mapping](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-requestmapping.html)
- [Handler Methods - Method Arguments](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html)
- [Type Conversion](https://docs.spring.io/spring-framework/reference/core/validation/convert.html)

### Tutoriales

- [Building REST services with Spring](https://spring.io/guides/tutorials/rest/)
- [Request Parameters](https://www.baeldung.com/spring-request-param)

### Próximos Pasos

- **Práctica 3:** CRUD completo con TODO API
- **Concepto nuevo:** Request Body con `@RequestBody` (Práctica 3)
- **Cheatsheet:** [Anotaciones de parámetros](../../cheatsheet.md#parámetros)

---

## Código de Solución

El código completo está disponible en: `solucion/`

**Incluye:**
- Todos los ejemplos funcionales
- Desafíos resueltos
- Casos de prueba adicionales

---

**¡Excelente trabajo!** Ahora dominas los parámetros dinámicos en REST APIs.

[← Práctica 1](../01-hola-mundo/) | [Volver a Clase 1](../../README.md) | [Práctica 3 →](../03-todo-api/)
