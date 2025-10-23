# FAQ · Clase 2

Preguntas frecuentes y soluciones rápidas para la sesión de REST + JPA.

---

## Dependencias y compilación

### `Cannot resolve symbol JpaRepository`
Revisa que agregaste `spring-boot-starter-data-jpa` en `pom.xml`. Ejecuta `mvn dependency:tree` para confirmar. Si el IDE sigue mostrando errores, sincroniza el proyecto Maven (`Reimport` en IntelliJ).

### `ClassNotFoundException: org.postgresql.Driver`
Asegúrate de incluir el driver de PostgreSQL en `pom.xml`:
```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```
Vuelve a compilar con `mvn clean package`.

---

## Conexión a PostgreSQL

### `FATAL: password authentication failed`
Verifica usuario y contraseña tanto en `docker-compose.yml` como en `application.yml`. Si hiciste cambios, ejecuta `docker compose down -v` para recrear el contenedor con las nuevas credenciales.

### `org.postgresql.util.PSQLException: Connection refused`
Confirma que el contenedor está en ejecución (`docker compose ps`). Si usas WSL o Docker Desktop, asegúrate de que el puerto 5432 no esté bloqueado. Cambia el mapeo a `5433:5432` si ya está en uso.

---

## Errores de la API

### `HTTP 400 Bad Request`
El JSON enviado no coincide con los campos de la entidad `Product`. Revisa nombres, tipos y asegúrate de enviar números sin comillas. Usa Postman para validar el payload.

### `HTTP 404 Not Found` al actualizar o eliminar
El `id` indicado no existe. Consulta los productos disponibles con `GET /api/products` antes de ejecutar `PUT` o `DELETE`.

### `No serializer found for class Product`
Asegúrate de que la entidad tenga getters y setters públicos. Jackson necesita acceso a las propiedades para serializarlas correctamente.
---

## Esquema de base de datos

### La tabla `products` no se crea
Asegúrate de haber eliminado `application.properties`. El archivo `application.yml` debe contener:
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```
Reinicia la aplicación después de cualquier cambio de configuración.

### `could not execute statement` al guardar
Revisa las restricciones de la entidad: campos `@Column(nullable = false)` no pueden recibir `null`. Verifica que el JSON contenga todos los valores requeridos.
---

## Docker

### `docker compose` no está disponible
En versiones antiguas de Docker usa `docker-compose`. Si prefieres la nueva sintaxis, actualiza Docker Desktop o instala el plugin `docker-compose-plugin`.

### Los datos se pierden al reiniciar
Asegúrate de no ejecutar `docker compose down -v` a menos que quieras borrar el volumen. Este comando elimina los datos persistentes.

---

## Otros inconvenientes

### `java.lang.OutOfMemoryError: Metaspace`
Limita el número de reinicios seguidos o agrega `-Dspring.jpa.properties.hibernate.globally_quoted_identifiers=true` si usas demasiadas entidades generadas dinámicamente. Para el laboratorio basta con reiniciar la aplicación.

### El IDE no detecta Lombok
Este proyecto no usa Lombok. Implementa manualmente los getters/setters o aprovecha registros (`record`) para DTOs.

---

Si encuentras un error no documentado, anótalo con el mensaje exacto y steps para reproducirlo. Lo revisaremos al inicio de la siguiente sesión.***
