# Cheatsheet - Clase 1: Spring Boot Fundamentals

Referencia rápida de comandos, anotaciones y conceptos de la Clase 1.

---

## Comandos Maven Esenciales

```bash
# Crear proyecto con Spring Initializr CLI
curl https://start.spring.io/starter.zip \
  -d dependencies=web,devtools,actuator \
  -d groupId=dev.alefiengo \
  -d artifactId=demo-app \
  -d javaVersion=17 \
  -o demo-app.zip && unzip demo-app.zip

# Compilar y ejecutar
mvn clean compile
mvn spring-boot:run

# Con perfil específico
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Empaquetar y ejecutar JAR
mvn clean package
java -jar target/demo-app-0.0.1-SNAPSHOT.jar

# Skip tests
mvn clean install -DskipTests
```

---

## Anotaciones Principales

### Clase Principal

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### Controllers

```java
@RestController
@RequestMapping("/api")
public class TaskController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello, World!";
    }

    @GetMapping("/tasks")
    public List<Task> getTasks() { }

    @GetMapping("/tasks/{id}")
    public Task getTask(@PathVariable Long id) { }

    @PostMapping("/tasks")
    public Task createTask(@RequestBody Task task) { }

    @PutMapping("/tasks/{id}")
    public Task updateTask(@PathVariable Long id, @RequestBody Task task) { }

    @DeleteMapping("/tasks/{id}")
    public void deleteTask(@PathVariable Long id) { }
}
```

**Anotaciones:**
- `@RestController` - Controlador que retorna datos JSON
- `@RequestMapping` - Prefijo de ruta para todo el controlador
- `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` - Mapeo HTTP
- `@PathVariable` - Captura variables de ruta `/tasks/{id}`
- `@RequestParam` - Captura query parameters `?name=value`
- `@RequestBody` - Deserializa JSON a objeto Java

### Services

```java
@Service
public class TaskService {

    private List<Task> tasks = new ArrayList<>();

    public List<Task> getAllTasks() {
        return tasks;
    }

    public Task createTask(Task task) {
        tasks.add(task);
        return task;
    }
}
```

### Inyección de Dependencias

```java
@RestController
public class TaskController {

    private final TaskService taskService;

    // Constructor injection (recomendado)
    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }
}
```

---

## Modelo (POJO)

```java
public class Task {
    private Long id;
    private String title;
    private boolean completed;

    // Constructor vacío
    public Task() {}

    // Constructor completo
    public Task(Long id, String title, boolean completed) {
        this.id = id;
        this.title = title;
        this.completed = completed;
    }

    // Getters y Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    // ... resto de getters/setters
}
```

**Con Records (Java 14+):**
```java
public record Task(Long id, String title, boolean completed) {}
```

---

## Configuración (application.yml)

```yaml
server:
  port: 8080

spring:
  application:
    name: demo-app

# Actuator
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always

# Info personalizado
info:
  app:
    name: Demo API
    version: 1.0.0
```

---

## Dependencias Maven (pom.xml)

```xml
<dependencies>
    <!-- Spring Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- DevTools (hot reload) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Códigos de Estado HTTP

| Código | Descripción | Uso |
|--------|-------------|-----|
| 200 | OK | GET, PUT exitoso |
| 201 | Created | POST exitoso |
| 204 | No Content | DELETE exitoso |
| 400 | Bad Request | Datos inválidos |
| 404 | Not Found | Recurso no existe |
| 500 | Internal Error | Error del servidor |

### Retornar códigos personalizados

```java
@PostMapping("/tasks")
public ResponseEntity<Task> createTask(@RequestBody Task task) {
    Task created = taskService.createTask(task);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}

@GetMapping("/tasks/{id}")
public ResponseEntity<Task> getTask(@PathVariable Long id) {
    Task task = taskService.getTaskById(id);
    if (task == null) {
        return ResponseEntity.notFound().build();
    }
    return ResponseEntity.ok(task);
}
```

---

## Estructura de Proyecto

```
src/
├── main/
│   ├── java/dev/alefiengo/demoapp/
│   │   ├── DemoAppApplication.java
│   │   ├── controller/
│   │   │   └── TaskController.java
│   │   ├── service/
│   │   │   └── TaskService.java
│   │   └── model/
│   │       └── Task.java
│   └── resources/
│       └── application.yml
└── test/
    └── java/dev/alefiengo/demoapp/
        └── DemoAppApplicationTests.java
