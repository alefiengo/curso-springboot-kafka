# Práctica 5: Refactoring a Arquitectura en Capas

**Nivel:** Intermedio

---

## 1. Objetivo

Refactorizar la TODO API para seguir el patrón de arquitectura en capas, separando responsabilidades entre Controller (presentación), Service (lógica de negocio) y Model (datos), aplicando principios SOLID e inyección de dependencias.

**Al finalizar esta práctica podrás:**
- Separar la lógica de negocio del controlador
- Crear una capa de servicio (Service Layer)
- Aplicar inyección de dependencias por constructor
- Entender el patrón de arquitectura en capas
- Mejorar la testabilidad y mantenibilidad del código

---

## 2. Comandos a Ejecutar

### Paso 1: Estructura actual (Práctica 3)

```
src/main/java/dev/alefiengo/todoapi/
├── TodoApiApplication.java
├── controller/
│   └── TaskController.java       # Controller + Lógica de negocio (mezclado)
└── model/
    └── Task.java
```

**Problema:** Toda la lógica está en el controller.

### Paso 2: Crear la capa de servicio

**Archivo:** `src/main/java/dev/alefiengo/todoapi/service/TaskService.java`

```java
package dev.alefiengo.todoapi.service;

import dev.alefiengo.todoapi.model.Task;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicLong;

@Service
public class TaskService {

    private final List<Task> tasks = new ArrayList<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    // Constructor - inicializa datos de ejemplo
    public TaskService() {
        tasks.add(new Task(idGenerator.getAndIncrement(), "Aprender Spring Boot", false));
        tasks.add(new Task(idGenerator.getAndIncrement(), "Crear REST API", false));
        tasks.add(new Task(idGenerator.getAndIncrement(), "Dominar Kafka", false));
    }

    // Crear tarea
    public Task createTask(Task task) {
        task.setId(idGenerator.getAndIncrement());
        tasks.add(task);
        return task;
    }

    // Obtener todas las tareas
    public List<Task> getAllTasks() {
        return new ArrayList<>(tasks);  // Retorna copia para inmutabilidad
    }

    // Obtener tarea por ID
    public Optional<Task> getTaskById(Long id) {
        return tasks.stream()
                .filter(t -> t.getId().equals(id))
                .findFirst();
    }

    // Actualizar tarea
    public Optional<Task> updateTask(Long id, Task updatedTask) {
        for (int i = 0; i < tasks.size(); i++) {
            Task task = tasks.get(i);
            if (task.getId().equals(id)) {
                updatedTask.setId(id);
                tasks.set(i, updatedTask);
                return Optional.of(updatedTask);
            }
        }
        return Optional.empty();
    }

    // Eliminar tarea
    public boolean deleteTask(Long id) {
        return tasks.removeIf(t -> t.getId().equals(id));
    }

    // Filtrar por estado
    public List<Task> filterByCompleted(boolean completed) {
        return tasks.stream()
                .filter(t -> t.isCompleted() == completed)
                .toList();
    }

    // Contar tareas
    public long countTasks() {
        return tasks.size();
    }

    // Contar tareas completadas
    public long countCompletedTasks() {
        return tasks.stream()
                .filter(Task::isCompleted)
                .count();
    }
}
```

### Paso 3: Refactorizar el controlador

**Archivo:** `src/main/java/dev/alefiengo/todoapi/controller/TaskController.java`

```java
package dev.alefiengo.todoapi.controller;

import dev.alefiengo.todoapi.model.Task;
import dev.alefiengo.todoapi.service.TaskService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/tasks")
public class TaskController {

    private final TaskService taskService;

    // Inyección de dependencias por constructor
    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }

    // CREATE
    @PostMapping
    public ResponseEntity<Task> createTask(@RequestBody Task task) {
        Task createdTask = taskService.createTask(task);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdTask);
    }

    // READ ALL
    @GetMapping
    public List<Task> getAllTasks() {
        return taskService.getAllTasks();
    }

    // READ ONE
    @GetMapping("/{id}")
    public ResponseEntity<Task> getTaskById(@PathVariable Long id) {
        return taskService.getTaskById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    // UPDATE
    @PutMapping("/{id}")
    public ResponseEntity<Task> updateTask(@PathVariable Long id, @RequestBody Task task) {
        return taskService.updateTask(id, task)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    // DELETE
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteTask(@PathVariable Long id) {
        if (taskService.deleteTask(id)) {
            return ResponseEntity.noContent().build();
        }
        return ResponseEntity.notFound().build();
    }

    // FILTRAR
    @GetMapping("/filter")
    public List<Task> filterTasks(@RequestParam boolean completed) {
        return taskService.filterByCompleted(completed);
    }

    // ESTADÍSTICAS
    @GetMapping("/stats")
    public TaskStats getStats() {
        long total = taskService.countTasks();
        long completed = taskService.countCompletedTasks();
        long pending = total - completed;
        double completionRate = total > 0 ? (completed * 100.0) / total : 0.0;

        return new TaskStats(total, completed, pending, completionRate);
    }

    // Clase interna para estadísticas
    public static class TaskStats {
        private final long total;
        private final long completed;
        private final long pending;
        private final double completionRate;

        public TaskStats(long total, long completed, long pending, double completionRate) {
            this.total = total;
            this.completed = completed;
            this.pending = pending;
            this.completionRate = completionRate;
        }

        public long getTotal() { return total; }
        public long getCompleted() { return completed; }
        public long getPending() { return pending; }
        public double getCompletionRate() { return completionRate; }
    }
}
```

