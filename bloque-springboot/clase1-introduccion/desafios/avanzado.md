# Desafío Avanzado - Clase 1

**Dificultad:** Avanzada
**Modalidad:** Para Casa (Extra/Opcional)

---

## Objetivo

Implementar una TODO API completa con paginación, ordenamiento, validación avanzada y manejo profesional de errores, aplicando patrones de diseño y mejores prácticas de Spring Boot.

---

## Contexto

Este desafío te prepara para el nivel profesional que veremos en las próximas clases. Integra múltiples conceptos:
- Paginación y ordenamiento
- DTOs (Data Transfer Objects)
- Validación con Bean Validation
- Manejo global de excepciones
- Respuestas HTTP estandarizadas
- Documentación con comentarios Javadoc

---

## Requisitos Funcionales

### 1. Paginación y Ordenamiento

```
GET /tasks?page=0&size=10&sort=title,asc
```

**Parámetros:**
- `page` (default: 0) - Número de página
- `size` (default: 10) - Tamaños de página
- `sort` (default: id,asc) - Campo y dirección (asc/desc)

**Respuesta:**
```json
{
  "content": [
    {
      "id": 1,
      "title": "Aprender Spring Boot",
      "description": "Completar el curso de Spring Boot",
      "completed": false,
      "priority": "HIGH",
      "createdAt": "2025-10-20T10:30:00",
      "dueDate": "2025-10-25T23:59:59"
    }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 25,
  "totalPages": 3,
  "last": false
}
```

---

### 2. Modelo Extendido

Agrega nuevos campos al modelo `Task`:

```java
public class Task {
    private Long id;
    private String title;
    private String description;
    private boolean completed;
    private Priority priority;        // Enum: LOW, MEDIUM, HIGH
    private LocalDateTime createdAt;
    private LocalDateTime dueDate;
    private List<String> tags;        // Etiquetas
}
```

---

### 3. Validación Avanzada

Implementa validaciones con Bean Validation:

```java
public class TaskCreateRequest {
    @NotBlank(message = "El título es obligatorio")
    @Size(min = 3, max = 100, message = "El título debe tener entre 3 y 100 caracteres")
    private String title;

    @Size(max = 500, message = "La descripción no puede exceder 500 caracteres")
    private String description;

    @NotNull(message = "La prioridad es obligatoria")
    private Priority priority;

    @Future(message = "La fecha de vencimiento debe ser futura")
    private LocalDateTime dueDate;

    @Size(max = 5, message = "Máximo 5 etiquetas permitidas")
    private List<String> tags;
}
```

---

### 4. DTOs Separados

Crea DTOs específicos para cada operación:

- `TaskCreateRequest` - Para crear tareas
- `TaskUpdateRequest` - Para actualizar tareas
- `TaskResponse` - Para respuestas
- `TaskSummaryResponse` - Para listados (sin descripción)

---

### 5. Filtros Avanzados

```
GET /tasks/filter?priority=HIGH&completed=false&tag=urgent&dueBefore=2025-10-30
```

Permite combinar múltiples filtros:
- `priority` - Filtrar por prioridad
- `completed` - Filtrar por estado
- `tag` - Filtrar por etiqueta
- `dueBefore` - Tareas que vencen antes de una fecha

---

### 6. Manejo de Excepciones

Implementa un `@ControllerAdvice` que maneje:

- `TaskNotFoundException` → 404
- `MethodArgumentNotValidException` → 400 con detalles de validación
- `IllegalArgumentException` → 400
- `Exception` → 500

**Formato de respuesta de error:**
```json
{
  "timestamp": "2025-10-20T10:30:00",
  "status": 400,
  "error": "Bad Request",
  "message": "Error de validación",
  "errors": [
    {
      "field": "title",
      "message": "El título es obligatorio"
    },
    {
      "field": "dueDate",
      "message": "La fecha de vencimiento debe ser futura"
    }
  ],
  "path": "/tasks"
}
```

---

## Requisitos Técnicos

### Arquitectura

```
controller/
├── TaskController.java
├── dto/
│   ├── TaskCreateRequest.java
│   ├── TaskUpdateRequest.java
│   ├── TaskResponse.java
│   ├── TaskSummaryResponse.java
│   └── PageResponse.java
└── exception/
    ├── TaskNotFoundException.java
    ├── ErrorResponse.java
    └── GlobalExceptionHandler.java

service/
├── TaskService.java
└── impl/
    └── TaskServiceImpl.java

model/
├── Task.java
└── Priority.java (enum)

util/
└── TaskMapper.java  // Convertir entre Task y DTOs
```

---

## Código de Arranque

### Priority.java

```java
public enum Priority {
    LOW, MEDIUM, HIGH
}
```

### TaskNotFoundException.java

```java
public class TaskNotFoundException extends RuntimeException {
    public TaskNotFoundException(Long id) {
        super("No se encontró la tarea con ID: " + id);
    }
}
```

### GlobalExceptionHandler.java

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(TaskNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleTaskNotFound(TaskNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            ex.getMessage(),
            null
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex) {
        List<FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> new FieldError(error.getField(), error.getDefaultMessage()))
            .collect(Collectors.toList());

        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Bad Request",
            "Error de validación",
            fieldErrors
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
}
```

### TaskMapper.java (Utility)

```java
public class TaskMapper {

