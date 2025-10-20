# Colecciones de Postman - Clase 1

Este directorio contiene colecciones de Postman para probar las prácticas de la Clase 1.

---

## Colecciones Disponibles

### 1. clase1-todo-api.postman_collection.json

Colección completa con todos los endpoints de las 5 prácticas:

- **Práctica 1**: Hola Mundo (2 requests)
- **Práctica 2**: Parámetros (4 requests)
- **Práctica 3**: TODO API CRUD (7 requests)
- **Práctica 4**: Actuator (7 requests)

**Total**: 20 requests organizados por práctica

---

## Cómo Importar en Postman

### Opción 1: Importar desde archivo

1. Abre Postman
2. Click en **Import** (esquina superior izquierda)
3. Selecciona **File**
4. Navega a `recursos/postman/clase1-todo-api.postman_collection.json`
5. Click **Import**

### Opción 2: Arrastrar y soltar

1. Abre Postman
2. Arrastra el archivo `.json` directamente a la ventana de Postman
3. La colección se importa automáticamente

---

## Variables de Entorno

La colección usa la variable `{{base_url}}` que está configurada por defecto como:

```
base_url = http://localhost:8080
```

### Cambiar el puerto

Si tu aplicación corre en otro puerto, puedes:

**Opción 1: Editar la variable en la colección**

1. Click derecho en la colección
2. **Edit**
3. Tab **Variables**
4. Cambiar `base_url` a `http://localhost:9090` (o el puerto que uses)

**Opción 2: Crear un entorno**

1. Click en **Environments** (sidebar izquierdo)
2. Click **+** para crear nuevo entorno
3. Nombre: `Clase 1 - Dev`
4. Agregar variable:
   - Key: `base_url`
   - Value: `http://localhost:8080`
5. Selecciona el entorno en el dropdown (esquina superior derecha)

---

## Orden Sugerido de Ejecución

### Práctica 1: Hola Mundo

```
1. GET Home
2. GET Hello
```

### Práctica 2: Parámetros

```
1. GET Hello con Path Variable
2. GET Greet con Query Param
3. GET User Profile
4. GET Search con Múltiples Params
```

### Práctica 3: TODO API

```
1. GET All Tasks (ver tareas iniciales)
2. POST Create Task (crear nueva tarea)
3. GET All Tasks (verificar creación)
4. GET Task by ID (obtener tarea específica)
5. PUT Update Task (actualizar tarea)
6. GET Filter Tasks (filtrar por completadas)
7. GET Task Stats (estadísticas)
8. DELETE Task (eliminar tarea)
9. GET All Tasks (verificar eliminación)
```

### Práctica 4: Actuator

```
1. GET Actuator Root (ver endpoints disponibles)
2. GET Health (health check)
3. GET Info (información de la app)
4. GET Metrics (listar métricas)
5. GET Specific Metric - JVM Memory
6. GET Specific Metric - HTTP Requests
7. GET Env (solo en perfil dev)
```

---

## Tests Automatizados

Cada request incluye tests básicos para validar respuestas. Puedes ejecutarlos:

1. Selecciona la colección
2. Click en **Run** (▶️)
3. Selecciona los requests a ejecutar
4. Click **Run Clase 1 - TODO API**

Los tests verifican:
- Códigos de estado HTTP correctos
- Estructura de respuesta JSON
- Presencia de campos requeridos

---

## Troubleshooting

### Problema: "Could not send request - Error: connect ECONNREFUSED"

**Causa**: La aplicación no está corriendo.

**Solución**:
```bash
cd tu-proyecto
mvn spring-boot:run
```

---

### Problema: "404 Not Found" en todos los endpoints

**Causa**: Puerto incorrecto o aplicación no corrió correctamente.

**Solución**:
1. Verifica en los logs de la aplicación el puerto:
   ```
   Tomcat started on port(s): 8080 (http)
   ```
2. Actualiza `base_url` en Postman si es necesario

---

### Problema: POST/PUT retorna "415 Unsupported Media Type"

**Causa**: Falta el header `Content-Type: application/json`

**Solución**: Los requests de la colección ya incluyen este header, pero si creas uno nuevo, agrégalo manualmente.

---

### Problema: GET /actuator/env retorna 404

**Causa**: El endpoint no está expuesto (solo debe estar en perfil dev)

**Solución**: Ejecuta la aplicación con perfil dev:
```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

---

## Personalización

### Agregar Autenticación (para clases futuras)

Cuando agregues seguridad en Clase 7:

1. Edit collection
2. Tab **Authorization**
3. Type: **Bearer Token** o **Basic Auth**
4. Configura según tu implementación

### Agregar Variables Adicionales

Ejemplos útiles:

```
task_id = 1
user_name = Juan
query = spring
```

Uso en requests:
```
GET {{base_url}}/api/tasks/{{task_id}}
```

---

## Recursos Adicionales

- [Postman Learning Center](https://learning.postman.com/)
- [Writing Tests in Postman](https://learning.postman.com/docs/writing-scripts/test-scripts/)
- [Variables in Postman](https://learning.postman.com/docs/sending-requests/variables/)

---

[← Volver a Recursos](../) | [← Volver a Clase 1](../../README.md)