### Paso 4: Estructura final

```
src/main/java/dev/alefiengo/todoapi/
├── TodoApiApplication.java
├── controller/
│   └── TaskController.java       # Solo manejo de HTTP (✅)
├── service/
│   └── TaskService.java          # Lógica de negocio (✅)
└── model/
    └── Task.java                 # Modelo de datos (✅)
```

### Paso 5: Ejecutar y probar

```bash
mvn spring-boot:run
```

```bash
# Los mismos endpoints funcionan igual
curl http://localhost:8080/api/tasks
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Nueva tarea","completed":false}'

# Nuevo endpoint de estadísticas
curl http://localhost:8080/api/tasks/stats
```

**Respuesta de /stats:**
```json
{
  "total": 4,
  "completed": 1,
  "pending": 3,
  "completionRate": 25.0
}
```

---

## 3. Desglose del Comando

### Arquitectura en Capas

| Capa | Responsabilidad | Anotación | Ejemplo |
|------|-----------------|-----------|---------|
| **Controller** | Manejo de HTTP (request/response) | `@RestController` | TaskController |
| **Service** | Lógica de negocio | `@Service` | TaskService |
| **Model** | Representación de datos | Ninguna | Task |
| **Repository** | Acceso a datos (verás en Clase 2) | `@Repository` | TaskRepository |

### Inyección de Dependencias

| Tipo | Código | Ventajas |
|------|--------|----------|
| **Constructor** (recomendado) | `public TaskController(TaskService service)` | Inmutable, testable, explícito |
| **Field** (no recomendado) | `@Autowired private TaskService service;` | No testable, mutación oculta |
| **Setter** (ocasional) | `@Autowired public void setService(...)` | Dependencias opcionales |

### Optional API

| Método | Descripción | Uso |
|--------|-------------|-----|
| `Optional.of(value)` | Crea Optional con valor (no null) | Cuando sabes que no es null |
| `Optional.empty()` | Crea Optional vacío | Cuando no hay valor |
| `optional.map(fn)` | Transforma el valor si existe | `.map(ResponseEntity::ok)` |
| `optional.orElse(value)` | Valor por defecto si vacío | `.orElse(ResponseEntity.notFound())` |

---

## 4. Explicación Detallada

### ¿Por qué Arquitectura en Capas?

**Problema sin capas:**
```java
@RestController
public class TaskController {
    private List<Task> tasks = new ArrayList<>();

    @PostMapping("/tasks")
    public Task create(@RequestBody Task task) {
        // HTTP handling
        // Validación
        // Lógica de negocio
        // Persistencia
        // Logging
        // Todo mezclado ❌
    }
}
```

**Solución con capas:**
```java
@RestController
public class TaskController {
    private final TaskService taskService;

    @PostMapping("/tasks")
    public ResponseEntity<Task> create(@RequestBody Task task) {
        Task created = taskService.createTask(task);  // Delega
        return ResponseEntity.status(201).body(created);
    }
}

@Service
public class TaskService {
    public Task createTask(Task task) {
        // Solo lógica de negocio
        // Validaciones
        // Cálculos
    }
}
```

### Principios SOLID aplicados

**S - Single Responsibility Principle**
- Controller: Solo maneja HTTP
- Service: Solo lógica de negocio
- Cada clase tiene una única razón para cambiar

**D - Dependency Inversion Principle**
- Controller depende de Service (abstracción)
- No depende de implementación concreta

