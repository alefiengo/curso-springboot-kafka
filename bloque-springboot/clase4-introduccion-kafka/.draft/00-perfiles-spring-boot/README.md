# Lab 00: Perfiles de Spring Boot

> **Nota**: Este laboratorio se verá en la **Clase 8** como parte de la preparación para producción. Si estás siguiendo el curso en orden, continúa con el [Lab 00: Conceptos de Kafka](../labs/00-conceptos-kafka/).

---

## Objetivo

Configurar perfiles de Spring Boot para diferenciar el comportamiento de `product-service` entre desarrollo y producción, permitiendo usar configuraciones específicas por entorno.

---

## Comandos a ejecutar

```bash
# 1. Navegar al directorio del proyecto
cd product-service

# 2. IMPORTANTE: Mantener application.yml existente (configuración base común)
# NO borrar ni modificar application.yml - contiene la configuración compartida
# Los archivos application-dev.yml y application-prod.yml SOBRESCRIBEN propiedades específicas

# 3. Crear y configurar application-dev.yml

cat > src/main/resources/application-dev.yml << 'EOF'
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    dev.alefiengo: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
EOF

# 4. Crear y configurar application-prod.yml

cat > src/main/resources/application-prod.yml << 'EOF'
spring:
  jpa:
    show-sql: false

logging:
  level:
    dev.alefiengo: INFO
    org.springframework.web: WARN
    org.hibernate: WARN
EOF

# 5. Ejecutar con perfil de desarrollo
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# 6. Verificar perfil activo en los logs
# Buscar: "The following 1 profile is active: dev"

# 7. Probar endpoint con perfil dev
curl http://localhost:8080/api/products

# 8. Detener aplicación (Ctrl+C) y ejecutar con perfil prod
mvn spring-boot:run -Dspring-boot.run.profiles=prod

# 9. Observar diferencia en logs (sin SQL visible en prod)
```

**Estructura final de archivos**:

```
src/main/resources/
├── application.yml           # ✅ MANTENER - Configuración base común
├── application-dev.yml       # ✅ NUEVO - Configuración de desarrollo
└── application-prod.yml      # ✅ NUEVO - Configuración de producción
```

**Salida esperada con perfil `dev`**:

```
2025-11-05T10:30:45.123  INFO 12345 --- [main] d.a.productservice.Application
    : The following 1 profile is active: "dev"

2025-11-05T10:30:47.456  INFO 12345 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer
    : Tomcat started on port(s): 8080 (http)

Hibernate: select p1_0.id, p1_0.category_id, p1_0.created_at, ... from product p1_0
```

Notarás que con el perfil `dev`:
- Logs de SQL formateados y visibles
- Nivel de log `DEBUG` para paquete `dev.alefiengo`

**Salida esperada con perfil `prod`**:

```
2025-11-05T10:35:12.789  INFO 12346 --- [main] d.a.productservice.Application
    : The following 1 profile is active: "prod"

2025-11-05T10:35:14.012  INFO 12346 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer
    : Tomcat started on port(s): 8080 (http)
```

Notarás que con el perfil `prod`:
- NO se muestran sentencias SQL
- Logs mínimos, solo nivel `INFO` o superior

---

## Desglose del comando

### Comando: `mvn spring-boot:run -Dspring-boot.run.profiles=dev`

| Componente | Descripción |
|------------|-------------|
| `mvn` | Maven, herramienta de construcción y gestión de dependencias |
| `spring-boot:run` | Plugin de Maven para ejecutar aplicación Spring Boot sin crear JAR |
| `-D` | Define una propiedad del sistema Java |
| `spring-boot.run.profiles=dev` | Activa el perfil `dev` durante la ejecución |

**Alternativas para activar perfiles**:

```bash
# Con JAR compilado
java -jar target/product-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev

# Con variable de entorno (Linux/Mac)
export SPRING_PROFILES_ACTIVE=dev
mvn spring-boot:run

# Con variable de entorno (Windows)
set SPRING_PROFILES_ACTIVE=dev
mvn spring-boot:run
```

---

## Explicación detallada

### Paso 1: Estructura de archivos de configuración

Spring Boot permite tener múltiples archivos de configuración siguiendo la convención `application-{profile}.yml`:

```
src/main/resources/
├── application.yml           # Configuración común (todos los perfiles)
├── application-dev.yml       # Configuración específica de desarrollo
└── application-prod.yml      # Configuración específica de producción
```

**Prioridad de carga**:

1. Spring Boot carga primero `application.yml` (configuración base)
2. Luego carga `application-{profile}.yml` según el perfil activo
3. Las propiedades del perfil **sobrescriben** las de `application.yml`

### Paso 2: Configurar application.yml (base común)

**IMPORTANTE**: NO borrar ni modificar el archivo `application.yml` existente. Este archivo contiene la **configuración base común** que se aplica a todos los perfiles.

Abre `src/main/resources/application.yml` y verifica que contenga la configuración común:

```yaml
spring:
  application:
    name: product-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:product_db}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

server:
  port: 8080
```

**¿Qué hace este archivo?**

- Define la conexión a PostgreSQL (común a todos los perfiles)
- Configura el nombre de la aplicación
- Establece `ddl-auto: update` (común a dev y prod)
- Define el puerto 8080 (se mantiene igual en todos los perfiles)

**¿Por qué NO incluir `show-sql` aquí?**

- En desarrollo queremos ver SQL (`show-sql: true`)
- En producción NO queremos ver SQL (`show-sql: false`)
- Por eso esta propiedad va en los archivos de perfil específicos, NO en `application.yml`

### Paso 3: Configurar application-dev.yml (desarrollo)

Crear `src/main/resources/application-dev.yml`:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    dev.alefiengo: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

**Explicación de cada propiedad**:

- `show-sql: true`: Muestra las sentencias SQL ejecutadas por Hibernate
- `format_sql: true`: Formatea las sentencias SQL para legibilidad
- `logging.level.dev.alefiengo: DEBUG`: Nivel DEBUG para nuestro código
- `logging.level.org.hibernate.SQL: DEBUG`: Muestra SQL con Hibernate
- `logging.level.org.hibernate.type.descriptor.sql.BasicBinder: TRACE`: Muestra valores de parámetros SQL

**Ventajas en desarrollo**:

- Ver exactamente qué consultas se ejecutan
- Detectar problemas N+1 de consultas
- Debuggear lógica de negocio con logs detallados
- Identificar errores rápidamente

### Paso 4: Configurar application-prod.yml (producción)

Crear `src/main/resources/application-prod.yml`:

```yaml
spring:
  jpa:
    show-sql: false
    properties:
      hibernate:
        format_sql: false

logging:
  level:
    dev.alefiengo: INFO
    org.springframework.web: WARN
    org.hibernate: WARN
```

**Explicación de cada propiedad**:

- `show-sql: false`: NO mostrar SQL en producción (por seguridad y rendimiento)
- `format_sql: false`: No formatear SQL (innecesario si no se muestra)
- `logging.level.dev.alefiengo: INFO`: Solo logs informativos o superiores
- `logging.level.org.springframework.web: WARN`: Solo warnings de Spring Web
- `logging.level.org.hibernate: WARN`: Solo warnings de Hibernate

**Ventajas en producción**:

- Menor volumen de logs (reduce almacenamiento)
- Mejor rendimiento (menos I/O de logs)
- Mayor seguridad (no exponer consultas SQL)
- Logs enfocados en errores reales

### Paso 5: Activar perfil al ejecutar la aplicación

Existen múltiples formas de activar un perfil:

#### Opción 1: Mediante Maven (desarrollo)

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

#### Opción 2: Mediante JAR (producción)

```bash
# Compilar JAR
mvn clean package -DskipTests

# Ejecutar con perfil prod
java -jar target/product-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod
```

#### Opción 3: Variable de entorno (Docker, Kubernetes)

```bash
export SPRING_PROFILES_ACTIVE=prod
java -jar product-service.jar
```

#### Opción 4: En IDE (IntelliJ IDEA)

1. Run → Edit Configurations
2. En "Active profiles" escribir: `dev`
3. Apply → OK → Run

### Paso 6: Verificar el perfil activo

Al iniciar la aplicación, Spring Boot imprime el perfil activo:

```
2025-11-05T10:30:45.123  INFO 12345 --- [main] d.a.productservice.Application
    : The following 1 profile is active: "dev"
```

También puedes verificarlo mediante código:

```java
@RestController
@RequestMapping("/api/info")
public class InfoController {

    @Value("${spring.profiles.active:default}")
    private String activeProfile;

    @GetMapping("/profile")
    public String getActiveProfile() {
        return "Perfil activo: " + activeProfile;
    }
}
```

### Paso 7: Comparar comportamiento entre perfiles

**Con perfil `dev`**:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev
curl http://localhost:8080/api/products
```

**Logs generados**:

```
2025-11-05T10:30:47.890 DEBUG 12345 --- [nio-8080-exec-1] d.a.p.controller.ProductController
    : Obteniendo todos los productos
Hibernate:
    select
        p1_0.id,
        p1_0.category_id,
        p1_0.created_at,
        p1_0.description,
        p1_0.name,
        p1_0.price,
        p1_0.stock,
        p1_0.updated_at
    from
        product p1_0
2025-11-05T10:30:47.912 TRACE 12345 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder
    : binding parameter [1] as [BIGINT] - [1]
```

**Con perfil `prod`**:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=prod
curl http://localhost:8080/api/products
```

**Logs generados**:

```
2025-11-05T10:35:14.234  INFO 12346 --- [nio-8080-exec-1] d.a.p.controller.ProductController
    : Request GET /api/products completado
```

Notarás la diferencia significativa en la cantidad y detalle de logs.

### Paso 8: Usar múltiples perfiles simultáneamente

