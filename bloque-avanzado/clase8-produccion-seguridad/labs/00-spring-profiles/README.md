# Lab 00: Spring Boot Profiles

---

## 1. Objetivo

Configurar perfiles de Spring Boot (dev y prod) para manejar diferentes configuraciones según el entorno, permitiendo ejecutar la aplicación con configuraciones optimizadas para desarrollo o producción.

---

## 2. Comandos a ejecutar

```bash
# Crear archivos de perfil
cd ~/workspace/product-service/src/main/resources

# Crear application-dev.yml
cat > application-dev.yml << 'EOF'
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    dev.alefiengo: DEBUG
    org.springframework.web: DEBUG
EOF

# Crear application-prod.yml
cat > application-prod.yml << 'EOF'
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
  jpa:
    show-sql: false

logging:
  level:
    dev.alefiengo: INFO
    org.springframework.web: WARN
EOF

# Ejecutar con perfil dev
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Ejecutar con perfil prod (requiere variables de entorno)
export DB_HOST=localhost DB_PORT=5432 DB_NAME=ecommerce
java -jar target/product-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod
```

**Salida esperada**:

```
The following 1 profile is active: "dev"
Started ProductServiceApplication in 3.2 seconds (JVM running for 3.8)
```

---

## 3. Desglose del comando

| Elemento | Descripción |
|----------|-------------|
| `-Dspring-boot.run.profiles=dev` | Activa perfil "dev" al ejecutar con Maven |
| `--spring.profiles.active=prod` | Activa perfil "prod" al ejecutar JAR |
| `application-dev.yml` | Configuración específica para desarrollo |
| `application-prod.yml` | Configuración específica para producción |
| `${DB_HOST}` | Variable de entorno en prod (externalizar config) |

---

## 4. Explicación detallada

### 4.1 ¿Qué son los Spring Boot Profiles?

Los **profiles** permiten mantener **múltiples configuraciones** en una misma aplicación y activar la apropiada según el entorno (desarrollo, testing, producción).

### 4.2 Estructura de Archivos

```
src/main/resources/
├── application.yml           # Configuración común a todos los perfiles
├── application-dev.yml       # Sobrescribe valores para desarrollo
└── application-prod.yml      # Sobrescribe valores para producción
```

**Precedencia**: `application-{profile}.yml` sobrescribe `application.yml`

### 4.3 Configuración Dev (application-dev.yml)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce  # BD local
  jpa:
    show-sql: true          # Mostrar queries SQL
    properties:
      hibernate:
        format_sql: true    # Format

ear SQL legible

logging:
  level:
    dev.alefiengo: DEBUG    # Logs detallados
    org.springframework.web: DEBUG
```

**Características**:
- Conecta a base de datos local
- Muestra SQL en consola (debugging)
- Logging nivel DEBUG (verbose)
- Ideal para desarrollo

### 4.4 Configuración Prod (application-prod.yml)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}  # Variables de entorno
  jpa:
    show-sql: false         # NO mostrar SQL

logging:
  level:
    dev.alefiengo: INFO     # Solo INFO
    org.springframework.web: WARN  # Solo WARN
```

**Características**:
- Usa variables de entorno (12-Factor App)
- NO muestra SQL (performance)
- Logging nivel INFO (menos verbose)
- Ideal para producción

### 4.5 Aplicar a Todos los Servicios

Repetir para:
- product-service
- order-service
- inventory-service
- analytics-service

**IMPORTANTE**: Cada servicio debe tener su propia configuración específica.

---

## 5. Conceptos aprendidos

- **Spring Boot Profiles**: Configuraciones por entorno
- **application-{profile}.yml**: Archivos específicos por perfil
- **Precedencia de configuración**: Profile sobrescribe configuración base
- **Perfil dev**: Desarrollo local con logging verbose
- **Perfil prod**: Producción con variables de entorno
- **12-Factor App**: Externalizar configuración según entorno
- **Activación de perfil**: `-Dspring-boot.run.profiles` o `--spring.profiles.active`

---

## 6. Troubleshooting

### Problema 1: Perfil no se activa

**Síntoma**: Sigue usando configuración de `application.yml`.

**Solución**: Verificar que profile está activo en logs:

```
The following 1 profile is active: "dev"
```

Si no aparece, verificar comando:
```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

### Problema 2: Variables de entorno no se resuelven

**Error**: `Could not resolve placeholder 'DB_HOST'`

**Causa**: Variables de entorno no definidas en perfil prod.

**Solución**: Exportar variables antes de ejecutar:

```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=ecommerce
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

### Problema 3: SQL queries no se muestran

**Causa**: `show-sql: false` en perfil activo.

**Solución**: Cambiar a perfil dev:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

---

## 7. Desafío adicional

1. **Crea un perfil "test"** para testing con base de datos H2 en memoria:

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
```

2. **Agrega property específico por perfil** en código:

```java
@Value("${app.environment:unknown}")
private String environment;

@GetMapping("/info")
public String info() {
    return "Running in: " + environment;
}
```

En `application-dev.yml`:
```yaml
app:
  environment: Development
```

En `application-prod.yml`:
```yaml
app:
  environment: Production
```

---

## 8. Recursos adicionales

- [Spring Boot Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)
- [Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
- [12-Factor App - Config](https://12factor.net/config)

---

**Siguiente**: [Lab 01: Spring Boot Actuator](../01-actuator/README.md)
