# Desafío Rápido - Clase 1

**Dificultad:** Básica

---

## Objetivo

Agregar un nuevo endpoint a tu TODO API que permita filtrar las tareas por estado (completadas o pendientes).

---

## Requisitos

Partiendo de tu implementación de la Práctica 3 (TODO API), agrega:

1. Un nuevo endpoint `GET /tasks/filter`
2. Acepta un parámetro query `?completed=true` o `?completed=false`
3. Retorna solo las tareas que coincidan con ese estado

---

## Ejemplo de Uso

```bash
# Obtener solo tareas completadas
curl http://localhost:8080/tasks/filter?completed=true

# Obtener solo tareas pendientes
curl http://localhost:8080/tasks/filter?completed=false
```

---

## Respuesta Esperada

```json
[
  {
    "id": 1,
    "title": "Aprender Spring Boot",
    "completed": true
  },
  {
    "id": 3,
    "title": "Dominar REST APIs",
    "completed": true
  }
]
```

---

## Pistas

1. Usa `@RequestParam` para capturar el parámetro `completed`
2. En el servicio, filtra la lista con un stream:
   ```java
   return tasks.stream()
       .filter(t -> t.isCompleted() == completed)
       .collect(Collectors.toList());
   ```
3. Asegúrate que el método retorne `List<Task>`

---

## Código de Arranque

```java
@GetMapping("/filter")
public List<Task> filterTasks(@RequestParam boolean completed) {
    // Tu código aquí
}
```

---

## Verificación

Ejecuta estos comandos para verificar:

```bash
# 1. Crear algunas tareas
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Tarea 1","completed":true}'

curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Tarea 2","completed":false}'

# 2. Filtrar por completadas
curl http://localhost:8080/tasks/filter?completed=true

# 3. Filtrar por pendientes
curl http://localhost:8080/tasks/filter?completed=false
```

---

## Solución

La solución se encuentra en: `practicas/03-todo-api/solucion/` (con el desafío implementado)

---

[← Volver a Clase 1](../README.md)