Spring Boot permite activar múltiples perfiles separados por coma:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev,local
```

En este caso, Spring Boot cargará:

1. `application.yml`
2. `application-dev.yml`
3. `application-local.yml`

Las propiedades de perfiles posteriores sobrescriben las anteriores.

---

## Conceptos aprendidos

- **Perfiles de Spring Boot**: Mecanismo para configurar comportamiento por entorno
- **Convención de nombres**: `application-{profile}.yml` para configuraciones específicas
- **Prioridad de carga**: Configuración base + configuración de perfil activo
- **Activación de perfiles**: Maven, JAR, variable de entorno, IDE
- **Diferenciación dev/prod**: Logs detallados en desarrollo, logs mínimos en producción
- **Seguridad**: NO mostrar SQL ni datos sensibles en producción
- **Rendimiento**: Reducir logs en producción mejora I/O y almacenamiento
- **Múltiples perfiles**: Posibilidad de activar varios perfiles simultáneamente

---

## Troubleshooting

### Problema 1: No se activa el perfil

**Síntoma**:

```
The following 1 profile is active: "default"
```

**Causa**: No se especificó el perfil correctamente.

**Solución**:

```bash
# Verificar sintaxis correcta
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# NO usar:
mvn spring-boot:run -Dspring.profiles.active=dev  # Incorrecto con Maven
```

Para Maven usar: `-Dspring-boot.run.profiles`
Para JAR usar: `--spring.profiles.active`

### Problema 2: Perfil no encontrado

**Síntoma**:

```
Requested bean is currently in creation: Is there an unresolvable circular reference?
```

**Causa**: Archivo `application-dev.yml` con sintaxis YAML incorrecta.

**Solución**:

- Verificar indentación (usar espacios, NO tabs)
- Validar sintaxis en [https://www.yamllint.com/](https://www.yamllint.com/)
- Revisar estructura:

```yaml
# ✅ CORRECTO
spring:
  jpa:
    show-sql: true

# ❌ INCORRECTO (indentación)
spring:
jpa:
  show-sql: true
```

### Problema 3: Configuración no se sobrescribe

**Síntoma**: Aunque se configura `show-sql: false` en `prod`, las sentencias SQL aparecen.

**Causa**: Propiedad configurada incorrectamente en `application.yml`.

**Solución**:

NO definir `show-sql` en `application.yml`, solo en perfiles específicos:

```yaml
# application.yml - NO incluir show-sql aquí
spring:
  jpa:
    hibernate:
      ddl-auto: update

# application-dev.yml - Aquí sí
spring:
  jpa:
    show-sql: true

# application-prod.yml - Aquí también
spring:
  jpa:
    show-sql: false
```

### Problema 4: Logs duplicados

**Síntoma**: Cada log aparece dos veces en la consola.

**Causa**: Configuración de logging heredada de múltiples fuentes.

**Solución**:

Agregar en `application-prod.yml`:

```yaml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
  level:
    root: WARN
    dev.alefiengo: INFO
```

### Problema 5: Perfil activo en producción incorrecta

**Síntoma**: Aplicación en producción usa perfil `dev`.

**Causa**: Variable de entorno `SPRING_PROFILES_ACTIVE` no configurada.

**Solución**:

En Docker Compose:

```yaml
services:
  product-service:
    image: product-service:latest
    environment:
      SPRING_PROFILES_ACTIVE: prod
```

En Kubernetes:

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "prod"
```

---

## Desafío adicional

### Desafío 1: Crear perfil `test` para pruebas

Crea un perfil `test` con las siguientes características:

- Base de datos H2 en memoria (NO PostgreSQL)
- Logs nivel `WARN` para Spring Framework
- Logs nivel `ERROR` para Hibernate
- `ddl-auto: create-drop` (recrea schema en cada ejecución)

**Archivo**: `src/main/resources/application-test.yml`

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false
  h2:
    console:
      enabled: true

logging:
  level:
    org.springframework: WARN
    org.hibernate: ERROR
```

**Dependencia H2** (agregar a `pom.xml` con `scope:test`):

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

**Ejecutar tests con perfil test**:

```bash
mvn test -Dspring.profiles.active=test
```

### Desafío 2: Diferenciar base de datos por perfil

Modifica los perfiles para usar bases de datos diferentes:

- `dev` → `product_db_dev`
- `prod` → `product_db_prod`

**application-dev.yml**:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/product_db_dev
```

**application-prod.yml**:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/product_db_prod
```

Crear ambas bases de datos:

```bash
docker exec -it postgres psql -U postgres -c "CREATE DATABASE product_db_dev;"
docker exec -it postgres psql -U postgres -c "CREATE DATABASE product_db_prod;"
```

---

## Recursos adicionales

- [Spring Boot Profiles - Documentación oficial](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)
- [Externalizing Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
- [Spring Boot Logging](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)
- [Baeldung - Spring Profiles](https://www.baeldung.com/spring-profiles)
- [12-Factor App - Config](https://12factor.net/config)
