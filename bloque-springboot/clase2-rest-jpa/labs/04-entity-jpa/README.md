# Lab 04 · Entidad y repositorio con Spring Data JPA

## 1. Objetivo

Configurar Spring Data JPA, mapear la entidad `Product` y crear el repositorio que permitirá persistir datos en PostgreSQL.

---

## 2. Comandos a ejecutar

```bash
# 1. Ubicarse en el proyecto
cd ~/workspace/product-service

# 2. Abrir pom.xml con tu editor y agregar Spring Data JPA + PostgreSQL
code pom.xml  # Usa tu editor preferido (nano, vim, IntelliJ, etc.)

# 3. Cambiar a application.yml si aún usas application.properties
rm -f src/main/resources/application.properties
cat <<'EOF' > src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/product_db
    username: product_user
    password: product_password
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        format_sql: true
    show-sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
EOF

# 4. Crear la entidad Product
mkdir -p src/main/java/dev/alefiengo/productservice/model
cat <<'EOF' > src/main/java/dev/alefiengo/productservice/model/Product.java
package dev.alefiengo.productservice.model;

import java.math.BigDecimal;
import java.time.OffsetDateTime;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.persistence.PrePersist;
import jakarta.persistence.PreUpdate;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120)
    private String name;

    @Column(nullable = false, length = 255)
    private String description;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock;

    @Column(name = "created_at", nullable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt;

    @PrePersist
    public void prePersist() {
        OffsetDateTime now = OffsetDateTime.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    @PreUpdate
    public void preUpdate() {
        this.updatedAt = OffsetDateTime.now();
    }

    // Getters y setters (puedes generar con tu IDE)

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }

    public Integer getStock() {
        return stock;
    }

    public void setStock(Integer stock) {
        this.stock = stock;
    }

    public OffsetDateTime getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(OffsetDateTime createdAt) {
        this.createdAt = createdAt;
    }

    public OffsetDateTime getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(OffsetDateTime updatedAt) {
        this.updatedAt = updatedAt;
    }
}
EOF

# 5. Crear el repositorio
mkdir -p src/main/java/dev/alefiengo/productservice/repository
cat <<'EOF' > src/main/java/dev/alefiengo/productservice/repository/ProductRepository.java
package dev.alefiengo.productservice.repository;

import java.util.List;

import org.springframework.data.jpa.repository.JpaRepository;

import dev.alefiengo.productservice.model.Product;

public interface ProductRepository extends JpaRepository<Product, Long> {

    List<Product> findByNameContainingIgnoreCase(String name);
}
EOF

# 6. Verificar el arranque y la creación de la tabla
mvn spring-boot:run
```

---

## 3. Desglose del comando

| Paso | Descripción |
|------|-------------|
| Dependencias agregadas | `spring-boot-starter-data-jpa` provee JPA/Hibernate y `postgresql` actúa como driver JDBC. |
| `application.yml` | Configura la conexión a PostgreSQL, habilita `ddl-auto=update` para generar tablas y muestra SQL en consola. |
| `Product` | Entidad con campos básicos, auditoría de creación/actualización y restricciones (`nullable=false`, longitudes, precisión). |
| `ProductRepository` | Extiende `JpaRepository` para obtener métodos CRUD y agrega consulta derivada `findByNameContainingIgnoreCase`. |
| `mvn spring-boot:run` | Conecta a PostgreSQL, crea la tabla `products` y muestra los SQL de Hibernate. |

---

## 4. Explicación detallada

1. **Dependencias**  
   Agrega dentro del bloque `<dependencies>` de `pom.xml`:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-jpa</artifactId>
   </dependency>
   <dependency>
       <groupId>org.postgresql</groupId>
       <artifactId>postgresql</artifactId>
       <scope>runtime</scope>
   </dependency>
   ```
   Spring Data JPA aporta repositorios automáticos, `EntityManager` y transacciones. El driver PostgreSQL permite que Hibernate se comunique con la base de datos.
2. **Configuración**  
   - `ddl-auto: update` genera o actualiza la tabla según la entidad.  
   - `show-sql: true` imprime las consultas para facilitar el aprendizaje. En producción se recomienda desactivar o ajustar el nivel de log.
3. **Entidad `Product`**  
   Se anotan restricciones que Hibernate traducirá a columnas SQL (`VARCHAR`, `NUMERIC`, `TIMESTAMP WITH TIME ZONE`). Los métodos `@PrePersist` y `@PreUpdate` mantienen la auditoría básica.
4. **Repositorio**  
   `JpaRepository` ofrece métodos como `findAll`, `findById`, `save`, `deleteById`. La consulta `findByNameContainingIgnoreCase` evita escribir JPQL manual al construir la cláusula `LIKE` automáticamente.
5. **Verificación**  
   Con la aplicación corriendo, ejecuta en otra terminal:
   ```bash
   docker compose exec postgres psql -U product_user -d product_db -c "\d products"
   ```
   Deberías ver la tabla `products` con las columnas definidas.

---

## 5. Conceptos aprendidos

- Configuración mínima de Spring Data JPA con PostgreSQL.
- Mapeo de entidades JPA y anotaciones básicas de columna.
- Ganchos de ciclo de vida (`@PrePersist`, `@PreUpdate`) para auditoría.
- Creación de repositorios con métodos derivados.

---

## 6. Troubleshooting

- **`relation "products" does not exist`**  
  Revisa los logs de arranque. Si Hibernate no creó la tabla, asegúrate de haber eliminado `application.properties` y de que `application.yml` esté en `src/main/resources`.
- **`org.postgresql.util.PSQLException: FATAL: password authentication failed`**  
  Confirma usuario y contraseña en `application.yml` y en `docker-compose.yml`. Reinicia el contenedor si cambiaste credenciales.
- **Errores de tipos numéricos**  
  Usa `BigDecimal` para precios y define `precision/scale` en la anotación `@Column` para evitar redondeos.
- **Migraciones automáticas en producción**  
  `ddl-auto=update` es conveniente para laboratorios, pero en proyectos reales se recomienda manejar migraciones con herramientas como Flyway o Liquibase.

---

## 7. Desafío adicional/final

Agrega índices a la tabla `products` creando una migración manual (archivo `.sql`) con:

```sql
CREATE INDEX idx_products_name ON products (LOWER(name));
```

Ejecuta la sentencia en PostgreSQL y verifica que aparezca con `\d products`.

---

## 8. Recursos adicionales

- [Spring Data JPA Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [PostgreSQL Data Types](https://www.postgresql.org/docs/current/datatype.html)