### Inyección de Dependencias por Constructor

```java
@RestController
public class TaskController {
    private final TaskService taskService;

    // Constructor - Spring inyecta automáticamente
    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }
}
```

**¿Cómo funciona?**

1. Spring Boot encuentra `@Service` en TaskService
2. Crea una instancia de TaskService (singleton)
3. Ve que TaskController necesita TaskService en el constructor
4. Inyecta automáticamente la instancia
5. TaskController está listo para usar

**Ventajas:**
- `final` garantiza inmutabilidad
- Fácil de testear (puedes pasar mock en constructor)
- Dependencias explícitas y obligatorias

### Optional para manejo de ausencia

**Antes (con nulls):**
```java
public Task getTaskById(Long id) {
    for (Task task : tasks) {
        if (task.getId().equals(id)) {
            return task;
        }
    }
    return null;  // Propenso a NullPointerException
}
```

**Después (con Optional):**
```java
public Optional<Task> getTaskById(Long id) {
    return tasks.stream()
            .filter(t -> t.getId().equals(id))
            .findFirst();  // Retorna Optional<Task>
}

// En el controller
return taskService.getTaskById(id)
        .map(ResponseEntity::ok)           // Si existe: 200 OK
        .orElse(ResponseEntity.notFound().build());  // Si no: 404
```

**Beneficios:**
- Explicita que el valor puede no existir
- Previene NullPointerException
- API fluida con `map`, `filter`, `orElse`

### Flujo completo de una petición

```
[Cliente] POST /api/tasks + JSON
    |
    v
[DispatcherServlet]
    |
    | Encuentra @PostMapping en TaskController
    v
[TaskController.createTask()]
    |
    | 1. Recibe Task deserializado
    | 2. Llama taskService.createTask(task)
    v
[TaskService.createTask()]
    |
    | 1. Genera ID
    | 2. Agrega a lista
    | 3. Retorna task con ID
    v
[TaskController]
    |
    | Crea ResponseEntity con status 201
    v
[Cliente recibe] 201 Created + JSON
```

### Testabilidad mejorada

**Antes (sin service layer):**
```java
@Test
public void testCreateTask() {
    // Difícil: necesitas MockMvc, servidor, etc.
}
```

**Después (con service layer):**
```java
@Test
public void testCreateTask() {
    // Fácil: solo pruebas lógica
    TaskService service = new TaskService();
    Task task = new Task(null, "Test", false);

    Task created = service.createTask(task);

    assertNotNull(created.getId());
    assertEquals("Test", created.getTitle());
}
```

---

## 5. Conceptos Aprendidos

### Arquitectura en Capas (Layered Architecture)

```
┌─────────────────────────┐
│   Presentation Layer    │  @RestController
│   (Controller)          │  HTTP, JSON, validación
├─────────────────────────┤
│   Business Logic Layer  │  @Service
│   (Service)             │  Lógica de negocio, reglas
├─────────────────────────┤
│   Data Access Layer     │  @Repository (Clase 2)
│   (Repository)          │  Persistencia, queries
├─────────────────────────┤
│   Domain Model          │  POJOs
│   (Model/Entity)        │  Task, User, Order, etc.
└─────────────────────────┘
```

### Inversión de Control (IoC)

Spring Boot maneja la creación de objetos (beans):

```java
// Tú NO haces esto:
TaskService service = new TaskService();
TaskController controller = new TaskController(service);

// Spring Boot lo hace por ti:
// 1. Encuentra @Service y @RestController
// 2. Crea instancias
// 3. Inyecta dependencias automáticamente
```

### Dependency Injection (DI)

Tres formas de inyección:

```java
// 1. Constructor (recomendado)
public TaskController(TaskService service) {
    this.service = service;
}

// 2. Field (no recomendado)
@Autowired
private TaskService service;

// 3. Setter (ocasional)
@Autowired
public void setService(TaskService service) {
    this.service = service;
}
```

### Bean Scopes

```java
@Service  // Singleton por defecto (una sola instancia)
public class TaskService { }

@Service
@Scope("prototype")  // Nueva instancia cada vez
public class ReportService { }

@Service
@Scope("request")  // Una instancia por HTTP request
public class RequestScopedService { }
```

### Separación de Responsabilidades

