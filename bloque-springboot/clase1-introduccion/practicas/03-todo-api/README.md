# Práctica 3: TODO API - CRUD Completo

**Nivel:** Intermedio

---

## 1. Objetivo

Construir una REST API completa con operaciones CRUD (Create, Read, Update, Delete) para gestionar tareas, usando `@RequestBody` para recibir datos JSON y almacenamiento en memoria con `ArrayList`.

**Al finalizar esta práctica podrás:**
- Crear un modelo de datos con clase Java
- Implementar endpoints para todas las operaciones CRUD
- Recibir datos JSON con `@RequestBody`
- Usar diferentes métodos HTTP (GET, POST, PUT, DELETE)
- Gestionar colecciones con `ArrayList`
- Retornar códigos HTTP apropiados con `ResponseEntity`

---

## 2. Comandos a Ejecutar

### Paso 1: Crear el proyecto

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,devtools \
  -d groupId=dev.alefiengo \
  -d artifactId=todo-api \
  -d name=TodoApi \
  -d packageName=dev.alefiengo.todoapi \
  -d javaVersion=17 \
  -o todo-api.zip

unzip todo-api.zip -d todo-api
cd todo-api
```

### Paso 2: Crear el modelo Task

**Archivo:** `src/main/java/dev/alefiengo/todoapi/model/Task.java`

```java
package dev.alefiengo.todoapi.model;

public class Task {
    private Long id;
    private String title;
    private boolean completed;

    // Constructor vacío (requerido para JSON deserialización)
    public Task() {
    }

    // Constructor con parámetros
    public Task(Long id, String title, boolean completed) {
        this.id = id;
        this.title = title;
        this.completed = completed;
    }

    // Getters y Setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public boolean isCompleted() {
        return completed;
    }

    public void setCompleted(boolean completed) {
        this.completed = completed;
    }
}
```

### Paso 3: Crear el controlador

**Archivo:** `src/main/java/dev/alefiengo/todoapi/controller/TaskController.java`

```java
package dev.alefiengo.todoapi.controller;

import dev.alefiengo.todoapi.model.Task;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

@RestController
@RequestMapping("/api/tasks")
public class TaskController {

    private final List<Task> tasks = new ArrayList<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    // Constructor - inicializa con datos de ejemplo
    public TaskController() {
        tasks.add(new Task(idGenerator.getAndIncrement(), "Aprender Spring Boot", false));
        tasks.add(new Task(idGenerator.getAndIncrement(), "Crear REST API", false));
        tasks.add(new Task(idGenerator.getAndIncrement(), "Dominar Kafka", false));
    }

    // CREATE - POST /api/tasks
    @PostMapping
    public ResponseEntity<Task> createTask(@RequestBody Task task) {
        task.setId(idGenerator.getAndIncrement());
        tasks.add(task);
        return ResponseEntity.status(HttpStatus.CREATED).body(task);
    }

    // READ ALL - GET /api/tasks
    @GetMapping
    public List<Task> getAllTasks() {
        return tasks;
    }

    // READ ONE - GET /api/tasks/{id}
    @GetMapping("/{id}")
    public ResponseEntity<Task> getTaskById(@PathVariable Long id) {
        return tasks.stream()
                .filter(t -> t.getId().equals(id))
                .findFirst()
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    // UPDATE - PUT /api/tasks/{id}
    @PutMapping("/{id}")
    public ResponseEntity<Task> updateTask(@PathVariable Long id, @RequestBody Task updatedTask) {
        for (int i = 0; i < tasks.size(); i++) {
            Task task = tasks.get(i);
            if (task.getId().equals(id)) {
                updatedTask.setId(id);
                tasks.set(i, updatedTask);
                return ResponseEntity.ok(updatedTask);
            }
        }
        return ResponseEntity.notFound().build();
    }

    // DELETE - DELETE /api/tasks/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteTask(@PathVariable Long id) {
        boolean removed = tasks.removeIf(t -> t.getId().equals(id));
        if (removed) {
            return ResponseEntity.noContent().build();
        }
        return ResponseEntity.notFound().build();
    }
}
```

### Paso 4: Ejecutar la aplicación

```bash
mvn spring-boot:run
```

### Paso 5: Probar la API

```bash
# 1. Listar todas las tareas (GET)
curl http://localhost:8080/api/tasks

