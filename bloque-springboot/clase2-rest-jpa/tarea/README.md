# Tarea · CRUD Producto–Categoría (1:N)

## Objetivo

Extender el microservicio `product-service` para modelar categorías y relacionarlas con los productos, aplicando todo lo visto en clase (REST, JPA, Docker Compose y pruebas con Postman).

---

## Descripción general

1. Crea la entidad `Category` con los campos mínimos: `id`, `name`, `description`, `createdAt`, `updatedAt`.
2. Relaciona `Product` con `Category` mediante una asociación `@ManyToOne` (cada producto pertenece a una categoría).
3. Implementa controladores y servicios independientes para cada recurso.
4. Expone endpoints para listar, crear, actualizar y eliminar categorías, así como para consultar los productos de una categoría específica.
5. Documenta cómo ejecutar la solución completa (servicio + base de datos) en tu repositorio.

---

## Requerimientos funcionales

- `POST /api/categories` crea una categoría.
- `GET /api/categories` lista todas las categorías.
- `GET /api/categories/{id}` entrega una categoría específica.
- `PUT /api/categories/{id}` actualiza una categoría existente.
- `DELETE /api/categories/{id}` elimina una categoría (validar que no existan productos asociados o manejarlo con una respuesta clara).
- `GET /api/categories/{id}/products` lista los productos pertenecientes a la categoría.
- `POST /api/products` permite asignar la categoría al crear un producto (por `categoryId`).
- `PUT /api/products/{id}` permite actualizar la categoría del producto.

---

## Requerimientos técnicos

- Reutiliza el `docker-compose.yml` de clase; asegúrate de documentar cómo iniciarlo.
- Utiliza DTOs para requests y responses en ambas entidades.
- Maneja códigos HTTP adecuados (`201`, `200`, `204`, `400`, `404`).
- Incluye validaciones básicas (por ejemplo: nombre obligatorio, longitud mínima o máxima, precio mayor a cero).
- Mantén los repositorios en `dev.alefiengo.productservice.repository` y las nuevas clases en paquetes coherentes (`service`, `controller`, `dto`).

---

## Entregables

1. Repositorio público (GitHub/GitLab) con:
   - Código fuente.
   - `README.md` con instrucciones de ejecución, variables y colecciones Postman.
   - Carpeta `docs/` opcional para diagramas o notas adicionales.
2. Colección Postman o equivalente (`.json`) con ejemplos para todos los endpoints.
3. Archivo `.env.sample` si defines variables de entorno adicionales.
4. Checklist marcado en el README explicando qué puntos completaste.

---

## Criterios de evaluación

| Criterio | Peso |
|----------|------|
| Correctitud funcional de endpoints y relación 1:N | 35% |
| Diseño de capas y uso adecuado de DTOs y servicios | 25% |
| Validaciones y manejo de errores con códigos HTTP correctos | 20% |
| Documentación del repositorio y colecciones Postman | 10% |
| Buenas prácticas de Git (estructura, `.gitignore`, commits claros) | 10% |

**Puntaje total:** 100%

---

## Pasos sugeridos

1. Diseña los DTOs para `Category` (request/response).
2. Crea el repositorio `CategoryRepository` y el servicio correspondiente.
3. Implementa los controladores de categorías y actualiza el controlador de productos para aceptar `categoryId`.
4. Agrega consultas personalizadas si necesitas filtrar productos por categoría o nombre.
5. Prueba todos los endpoints con Postman y exporta la colección.
6. Documenta cómo levantar Docker Compose y cómo ejecutar el servicio.

---

## Reglas y recomendaciones

- Trabaja de manera individual. Cita cualquier recurso externo que utilices.
- Mantén el proyecto con Java 17 y Spring Boot 3.x.
- No incluyas credenciales reales; usa variables de entorno o valores ejemplo.
- Usa commits con formato `feat: ...`, `fix: ...`, `docs: ...`, etc.
- Si encuentras problemas, documéntalos en el README y explica cómo los abordarías.

---

## Entrega

Sube a Moodle un documento con:

```
Nombre completo:
Repositorio:
Colección Postman:
Notas adicionales:
```

Incluye también capturas opcionales (por ejemplo, tablas en PostgreSQL o respuestas de Postman) si deseas reforzar la evidencia.

---

¡Éxitos con la tarea! El objetivo es que llegues a la próxima clase con una base sólida para aplicar validaciones, manejo de excepciones y arquitectura en capas.
