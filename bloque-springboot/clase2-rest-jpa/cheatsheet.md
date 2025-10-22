# Cheatsheet · Clase 2

Referencia rápida de comandos y fragmentos útiles para la clase de REST + JPA.

---

## Maven y ejecución

```bash
# Compilar y ejecutar con Spring Boot
mvn spring-boot:run

# Limpiar y compilar nuevamente
mvn clean compile

# Ejecutar pruebas (si existen)
mvn test
```

---

## Docker Compose (PostgreSQL)

```bash
# Levantar base de datos en segundo plano
docker compose up -d

# Ver servicios activos
docker compose ps

# Consultar logs del contenedor
docker compose logs -f postgres

# Detener y mantener datos
docker compose down

# Detener y borrar volumen
docker compose down -v
```

---

## PostgreSQL

```bash
# Entrar a psql dentro del contenedor
docker compose exec postgres psql -U product_user -d product_db

# Listar tablas
\dt

# Describir estructura de tabla
\d products

# Consultar registros
SELECT * FROM products;
```

---

## Spring Data JPA

- `JpaRepository<Product, Long>`: expone `findAll`, `findById`, `save`, `deleteById`.
- Métodos derivados útiles:
  - `findByNameContainingIgnoreCase(String name)`
  - `findByPriceBetween(BigDecimal min, BigDecimal max)`
- `@Transactional(readOnly = true)` para consultas, `@Transactional` estándar para escritura.

---

## HTTP y pruebas rápidas

```bash
# Crear producto
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Tablet","description":"10 pulgadas","price":349.90,"stock":12}'

# Listar productos
curl http://localhost:8080/api/products

# Buscar por id
curl http://localhost:8080/api/products/1

# Filtrar por nombre
curl "http://localhost:8080/api/products?name=tab"

# Actualizar
curl -X PUT http://localhost:8080/api/products/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Tablet","description":"10 pulgadas 2025","price":329.90,"stock":9}'

# Eliminar
curl -X DELETE http://localhost:8080/api/products/1
```

---

## Glosario técnico

| Término | Descripción |
|---------|-------------|
| Controlador REST | Clase anotada con `@RestController` que expone endpoints HTTP. |
| DTO | Objeto de transferencia de datos usado para separar la API de la capa de persistencia. |
| Repository | Interfaz de Spring Data que agrupa operaciones CRUD y consultas derivadas. |
| Entity | Clase anotada con `@Entity` y mapeada a una tabla de base de datos. |
| Transacción | Unidad atómica de trabajo que garantiza consistencia en la base de datos. |