# 2. Obtener tarea específica (GET)
curl http://localhost:8080/api/tasks/1

# 3. Crear nueva tarea (POST)
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Estudiar Docker","completed":false}'

# 4. Actualizar tarea (PUT)
curl -X PUT http://localhost:8080/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Aprender Spring Boot - Actualizado","completed":true}'

# 5. Eliminar tarea (DELETE)
curl -X DELETE http://localhost:8080/api/tasks/2

# 6. Verificar eliminación
curl http://localhost:8080/api/tasks
```

**Respuestas esperadas:**

```json
// GET /api/tasks
[
  {"id":1,"title":"Aprender Spring Boot","completed":false},
  {"id":2,"title":"Crear REST API","completed":false},
  {"id":3,"title":"Dominar Kafka","completed":false}
]

// POST /api/tasks (Status: 201 Created)
{"id":4,"title":"Estudiar Docker","completed":false}

// PUT /api/tasks/1 (Status: 200 OK)
{"id":1,"title":"Aprender Spring Boot - Actualizado","completed":true}

// DELETE /api/tasks/2 (Status: 204 No Content)
```

---

## 3. Desglose del Comando

### Métodos HTTP y Operaciones CRUD

| Operación | Método HTTP | Endpoint | Request Body | Respuesta |
|-----------|-------------|----------|--------------|-----------|
| **Create** | POST | `/api/tasks` | JSON con task | Task creado (201) |
| **Read All** | GET | `/api/tasks` | Ninguno | Array de tasks (200) |
| **Read One** | GET | `/api/tasks/{id}` | Ninguno | Task (200) o 404 |
| **Update** | PUT | `/api/tasks/{id}` | JSON con task | Task actualizado (200) o 404 |
| **Delete** | DELETE | `/api/tasks/{id}` | Ninguno | 204 (sin contenido) o 404 |

### Anotaciones Nuevas

| Anotación | Propósito | Ejemplo |
|-----------|-----------|---------|
| `@RequestBody` | Deserializa JSON del body a objeto Java | `createTask(@RequestBody Task task)` |
| `@PostMapping` | Mapea peticiones HTTP POST | `@PostMapping` |
| `@PutMapping` | Mapea peticiones HTTP PUT | `@PutMapping("/{id}")` |
| `@DeleteMapping` | Mapea peticiones HTTP DELETE | `@DeleteMapping("/{id}")` |
| `ResponseEntity<T>` | Respuesta con control de status HTTP | `ResponseEntity.ok(task)` |

### Códigos HTTP Usados

| Código | Significado | Cuándo usarlo |
|--------|-------------|---------------|
| 200 OK | Operación exitosa | GET, PUT exitosos |
| 201 Created | Recurso creado | POST exitoso |
| 204 No Content | Operación exitosa sin retorno | DELETE exitoso |
| 404 Not Found | Recurso no encontrado | GET/PUT/DELETE de ID inexistente |

---

## 4. Explicación Detallada

### ¿Qué es CRUD?

**CRUD** = Create, Read, Update, Delete

Las cuatro operaciones básicas de cualquier sistema de gestión de datos:

1. **Create** - Crear nuevos registros
2. **Read** - Leer/consultar registros existentes
3. **Update** - Actualizar registros existentes
4. **Delete** - Eliminar registros

### Modelo de Datos (Task)

```java
public class Task {
    private Long id;              // Identificador único
    private String title;         // Descripción de la tarea
    private boolean completed;    // Estado: completada o pendiente
}
```

**¿Por qué necesitamos constructor vacío?**

Spring Boot usa Jackson para convertir JSON a objetos Java. Jackson necesita un constructor sin parámetros para crear instancias.

### @RequestBody

```java
@PostMapping
public ResponseEntity<Task> createTask(@RequestBody Task task) {
    // task contiene los datos del JSON recibido
}
```

**JSON enviado:**
```json
{"title":"Nueva tarea","completed":false}
```

**Spring automáticamente:**
1. Lee el JSON del body de la petición
2. Crea una instancia de Task usando el constructor vacío
3. Llama a los setters para asignar valores
4. Pasa el objeto task al método

### ResponseEntity

```java
// Retornar 200 OK con el objeto
ResponseEntity.ok(task)

