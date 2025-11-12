# Lab 03: JWT Security

---

## 1. Objetivo

Implementar autenticación y autorización con JSON Web Tokens (JWT) en product-service para proteger los endpoints REST, aprendiendo los fundamentos de seguridad en microservicios.

---

## 2. Comandos a ejecutar

```bash
# Agregar dependencias de seguridad a product-service
cd ~/workspace/product-service

# Editar pom.xml y agregar antes de </dependencies>
cat >> pom.xml << 'EOF'
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
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
EOF

# Agregar configuración JWT en application.yml
cat >> src/main/resources/application.yml << 'EOF'

jwt:
  secret: ${JWT_SECRET:mySecretKeyForDevelopmentOnlyDoNotUseInProductionChangeThis}
  expiration: 86400000  # 24 horas en milisegundos
EOF

# Reconstruir proyecto
mvn clean install

# Ejecutar aplicación
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Probar login (obtener token)
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# Usar token para acceder a productos
TOKEN="eyJhbGciOiJIUzUxMiJ9..."  # Copiar del response anterior
curl http://localhost:8080/api/products \
  -H "Authorization: Bearer $TOKEN"
```

**Salida esperada** (login):

```json
{
  "token": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJhZG1pbiIsImlhdCI6MTYzOTU4MjQwMCwiZXhwIjoxNjM5NjY4ODAwfQ.aBcDeFg..."
}
```

---

## 3. Desglose del comando

| Elemento | Descripción |
|----------|-------------|
| `spring-boot-starter-security` | Framework de seguridad de Spring |
| `jjwt-api` | Librería para crear y validar JWT |
| `jjwt-impl` | Implementación de JJWT |
| `jjwt-jackson` | Serialización JSON para JWT |
| `jwt.secret` | Clave secreta para firmar tokens |
| `jwt.expiration` | Tiempo de vida del token (ms) |
| `Authorization: Bearer <token>` | Header HTTP con token JWT |

---

## 4. Explicación detallada

### 4.1 ¿Qué es JWT?

**JSON Web Token (JWT)** es un estándar abierto (RFC 7519) para transmitir información de forma segura entre partes como un objeto JSON.

**Estructura de JWT**: `header.payload.signature`

```
eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJhZG1pbiIsImlhdCI6MTYzOTU4MjQwMH0.aBcDeFg...
│       Header        │         Payload          │    Signature   │
```

**Header** (Base64):
```json
{
  "alg": "HS512",
  "typ": "JWT"
}
```

**Payload** (Base64):
```json
{
  "sub": "admin",
  "iat": 1639582400,
  "exp": 1639668800
}
```

**Signature**: `HMACSHA512(header + payload, secret)`

### 4.2 Flujo de Autenticación

```
1. Usuario → POST /api/auth/login (username + password)
2. Servidor verifica credenciales
3. Servidor genera JWT firmado
4. Servidor → JWT al usuario
5. Usuario → GET /api/products (Header: Authorization: Bearer <JWT>)
6. Servidor valida JWT
7. Servidor → Datos si JWT válido
```

### 4.3 Crear JwtUtil.java

```bash
mkdir -p src/main/java/dev/alefiengo/productservice/security
```

**src/main/java/dev/alefiengo/productservice/security/JwtUtil.java**:

```java
package dev.alefiengo.productservice.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;

@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private Long expiration;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    public String generateToken(String username) {
        return Jwts.builder()
                .setSubject(username)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey(), SignatureAlgorithm.HS512)
                .compact();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                    .setSigningKey(getSigningKey())
                    .build()
                    .parseClaimsJws(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    public String getUsernameFromToken(String token) {
        Claims claims = Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
        return claims.getSubject();
    }
}
```

### 4.4 Crear JwtAuthenticationFilter.java

**src/main/java/dev/alefiengo/productservice/security/JwtAuthenticationFilter.java**:

```java
package dev.alefiengo.productservice.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.Collections;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);

            if (jwtUtil.validateToken(token)) {
                String username = jwtUtil.getUsernameFromToken(token);

                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(username, null, Collections.emptyList());

                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

### 4.5 Crear SecurityConfig.java

**src/main/java/dev/alefiengo/productservice/config/SecurityConfig.java**:

```java
package dev.alefiengo.productservice.config;