```

---

## Actuator Endpoints

```bash
# Health check
curl http://localhost:8080/actuator/health

# Info
curl http://localhost:8080/actuator/info

# Métricas
curl http://localhost:8080/actuator/metrics
curl http://localhost:8080/actuator/metrics/http.server.requests
```

---

## Probar API con curl

```bash
# GET - Listar
curl http://localhost:8080/tasks

# GET - Por ID
curl http://localhost:8080/tasks/1

# POST - Crear
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Aprender Spring Boot","completed":false}'

# PUT - Actualizar
curl -X PUT http://localhost:8080/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"id":1,"title":"Aprender Spring Boot","completed":true}'

# DELETE - Eliminar
curl -X DELETE http://localhost:8080/tasks/1
```

---

## Mejores Prácticas

### Naming Conventions

- **Clases**: `PascalCase` → `TaskController`, `TaskService`
- **Métodos**: `camelCase` → `getAllTasks()`, `createTask()`
- **Constantes**: `UPPER_SNAKE_CASE` → `MAX_TASKS`, `DEFAULT_PORT`
- **Packages**: `lowercase` → `controller`, `service`, `model`

### REST API Design

```bash
# BUENO
GET    /tasks
GET    /tasks/{id}
POST   /tasks
PUT    /tasks/{id}
DELETE /tasks/{id}

# MALO
GET    /getTasks
POST   /createTask
GET    /tasks/delete/{id}
```

### Separación de Responsabilidades

```java
// BUENO
@RestController
public class TaskController {
    private final TaskService service;

    @GetMapping("/tasks")
    public List<Task> getTasks() {
        return service.getAllTasks();  // Delega al servicio
    }
}

// MALO
@RestController
public class TaskController {
    private List<Task> tasks = new ArrayList<>();

    @GetMapping("/tasks")
    public List<Task> getTasks() {
        return tasks;  // Lógica en el controlador
    }
}
```

---

## Troubleshooting Común

### Puerto ya en uso

**Error:**
```
Web server failed to start. Port 8080 was already in use.
```

**Solución:**
```yaml
# application.yml
server:
  port: 8081
```

### Bean no encontrado

**Error:**
```
Parameter 0 of constructor in TaskController required a bean of type 'TaskService' that could not be found.
```

**Solución:** Asegúrate que `TaskService` tenga `@Service`:
```java
@Service
public class TaskService { }
```

### 404 Not Found en endpoint

**Verificar:**
1. Método tiene `@GetMapping`, `@PostMapping`, etc.
2. Clase tiene `@RestController`
3. Ruta es correcta (case-sensitive)
4. Aplicación corriendo en puerto correcto

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Microservicio | Microservice | Aplicación pequeña e independiente |
| Punto final / Endpoint | Endpoint | URL que expone un recurso REST |
| Controlador | Controller | Capa que maneja peticiones HTTP |
| Servicio | Service | Capa de lógica de negocio |
| Modelo | Model | Representación de datos |
| Inyección de dependencias | Dependency Injection | Patrón para desacoplar componentes |
| Anotación | Annotation | Metadato en código Java (`@RestController`) |
| Contenedor | Container | Spring IoC Container (gestor de beans) |
| Bean | Bean | Objeto gestionado por Spring |
| Actuador | Actuator | Herramienta de monitoreo de Spring Boot |
| Perfil | Profile | Configuración por entorno (dev, prod) |
| POJO | POJO | Plain Old Java Object (clase simple) |
| DTO | DTO | Data Transfer Object |
| Arranque | Bootstrap | Inicio de la aplicación |
| Servidor embebido | Embedded Server | Tomcat integrado en Spring Boot |

---

## Recursos Adicionales

- [Spring Boot Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Guides](https://spring.io/guides)
- [Baeldung Spring Boot](https://www.baeldung.com/spring-boot)
- [REST API Design](https://restfulapi.net/)
