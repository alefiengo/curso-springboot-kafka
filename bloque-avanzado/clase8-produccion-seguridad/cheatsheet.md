# Cheatsheet - Clase 8: Producción, Seguridad y Consolidación

## Spring Boot Profiles

### Crear archivos de perfiles

```yaml
# src/main/resources/application.yml (común)
spring:
  application:
    name: product-service
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

# src/main/resources/application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce_dev
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    dev.alefiengo: DEBUG
    org.springframework.web: DEBUG

# src/main/resources/application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
  jpa:
    show-sql: false

logging:
  level:
    dev.alefiengo: INFO
    org.springframework.web: WARN
```

### Activar perfiles

```bash
# Maven con perfil dev
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Maven con perfil prod
mvn spring-boot:run -Dspring-boot.run.profiles=prod

# JAR con perfil
java -jar target/product-service.jar --spring.profiles.active=prod

# Variable de entorno
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar

# Múltiples perfiles
java -jar app.jar --spring.profiles.active=prod,monitoring
```

---

## Spring Boot Actuator

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Configuración en application.yml

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env
  endpoint:
    health:
      show-details: always
  health:
    kafka:
      enabled: true  # Solo en servicios con Kafka
  info:
    git:
      mode: full
```

### Endpoints útiles

```bash
# Health check
curl http://localhost:8080/actuator/health

# Info de la aplicación
curl http://localhost:8080/actuator/info

# Listar todas las métricas
curl http://localhost:8080/actuator/metrics

# Métrica específica (memoria)
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Variables de entorno
curl http://localhost:8080/actuator/env

# Loggers configurados
curl http://localhost:8080/actuator/loggers
```

### Health check con salida

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "kafka": {
      "status": "UP",
      "details": {
        "clusterId": "...",
        "nodes": 1
      }
    },
    "diskSpace": {
      "status": "UP"
    }
  }
}
```

---

## Variables de Entorno

### Configurar en application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

server:
  port: ${SERVER_PORT:8080}

jwt:
  secret: ${JWT_SECRET:defaultSecretForDevOnly}
  expiration: ${JWT_EXPIRATION:86400000}
```

### Definir variables de entorno

```bash
# Linux/Mac (terminal)
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=ecommerce
export DB_USER=postgres
export DB_PASSWORD=postgres
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export SERVER_PORT=8080
export JWT_SECRET=myVerySecretKeyThatIsAtLeast256BitsLong

# Ejecutar aplicación
mvn spring-boot:run
```

### Archivo .env (NO committear)

```bash
# .env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=postgres
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
SERVER_PORT=8080
JWT_SECRET=myVerySecretKeyThatIsAtLeast256BitsLong
JWT_EXPIRATION=86400000
```

### Docker Compose con variables

```yaml
services:
  product-service:
    image: product-service:latest
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=ecommerce
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - SERVER_PORT=8080
      - JWT_SECRET=${JWT_SECRET}
```

---

## JWT Security

### Dependencias Maven

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

### JwtUtil (generación y validación)

```java
package dev.alefiengo.productservice.security;

import io.jsonwebtoken.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import java.util.Date;

@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private Long expiration;

    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(secret)
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }

    public String getUsernameFromToken(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(secret)
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }
}
```

### SecurityConfig

```java
package dev.alefiengo.productservice.config;

import dev.alefiengo.productservice.security.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Testing JWT

```bash
# 1. Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "admin123"
  }'

# Response:
# {"token":"eyJhbGciOiJIUzUxMiJ9..."}

# 2. Guardar token
TOKEN="eyJhbGciOiJIUzUxMiJ9..."

# 3. Usar token en requests
curl http://localhost:8080/api/products \
  -H "Authorization: Bearer $TOKEN"

# 4. Crear producto con JWT
curl -X POST http://localhost:8080/api/products \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop",
    "description": "High-end laptop",
    "price": 1500.00,
    "categoryId": 1
  }'
```

---

## Logging Estructurado

### Configurar Logback (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- Perfil DEV: Console con formato simple -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>

    <!-- Perfil PROD: File con JSON -->
    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/application.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/application-%d{yyyy-MM-dd}.log</fileNamePattern>
                <maxHistory>30</maxHistory>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="FILE" />
        </root>
    </springProfile>
</configuration>
```

### Logging en código

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public OrderResponse createOrder(OrderRequest request) {
        log.info("Creating order: productId={}, quantity={}, customer={}",
            request.getProductId(),
            request.getQuantity(),
            request.getCustomerName());

        try {
            Order order = repository.save(toEntity(request));
            log.info("Order created successfully: orderId={}", order.getId());
            return toResponse(order);
        } catch (Exception e) {
            log.error("Failed to create order: productId={}, error={}",
                request.getProductId(), e.getMessage(), e);
            throw e;
        }
    }
}
```

---

## Consolidación - Prueba End-to-End

```bash
# 1. Verificar infraestructura
docker compose ps

# 2. Health checks de todos los servicios
curl http://localhost:8080/actuator/health  # product-service
curl http://localhost:8081/actuator/health  # order-service
curl http://localhost:8082/actuator/health  # inventory-service
curl http://localhost:8083/actuator/health  # analytics-service

# 3. Login (obtener JWT)
TOKEN=$(curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' | jq -r '.token')

# 4. Crear producto con JWT
curl -X POST http://localhost:8080/api/products \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop Dell XPS 15",
    "description": "Professional laptop",
    "price": 1500.00,
    "categoryId": 1
  }'

# 5. Crear inventario
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "initialStock": 50
  }'

# 6. Crear orden
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 5,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com",
    "totalAmount": 7500.00
  }'

# 7. Verificar analytics
curl http://localhost:8083/api/analytics/orders/stats
curl http://localhost:8083/api/analytics/products/top

# 8. Ver topics de Kafka
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# 9. Consumir eventos
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning
```

---

## Comandos Docker útiles

```bash
# Ver logs de servicios
docker compose logs -f product-service
docker compose logs -f kafka
docker compose logs -f postgres

# Reiniciar servicio
docker compose restart product-service

# Ver recursos usados
docker stats

# Limpiar sistema
docker system prune -a
```

---

## Generar password BCrypt

```bash
# Usando Spring Boot CLI (si está instalado)
spring encodepassword admin123

# Usando sitio web
# https://bcrypt-generator.com/

# Resultado ejemplo para "admin123":
# $2a$10$N9qo8uLOickgx2ZMRZoMye6lHgl6XUUGpJdCo/IqLSAWFZHvF9vF2
```

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Perfil | Profile | Configuración por entorno |
| Verificación de salud | Health Check | Endpoint de estado |
| Métrica | Metric | Medición del sistema |
| Variable de entorno | Environment Variable | Config externa |
| Token web JSON | JWT | Token de autenticación |
| Secreto | Secret | Información sensible |
| Registro | Logging | Sistema de logs |
| Sin estado | Stateless | No guarda sesiones |