import dev.alefiengo.productservice.security.JwtAuthenticationFilter;
import org.springframework.beans.factory.annotation.Autowired;
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

    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/auth/**").permitAll()
                        .requestMatchers("/actuator/**").permitAll()
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

### 4.6 Crear DTOs de Autenticación

**src/main/java/dev/alefiengo/productservice/model/dto/LoginRequest.java**:

```java
package dev.alefiengo.productservice.model.dto;

public class LoginRequest {
    private String username;
    private String password;

    // Constructors
    public LoginRequest() {}

    public LoginRequest(String username, String password) {
        this.username = username;
        this.password = password;
    }

    // Getters and Setters
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

**src/main/java/dev/alefiengo/productservice/model/dto/AuthResponse.java**:

```java
package dev.alefiengo.productservice.model.dto;

public class AuthResponse {
    private String token;

    // Constructors
    public AuthResponse() {}

    public AuthResponse(String token) {
        this.token = token;
    }

    // Getters and Setters
    public String getToken() {
        return token;
    }

    public void setToken(String token) {
        this.token = token;
    }
}
```

### 4.7 Crear AuthController.java

**src/main/java/dev/alefiengo/productservice/controller/AuthController.java**:

```java
package dev.alefiengo.productservice.controller;

import dev.alefiengo.productservice.model.dto.AuthResponse;
import dev.alefiengo.productservice.model.dto.LoginRequest;
import dev.alefiengo.productservice.security.JwtUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private PasswordEncoder passwordEncoder;

    // Hardcoded user para demo
    // En producción: consultar base de datos con UserService
    private static final String DEMO_USERNAME = "admin";
    // Password: "admin123" hasheado con BCrypt
    private static final String DEMO_PASSWORD_HASH = "$2a$10$ZwfJvLXb6hJZHnC8vkLYZu5M5u5qVqBqBFGQyPXyZJqJZqJZqJZqJ";

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {

        // Verificar username
        if (!DEMO_USERNAME.equals(request.getUsername())) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body("Invalid credentials");
        }

        // Verificar password
        if (!passwordEncoder.matches(request.getPassword(), DEMO_PASSWORD_HASH)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body("Invalid credentials");
        }

        // Generar JWT
        String token = jwtUtil.generateToken(request.getUsername());

        return ResponseEntity.ok(new AuthResponse(token));
    }
}
```

**IMPORTANTE**: Este controlador usa usuario hardcodeado para simplificar el lab. En producción, debes:
1. Crear tabla `users` en base de datos
2. Implementar `UserService` para consultar usuarios
3. Guardar passwords hasheados con BCrypt

### 4.8 Generar Hash de Password

Para obtener el hash de "admin123":

```bash
# Ejecutar en cualquier Spring Boot app con BCrypt
java -jar target/product-service.jar

# O usar herramienta online: https://bcrypt-generator.com/
# Password: admin123
# Rounds: 10
# Hash: $2a$10$...
```

### 4.9 Probar Autenticación

```bash
# 1. Intentar acceder sin token (debe fallar con 403)
curl http://localhost:8080/api/products

# Response: 403 Forbidden

# 2. Login para obtener token
TOKEN=$(curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  -s | jq -r '.token')

echo $TOKEN

# 3. Acceder con token
curl http://localhost:8080/api/products \
  -H "Authorization: Bearer $TOKEN"

# Response: 200 OK con lista de productos

# 4. Crear producto con token
curl -X POST http://localhost:8080/api/products \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop Dell",
    "description": "Laptop profesional",
    "price": 1500.00,
    "categoryId": 1
  }'
```

---

## 5. Conceptos aprendidos

- **JWT (JSON Web Token)**: Estándar para autenticación stateless
- **Bearer token**: Formato de autorización HTTP
- **Spring Security**: Framework de seguridad de Spring Boot
- **OncePerRequestFilter**: Filtro que se ejecuta una vez por request
- **SecurityFilterChain**: Cadena de filtros de seguridad
- **BCryptPasswordEncoder**: Hash seguro de passwords
- **STATELESS session**: No guardar sesión en servidor
- **CSRF disabled**: No necesario en APIs REST stateless
- **HS512 algorithm**: Algoritmo de firma HMAC con SHA-512

---

## 6. Troubleshooting

### Problema 1: 403 Forbidden en todos los endpoints

**Síntoma**: Todos los requests devuelven 403, incluso `/api/auth/login`.

**Causa**: Spring Security bloquea todo por defecto.

**Solución**: Verificar `SecurityConfig.java` permite `/api/auth/**`:

```java
.requestMatchers("/api/auth/**").permitAll()
```

### Problema 2: Token inválido

**Error**: `401 Unauthorized` al usar token.

**Causa 1**: JWT secret diferente entre generación y validación.

**Solución**: Verificar misma clave en `application.yml`:

```yaml
jwt:
  secret: ${JWT_SECRET:mySecretKeyForDevelopmentOnlyDoNotUseInProductionChangeThis}
```

**Causa 2**: Token expiró.

**Solución**: Generar nuevo token haciendo login nuevamente.

### Problema 3: Password no coincide

**Error**: Login devuelve 401 "Invalid credentials".

**Causa**: Password hash incorrecto.

**Solución**: Generar hash correcto de "admin123":

```java
public static void main(String[] args) {
    BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
    String hash = encoder.encode("admin123");
    System.out.println(hash);
}
```

Copiar hash generado a `DEMO_PASSWORD_HASH` en `AuthController`.

### Problema 4: CORS errors en frontend

**Error**: `CORS policy: No 'Access-Control-Allow-Origin' header`

**Solución**: Agregar configuración CORS en `SecurityConfig`:

```java
http
    .cors(cors -> cors.configurationSource(request -> {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("http://localhost:3000");
        config.addAllowedMethod("*");
        config.addAllowedHeader("*");
        return config;
    }))
    .csrf(csrf -> csrf.disable())
    // ...
```

### Problema 5: jjwt dependency errors

**Error**: `ClassNotFoundException: io.jsonwebtoken.Jwts`

**Causa**: Falta alguna dependencia de JJWT.

**Solución**: Verificar las 3 dependencias en `pom.xml`:

```xml
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

---

## 7. Desafío adicional

1. **Implementa refresh token** para renovar tokens sin re-login:

```java
@PostMapping("/refresh")
public ResponseEntity<?> refreshToken(@RequestHeader("Authorization") String authHeader) {
    String oldToken = authHeader.substring(7);

    if (jwtUtil.validateToken(oldToken)) {
        String username = jwtUtil.getUsernameFromToken(oldToken);
        String newToken = jwtUtil.generateToken(username);
        return ResponseEntity.ok(new AuthResponse(newToken));
    }

    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
}
```

2. **Agrega roles y permisos**:

```java
// En JwtUtil.generateToken()
return Jwts.builder()
    .setSubject(username)
    .claim("roles", Arrays.asList("ADMIN", "USER"))
    .setIssuedAt(new Date())
    .setExpiration(new Date(System.currentTimeMillis() + expiration))
    .signWith(getSigningKey(), SignatureAlgorithm.HS512)
    .compact();

// En SecurityConfig
.requestMatchers("/api/products/**").hasRole("ADMIN")
.requestMatchers("/api/categories/**").hasAnyRole("ADMIN", "USER")
```

3. **Implementa blacklist de tokens revocados** (usando Redis):

```java
@Service
public class TokenBlacklistService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public void blacklistToken(String token) {
        redisTemplate.opsForValue().set(token, "blacklisted", 24, TimeUnit.HOURS);
    }

    public boolean isBlacklisted(String token) {
        return redisTemplate.hasKey(token);
    }
}
```

---

## 8. Recursos adicionales

- [JWT.io](https://jwt.io/) - Debugger y documentación de JWT
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/index.html)
- [JJWT Documentation](https://github.com/jwtk/jjwt)
- [RFC 7519 - JWT Specification](https://datatracker.ietf.org/doc/html/rfc7519)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---

**Siguiente**: [Lab 04: Logging Estructurado](../04-logging/README.md)