| Capa | Lo que DEBE hacer | Lo que NO debe hacer |
|------|-------------------|----------------------|
| **Controller** | Manejar HTTP, validar input, retornar status | Lógica de negocio, acceso a DB |
| **Service** | Lógica de negocio, validaciones, cálculos | Conocer sobre HTTP o JSON |
| **Repository** | Queries, persistencia | Lógica de negocio |

---

## 6. Troubleshooting

### Problema 1: "Parameter 0 of constructor required a bean"

**Error completo:**
```
Parameter 0 of constructor in TaskController required a bean of type 'TaskService' that could not be found.
```

**Causa:** Falta `@Service` en TaskService.

**Solución:**
```java
@Service  // ← Esto registra TaskService como bean
public class TaskService {
    // ...
}
```

---

### Problema 2: Cambios en service no se reflejan

**Síntoma:** Modificas TaskService pero el controller usa la versión antigua.

**Causa:** Service es singleton, se crea una sola vez.

**Solución:** Reinicia la aplicación (o usa DevTools para hot reload).

---

### Problema 3: "Field service in TaskController required a bean"

**Error con field injection:**
```java
@Autowired
private TaskService service;  // Puede fallar
```

**Solución:** Usa constructor injection:
```java
private final TaskService service;

public TaskController(TaskService service) {
    this.service = service;
}
```

---

### Problema 4: Circular dependency

**Error:**
```
The dependencies of some of the beans form a cycle
```

**Causa:**
```java
@Service
public class ServiceA {
    public ServiceA(ServiceB b) { }
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) { }  // Ciclo
}
```

**Solución:** Rediseña para romper el ciclo (a menudo indica mal diseño).

---

### Problema 5: Tests fallan después de refactor

**Causa:** Tests antiguos esperaban la estructura anterior.

**Solución:** Actualiza tests para usar el service:
```java
@Test
public void testGetAllTasks() {
    TaskService service = new TaskService();
    List<Task> tasks = service.getAllTasks();
    assertEquals(3, tasks.size());
}
```

---

## 7. Desafío Adicional

### Desafío 1: Agregar validaciones en el service

```java
@Service
public class TaskService {

    public Task createTask(Task task) {
        // Validaciones
        if (task.getTitle() == null || task.getTitle().trim().isEmpty()) {
            throw new IllegalArgumentException("El título es obligatorio");
        }

        if (task.getTitle().length() < 3) {
            throw new IllegalArgumentException("El título debe tener al menos 3 caracteres");
        }

        task.setId(idGenerator.getAndIncrement());
        tasks.add(task);
        return task;
    }
}
```

---

### Desafío 2: Crear interface para el service

```java
// Interface
public interface ITaskService {
    Task createTask(Task task);
    List<Task> getAllTasks();
    Optional<Task> getTaskById(Long id);
    Optional<Task> updateTask(Long id, Task task);
    boolean deleteTask(Long id);
}

// Implementación
@Service
public class TaskServiceImpl implements ITaskService {
    // Implementación...
}

// Controller usa la interface
public TaskController(ITaskService taskService) {
    this.taskService = taskService;
}
```

**Ventaja:** Fácil cambiar implementación sin tocar controller.

---

### Desafío 3: Logger

Agrega logging al service:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class TaskService {

    private static final Logger logger = LoggerFactory.getLogger(TaskService.class);

    public Task createTask(Task task) {
        logger.info("Creando tarea: {}", task.getTitle());
        task.setId(idGenerator.getAndIncrement());
        tasks.add(task);
        logger.info("Tarea creada con ID: {}", task.getId());
        return task;
    }
}
```

---

## 8. Recursos Adicionales

### Documentación Oficial

- [Spring IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
- [Component Scanning](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html)

### Patrones de Diseño

- [Layered Architecture](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- [Dependency Injection](https://martinfowler.com/articles/injection.html)
- [Service Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html)

### Tutoriales

- [Spring Dependency Injection](https://www.baeldung.com/spring-dependency-injection)
- [Constructor vs Field Injection](https://www.baeldung.com/constructor-injection-in-spring)

### Próximos Pasos

- **Clase 2:** Repository layer con Spring Data JPA
- **Clase 3:** Exception handling y validación con Bean Validation
- **Clase 7:** Tests unitarios y de integración

---

## Código de Solución

El código completo está disponible en: `solucion/`

**Incluye:**
- Arquitectura completa en capas
- Validaciones en el service
- Logging
- Tests unitarios del service

---

**¡Excelente refactoring!** Tu código ahora es más limpio, testable y mantenible.

[← Práctica 4](../04-actuator/) | [Volver a Clase 1](../../README.md)