// Retornar 201 Created con el objeto
ResponseEntity.status(HttpStatus.CREATED).body(task)

// Retornar 404 Not Found
ResponseEntity.notFound().build()

// Retornar 204 No Content (sin body)
ResponseEntity.noContent().build()
```

`ResponseEntity` te permite:
- Controlar el código HTTP de respuesta
- Agregar headers
- Incluir o excluir el body

### AtomicLong para IDs

```java
private final AtomicLong idGenerator = new AtomicLong(1);

// Genera IDs únicos incrementales: 1, 2, 3, ...
Long newId = idGenerator.getAndIncrement();
```

**¿Por qué AtomicLong y no Long?**

AtomicLong es thread-safe. Si múltiples peticiones llegan simultáneamente, cada una obtendrá un ID único sin conflictos.

### Stream API para búsqueda

```java
@GetMapping("/{id}")
public ResponseEntity<Task> getTaskById(@PathVariable Long id) {
    return tasks.stream()                         // Convierte lista a stream
            .filter(t -> t.getId().equals(id))    // Filtra por ID
            .findFirst()                          // Obtiene el primero (Optional<Task>)
            .map(ResponseEntity::ok)              // Si existe, 200 OK
            .orElse(ResponseEntity.notFound().build());  // Si no, 404
}
```

**Alternativa sin streams:**
```java
for (Task task : tasks) {
    if (task.getId().equals(id)) {
        return ResponseEntity.ok(task);
    }
}
return ResponseEntity.notFound().build();
```

### Flujo de una petición POST

```
[Cliente] Envía:
POST /api/tasks
Content-Type: application/json
{"title":"Nueva tarea","completed":false}
    |
    v
[Spring MVC DispatcherServlet]
    |
    | Encuentra @PostMapping en TaskController
    v
[Jackson (JSON Processor)]
    |
    | Convierte JSON a objeto Task
    v
[TaskController.createTask()]
    |
    | 1. Genera nuevo ID (4)
    | 2. task.setId(4)
    | 3. tasks.add(task)
    | 4. return ResponseEntity.status(201).body(task)
    v
[Cliente recibe]:
HTTP/1.1 201 Created
Content-Type: application/json
{"id":4,"title":"Nueva tarea","completed":false}
```

---

## 5. Conceptos Aprendidos

### REST API Conventions

- **Recursos en plural**: `/api/tasks` (no `/api/task`)
- **POST para crear**: Retorna 201 Created con el recurso creado
- **PUT para actualizar**: Retorna 200 OK con el recurso actualizado
- **DELETE exitoso**: Retorna 204 No Content (sin body)
- **Recurso no encontrado**: Retorna 404 Not Found

### HTTP Methods (Verbos)

- **GET**: Solo lectura, idempotente, cacheable
- **POST**: Crear recursos, no idempotente
- **PUT**: Actualizar completo, idempotente
- **DELETE**: Eliminar, idempotente

**Idempotente:** Llamar N veces produce el mismo resultado que llamar 1 vez.

### JSON y Java

- `@RequestBody`: JSON → Objeto Java (deserialización)
- Return value: Objeto Java → JSON (serialización)
- Jackson hace la conversión automáticamente

### Almacenamiento en Memoria

```java
private final List<Task> tasks = new ArrayList<>();
```

**Limitaciones:**
- Se pierde al reiniciar la aplicación
- No escalable (una sola instancia)
- Sin persistencia

**Solución:** Base de datos (verás en Clase 2 con Spring Data JPA)

### ResponseEntity vs Objeto directo

```java
// Opción 1: Objeto directo (200 OK siempre)
@GetMapping
public List<Task> getAllTasks() {
    return tasks;
}

