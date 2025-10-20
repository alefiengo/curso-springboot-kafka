# Desafío Intermedio - Clase 1

**Dificultad:** Intermedia
**Modalidad:** Opcional - Para hacer en clase si hay tiempo

---

## Objetivo

Extender tu TODO API con funcionalidades de búsqueda y estadísticas, aplicando los conceptos de arquitectura en capas.

---

## Requisitos Funcionales

Agrega los siguientes endpoints a tu TODO API:

### 1. Búsqueda por título

```
GET /tasks/search?query=spring
```

Debe buscar tareas cuyo título contenga la palabra clave (case-insensitive).

**Ejemplo:**
```bash
curl http://localhost:8080/tasks/search?query=spring
```

**Respuesta:**
```json
[
  {
    "id": 1,
    "title": "Aprender Spring Boot",
    "completed": false
  },
  {
    "id": 5,
    "title": "Estudiar Spring Data JPA",
    "completed": false
  }
]
```

---

### 2. Contador de tareas por estado

```
GET /tasks/stats
```

Retorna estadísticas sobre las tareas.

**Ejemplo:**
```bash
curl http://localhost:8080/tasks/stats
```

**Respuesta:**
```json
{
  "total": 10,
  "completed": 4,
  "pending": 6,
  "completionRate": 40.0
}
```

---

### 3. Actualización parcial (solo completar/descompletar)

```
PATCH /tasks/{id}/toggle
```

Cambia el estado de `completed` sin necesidad de enviar el objeto completo.

**Ejemplo:**
```bash
curl -X PATCH http://localhost:8080/tasks/1/toggle
```

**Respuesta:**
```json
{
  "id": 1,
  "title": "Aprender Spring Boot",
  "completed": true
}
```

---

## Requisitos Técnicos

1. Implementa toda la lógica en el `TaskService`
2. Los controladores deben ser delgados (solo delegar al servicio)
3. Crea una clase `TaskStats` para las estadísticas
4. Maneja el caso cuando no se encuentra una tarea (retornar 404)

---

## Estructura Sugerida

### TaskStats.java

```java
public class TaskStats {
    private int total;
    private int completed;
    private int pending;
    private double completionRate;

    // Constructor, getters, setters
}
```

### TaskService.java (nuevos métodos)

```java
@Service
public class TaskService {

    public List<Task> searchByTitle(String query) {
        // Implementar búsqueda
    }

    public TaskStats getStatistics() {
        // Calcular estadísticas
    }

    public Task toggleCompleted(Long id) {
        // Cambiar estado
    }
}
```

### TaskController.java (nuevos endpoints)

```java
@GetMapping("/search")
public List<Task> searchTasks(@RequestParam String query) {
    return taskService.searchByTitle(query);
}

@GetMapping("/stats")
public TaskStats getStats() {
    return taskService.getStatistics();
}

@PatchMapping("/{id}/toggle")
public ResponseEntity<Task> toggleTask(@PathVariable Long id) {
    Task task = taskService.toggleCompleted(id);
    if (task == null) {
        return ResponseEntity.notFound().build();
    }
    return ResponseEntity.ok(task);
}
```

---

## Pistas

### Para la búsqueda:

```java
return tasks.stream()
    .filter(t -> t.getTitle().toLowerCase().contains(query.toLowerCase()))
    .collect(Collectors.toList());
```

### Para las estadísticas:

```java
int total = tasks.size();
int completed = (int) tasks.stream().filter(Task::isCompleted).count();
int pending = total - completed;
double rate = total > 0 ? (completed * 100.0) / total : 0.0;
```

### Para el toggle:

```java
Task task = tasks.stream()
    .filter(t -> t.getId().equals(id))
    .findFirst()
    .orElse(null);

if (task != null) {
    task.setCompleted(!task.isCompleted());
}
return task;
```

---

## Pruebas Sugeridas

```bash
# 1. Crear varias tareas
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Aprender Spring Boot","completed":false}'

curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Estudiar Spring Data JPA","completed":false}'

curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Configurar Docker","completed":true}'

# 2. Probar búsqueda
curl http://localhost:8080/tasks/search?query=spring

# 3. Obtener estadísticas
curl http://localhost:8080/tasks/stats

# 4. Cambiar estado de tarea
curl -X PATCH http://localhost:8080/tasks/1/toggle

# 5. Verificar cambio
curl http://localhost:8080/tasks/1
```

---

## Criterios de Éxito

- [ ] Los tres nuevos endpoints funcionan correctamente
- [ ] La búsqueda es case-insensitive
- [ ] Las estadísticas calculan correctamente el porcentaje
- [ ] El toggle cambia el estado sin afectar otros campos
- [ ] Retorna 404 cuando la tarea no existe
- [ ] El código está organizado en capas (controller → service)

---

## Extensión Adicional (Opcional)

Si terminas rápido, agrega:

1. **Ordenamiento**: `GET /tasks?sort=title` o `?sort=completed`
2. **Eliminar completadas**: `DELETE /tasks/completed` (elimina todas las tareas completadas)
3. **Validación**: El query de búsqueda debe tener al menos 3 caracteres

---

## Solución Completa

La solución se encuentra en: `practicas/03-todo-api/solucion-desafio-intermedio/`

---

[← Volver a Clase 1](../README.md)
