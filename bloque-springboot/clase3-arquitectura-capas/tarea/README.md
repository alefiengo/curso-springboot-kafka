# Tarea · Refuerzo de arquitectura, validaciones y relación 1:N

## Objetivo

Consolidar el `product-service` aplicando las buenas prácticas vistas en clase: capas bien definidas, DTOs validados, manejo uniforme de errores y categorías persistidas.

---

## Requerimientos

1. **Arquitectura en capas completa**
   - Controller sin lógica de negocio.
   - Service + mapper responsables de la conversión entidad ↔ DTO.
   - Repositorios Spring Data organizados por dominio.

2. **Validaciones y mensajes**
   - DTOs de producto y categoría con Bean Validation.
   - Archivo `ValidationMessages.properties` con mensajes legibles.

3. **Manejo de errores**
   - `@RestControllerAdvice` que cubra not found, validaciones y errores de integridad (incluye respuesta `409 Conflict` para categorías duplicadas).
   - Respuesta uniforme (`timestamp`, `status`, `code`, `message`, `path`).

4. **Relación producto–categoría**
   - Endpoints CRUD de categorías.
   - Endpoint `GET /api/categories/{id}/products` con DTOs.
   - Prevención de eliminación de categoría cuando tiene productos asociados (respuesta 400 o 409 con mensaje claro).

5. **Documentación**
   - README del proyecto con instrucciones para ejecutar Docker, crear categorías y probar validaciones.
   - Colección Postman actualizada con endpoints y casos de error.

---

## Entregables

1. Repositorio público con el código y README actualizado.
2. Colección Postman/Bruno (`.json`) con las pruebas.
3. Captura o comando `docker compose ps` mostrando el stack ejecutándose.
4. Breve sección en el README describiendo cómo reaccionan los errores (`404`, `400`, `409`).

---

## Criterios de evaluación

| Criterio | Peso |
|----------|------|
| Arquitectura en capas + DTOs | 30% |
| Validaciones y mensajes personalizados | 20% |
| Manejo global de excepciones | 20% |
| Relación Product–Category funcionando | 20% |
| Documentación y colección Postman | 10% |

**Total:** 100%

---

## Recomendaciones

- Usa migraciones manuales (DROP/CREATE) o reinicia el volumen de PostgreSQL si cambias el modelo.
- Documenta cualquier decisión adicional (por ejemplo, estrategia para borrar categorías).
- Mantén tu rama `main` limpia y crea pull requests si trabajas con un equipo.

---

## Fecha y entrega

Entrega antes de la Clase 4 mediante Moodle adjuntando un archivo `.txt` con:

```
Nombre completo:
Repositorio:
Colección Postman:
Notas relevantes:
```

Mantén el repositorio actualizado; se revisará el historial de commits para evaluar el progreso.