    public static TaskResponse toResponse(Task task) {
        return new TaskResponse(
            task.getId(),
            task.getTitle(),
            task.getDescription(),
            task.isCompleted(),
            task.getPriority(),
            task.getCreatedAt(),
            task.getDueDate(),
            task.getTags()
        );
    }

    public static Task toEntity(TaskCreateRequest request) {
        Task task = new Task();
        task.setTitle(request.getTitle());
        task.setDescription(request.getDescription());
        task.setPriority(request.getPriority());
        task.setDueDate(request.getDueDate());
        task.setTags(request.getTags());
        task.setCreatedAt(LocalDateTime.now());
        task.setCompleted(false);
        return task;
    }
}
```

---

## Implementación de Paginación

### PageResponse.java

```java
public class PageResponse<T> {
    private List<T> content;
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
    private boolean last;

    // Constructor, getters, setters
}
```

### TaskService.java

```java
public PageResponse<TaskSummaryResponse> getTasks(
        int page,
        int size,
        String sortBy,
        String direction) {

    // Validar parámetros
    if (page < 0) page = 0;
    if (size < 1 || size > 100) size = 10;

    // Ordenar
    List<Task> sorted = new ArrayList<>(tasks);
    Comparator<Task> comparator = getComparator(sortBy, direction);
    sorted.sort(comparator);

    // Paginar
    int start = page * size;
    int end = Math.min(start + size, sorted.size());

    List<Task> pageContent = start < sorted.size()
        ? sorted.subList(start, end)
        : Collections.emptyList();

    // Convertir a DTOs
    List<TaskSummaryResponse> summaries = pageContent.stream()
        .map(TaskMapper::toSummary)
        .collect(Collectors.toList());

    // Construir respuesta
    return new PageResponse<>(
        summaries,
        page,
        size,
        sorted.size(),
        (int) Math.ceil((double) sorted.size() / size),
        end >= sorted.size()
    );
}

private Comparator<Task> getComparator(String sortBy, String direction) {
    Comparator<Task> comparator = switch (sortBy) {
        case "title" -> Comparator.comparing(Task::getTitle);
        case "priority" -> Comparator.comparing(Task::getPriority);
        case "createdAt" -> Comparator.comparing(Task::getCreatedAt);
        case "dueDate" -> Comparator.comparing(Task::getDueDate,
            Comparator.nullsLast(Comparator.naturalOrder()));
        default -> Comparator.comparing(Task::getId);
    };

    return direction.equalsIgnoreCase("desc")
        ? comparator.reversed()
        : comparator;
}
```

---

## Pruebas Completas

```bash
# 1. Crear tareas con validación
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Aprender Spring Boot",
    "description": "Completar el curso",
    "priority": "HIGH",
    "dueDate": "2025-10-30T23:59:59",
    "tags": ["spring", "java"]
  }'

# 2. Probar validación (título muy corto)
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Ab", "priority": "LOW"}'

# 3. Obtener con paginación
curl "http://localhost:8080/tasks?page=0&size=5&sort=priority,desc"

# 4. Filtrar por prioridad y estado
curl "http://localhost:8080/tasks/filter?priority=HIGH&completed=false"

# 5. Filtrar por tag
curl "http://localhost:8080/tasks/filter?tag=spring"

# 6. Tareas que vencen antes de fecha
curl "http://localhost:8080/tasks/filter?dueBefore=2025-10-30"

# 7. Probar error 404
curl http://localhost:8080/tasks/999
```

---

## Criterios de Éxito

- [ ] Paginación funciona correctamente con todos los parámetros
- [ ] Ordenamiento soporta múltiples campos (title, priority, createdAt, dueDate)
- [ ] Validaciones funcionan y retornan errores descriptivos
- [ ] DTOs separados para request/response
- [ ] Manejo de excepciones global con formato estandarizado
- [ ] Filtros avanzados funcionan individual y combinados
- [ ] Modelo extendido con todos los campos nuevos
- [ ] Código documentado con Javadoc
- [ ] Arquitectura en capas clara
- [ ] Mapper utility para conversiones

---

## Bonus Extra

Si quieres ir más allá:

1. **Exportación**: `GET /tasks/export?format=json` (retorna todas las tareas)
2. **Importación**: `POST /tasks/import` (carga tareas desde JSON)
3. **Búsqueda full-text**: Buscar en título y descripción simultáneamente
4. **Subtareas**: Agregar relación uno-a-muchos (Task → SubTask)
5. **Usuarios**: Agregar campo `assignedTo` para asignar tareas

---

## Recursos de Apoyo

- [Bean Validation Annotations](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-defineconstraints-spec)
- [Spring @ControllerAdvice](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html)
- [ResponseEntity](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseEntity.html)
- [Java Records](https://docs.oracle.com/en/java/javase/17/language/records.html) - Alternativa a DTOs con constructores

---

## Solución Completa

La solución se encuentra en: `practicas/03-todo-api/solucion-desafio-avanzado/`

Incluye:
- Código completo con todos los requisitos
- Tests unitarios para el servicio
- Colección de Postman con todos los endpoints
- Documentación detallada de diseño

---

[← Volver a Clase 1](../README.md)
