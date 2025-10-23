# Tarea · CRUD básico de productos

## Objetivo

Construir un CRUD simple para la entidad `Product` utilizando Spring Data JPA, Docker Compose y pruebas con Postman, consolidando todo lo aprendido en la Clase 2.

---

## Requerimientos funcionales

- `POST /api/products` crea un producto con los campos `name`, `description`, `price`, `stock`.
- `GET /api/products` lista todos los productos y permite filtrar por nombre con `?name=`.
- `GET /api/products/{id}` devuelve un producto por su identificador.
- `PUT /api/products/{id}` actualiza un producto existente (todos los campos).
- `DELETE /api/products/{id}` elimina un producto.

---

## Requerimientos técnicos

- Utiliza el repositorio proporcionado (`ProductRepository`) directamente en el controlador.
- Persiste los datos en PostgreSQL (usa el `docker-compose.yml` de la clase).
- Documenta cómo iniciar y detener la base de datos.
- Incluye en tu README de la tarea los comandos ejecutados y las evidencias (capturas o salidas).

---

## Entregables

1. Repositorio público con:
   - Código fuente actualizado (`ProductController`, `Product` y cualquier ajuste menores).
   - `README.md` dentro de `clase2-rest-jpa/` con pasos, comandos y capturas.
   - Colección Postman o cURLs documentados para probar cada endpoint.
2. Archivo `.env.sample` si utilizas variables de entorno adicionales.

---

## Criterios de evaluación

| Criterio | Peso |
|----------|------|
| CRUD funcionando (GET, POST, PUT, DELETE) | 40% |
| Uso correcto de Spring Data JPA | 20% |
| Documentación (README, evidencias) | 20% |
| Colección Postman o comandos cURL | 10% |
| Buenas prácticas Git (estructura, commits, `.gitignore`) | 10% |

**Total:** 100%

---

## Pasos sugeridos

1. Genera datos de ejemplo usando Postman o cURL.
2. Verifica en PostgreSQL que se persisten correctamente (`SELECT * FROM products;`).
3. Documenta los endpoints con capturas o ejemplos de respuesta JSON.
4. Realiza commits incrementales explicando los cambios (`feat: implementar PUT products`).
5. Empuja tu rama principal (`git push`) y verifica que el repositorio sea público.

---

## Preguntas frecuentes

- **¿Debo generar DTOs?** No, eso se abordará en la Clase 3. Trabaja directamente con la entidad `Product`.
- **¿Debo manejar validaciones avanzadas?** No en esta tarea. Basta con asegurarse de que los datos requeridos se envían correctamente.
- **¿Qué hago si quiero añadir algo extra (paginación, OrderBy, etc.)?** Puedes hacerlo, pero deja claramente documentado qué agregaste y asegúrate de que el CRUD básico funcione primero.

---

Entrega el enlace de tu repositorio en Moodle antes de la Clase 3.