// Opción 2: ResponseEntity (control de status)
@GetMapping("/{id}")
public ResponseEntity<Task> getTaskById(@PathVariable Long id) {
    // Puede retornar 200 OK o 404 Not Found
}
```

**Cuándo usar cada uno:**
- Objeto directo: Cuando siempre retornas 200 OK
- ResponseEntity: Cuando necesitas diferentes códigos HTTP

---

## 6. Troubleshooting

### Problema 1: "Content type 'text/plain' not supported"

**Error al hacer POST:**
```
Content type 'text/plain' not supported
```

**Causa:** Falta el header `Content-Type: application/json`

**Solución:**
```bash
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","completed":false}'
```

---

### Problema 2: ID null en tasks creados

**Síntoma:** El JSON retornado tiene `"id":null`

**Causa:** Olvidaste asignar el ID antes de agregar a la lista.

**Verificar:**
```java
@PostMapping
public ResponseEntity<Task> createTask(@RequestBody Task task) {
    task.setId(idGenerator.getAndIncrement());  // ← Esto es crítico
    tasks.add(task);
    return ResponseEntity.status(HttpStatus.CREATED).body(task);
}
```

---

### Problema 3: PUT no actualiza

**Síntoma:** El PUT retorna 200 pero los datos no cambian.

**Causa posible:** No estás reemplazando correctamente en la lista.

**Verificar:**
```java
tasks.set(i, updatedTask);  // ← Reemplaza el elemento en la posición i
```

---

### Problema 4: DELETE retorna 404 pero el ID existe

**Causa:** Comparación incorrecta de IDs.

**Incorrecto:**
```java
tasks.removeIf(t -> t.getId() == id);  // == compara referencias
```

**Correcto:**
```java
tasks.removeIf(t -> t.getId().equals(id));  // .equals() compara valores
```

---

### Problema 5: JSON inválido

**Error:**
```
JSON parse error: Unexpected character
```

**Causas comunes:**
- Comillas simples en lugar de dobles: `{'title':'Test'}` ❌
- Falta coma entre propiedades
- Trailing comma: `{"title":"Test",}` ❌

**Correcto:**
```json
{"title":"Test","completed":false}
```

---

## 7. Desafío Adicional

### Desafío Rápido: Filtrado (5 minutos)

Implementa el endpoint del desafío rápido de la clase:

```java
@GetMapping("/filter")
public List<Task> filterTasks(@RequestParam boolean completed) {
    return tasks.stream()
            .filter(t -> t.isCompleted() == completed)
            .collect(Collectors.toList());
}
```

**Prueba:**
```bash
curl "http://localhost:8080/api/tasks/filter?completed=true"
```

---

### Desafío Intermedio: Búsqueda y Stats (15 minutos)

Ver [desafios/intermedio.md](../../desafios/intermedio.md)

---

### Desafío Avanzado: Paginación, DTOs, Validación (1-2 horas)

Ver [desafios/avanzado.md](../../desafios/avanzado.md)

---

## 8. Recursos Adicionales

### Documentación Oficial

- [Building REST services with Spring](https://spring.io/guides/tutorials/rest/)
- [ResponseEntity](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseEntity.html)
- [Jackson JSON Processor](https://github.com/FasterXML/jackson)

### Testing con Postman

Importa la colección: `../../recursos/postman/clase1-todo-api.json`

### HTTP Methods Reference

- [HTTP Methods - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)
- [HTTP Status Codes](https://httpstatuses.com/)

### Próximos Pasos

- **Práctica 4:** Spring Boot Actuator y configuración
- **Práctica 5:** Refactorizar a arquitectura en capas (Service layer)
- **Clase 2:** Persistencia con Spring Data JPA y PostgreSQL

---

## Código de Solución

El código completo está disponible en: `solucion/`

**Incluye:**
- Implementación completa de CRUD
- Desafío rápido resuelto
- Casos de prueba adicionales
- Colección de Postman

---

**¡Felicidades!** Has creado tu primera REST API CRUD completa.

[← Práctica 2](../02-parametros/) | [Volver a Clase 1](../../README.md) | [Práctica 4 →](../04-actuator/)
