# FAQ - Clase 8: Producción, Seguridad y Consolidación

## Errores comunes y soluciones

### 1. Error: "Profile 'prod' not found"

**Problema**: Spring Boot no encuentra el archivo de perfil.

**Solución**:

Verificar que existe el archivo en la ubicación correcta:
```
src/main/resources/application-prod.yml
```

Verificar sintaxis del nombre del archivo:
```bash
# Correcto
application-prod.yml

# Incorrecto
application-production.yml
application_prod.yml
```

---

### 2. Error: "Failed to configure a DataSource: 'url' attribute is not specified"

**Problema**: Variable de entorno no está definida y no hay valor por defecto.

**Causa**:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    # Sin valor por defecto (:localhost)
```

**Solución**:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
    # Con valores por defecto
```

O definir variables:
```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=ecommerce
```

---

### 3. Error: "Actuator endpoint /actuator/health returns 404"

**Problema**: Actuator no está habilitado o endpoint no está expuesto.

**Solución**:

Verificar dependencia en `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Verificar configuración en `application.yml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

Recompilar:
```bash
mvn clean install
mvn spring-boot:run
```

---

### 4. Error: "JWT signature does not match locally computed signature"

**Problema**: Secret key diferente entre generación y validación del JWT.

**Causas posibles**:
1. Variable `JWT_SECRET` no definida (usa default diferente)
2. Secret cambió entre generación del token y validación
3. Token copiado incorrectamente (falta caracteres)

**Solución**:

Verificar que el secret es el mismo:
```yaml
jwt:
  secret: ${JWT_SECRET:mySameSecretForDevAndProd}
```

Definir variable de entorno:
```bash
export JWT_SECRET=myVerySecretKeyThatIsAtLeast256BitsLong
```

Regenerar token:
```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

---

### 5. Error: "Access Denied" al intentar acceder a endpoints

**Problema**: JWT no se está enviando correctamente o es inválido.

**Solución**:

Verificar header de autorización:
```bash
# Correcto
curl http://localhost:8080/api/products \
  -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9..."

# Incorrecto (falta "Bearer")
curl http://localhost:8080/api/products \
  -H "Authorization: eyJhbGciOiJIUzUxMiJ9..."
```

Verificar que el token no expiró:
```bash
# Decodificar JWT en jwt.io y revisar campo "exp"
```

---

### 6. Error: "The dependencies of some of the beans in the application context form a cycle"

**Problema**: Inyección circular en configuración de seguridad.

**Causa común**:
```java
@Configuration
public class SecurityConfig {
    private UserDetailsService userDetailsService;  // Evitar inyección innecesaria

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        // ...
    }
}
```

**Solución**:

Usar constructor injection (Spring 4.3+):
```java
@Configuration
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    // Constructor injection - Spring 4.3+ no requiere @Autowired
    public SecurityConfig(JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        // ...
    }
}
```

---

### 7. Error: "Logback configuration error"

**Problema**: Sintaxis incorrecta en `logback-spring.xml`.

**Solución**:

Verificar que el XML está bien formado:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>
</configuration>
```

Verificar ubicación del archivo:
```
src/main/resources/logback-spring.xml
```

---

### 8. Error: "Cannot authenticate user with password"

**Problema**: Password hasheado incorrectamente o no coincide.

**Solución**:

Generar hash BCrypt correcto:
```bash
# Online: https://bcrypt-generator.com/
# Input: admin123
# Output: $2a$10$N9qo8uLOickgx2ZMRZoMye6lHgl6XUUGpJdCo/IqLSAWFZHvF9vF2
```

Usar en código:
```java
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request) {
    if ("admin".equals(request.getUsername()) &&
        passwordEncoder.matches(request.getPassword(),
            "$2a$10$N9qo8uLOickgx2ZMRZoMye6lHgl6XUUGpJdCo/IqLSAWFZHvF9vF2")) {

        String token = jwtUtil.generateToken(request.getUsername());
        return ResponseEntity.ok(new AuthResponse(token));
    }
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
}
```

---

### 9. Health check muestra "DOWN" para Kafka

**Problema**: Kafka no está accesible o configuración incorrecta.

**Verificar**:
```bash
# Kafka corriendo?
docker compose ps

# Logs de Kafka
docker compose logs kafka

# Conectividad
telnet localhost 9092
```

**Solución temporal**:

Deshabilitar health check de Kafka si no es crítico:
```yaml
management:
  health:
    kafka:
      enabled: false
```

---

### 10. Variables de entorno no se cargan en Maven

**Problema**: Variables definidas en terminal no llegan a Spring Boot.

**Solución 1 - Definir en comando**:
```bash
DB_HOST=localhost DB_PORT=5432 mvn spring-boot:run
```

**Solución 2 - Exportar antes**:
```bash
export DB_HOST=localhost
export DB_PORT=5432
mvn spring-boot:run
```

**Solución 3 - Pasar como parámetros**:
```bash
mvn spring-boot:run \
  -Dspring-boot.run.arguments="--DB_HOST=localhost --DB_PORT=5432"
```

---

## Preguntas frecuentes

### ¿Cuántos perfiles puedo tener?

**Respuesta**: Los que necesites. Comunes:

- `dev` - Desarrollo local
- `test` - Testing automatizado
- `staging` - Ambiente de staging
- `prod` - Producción

**Ejemplo**:
```bash
# Activar múltiples perfiles
java -jar app.jar --spring.profiles.active=prod,monitoring,aws
```

---

### ¿Debo proteger todos los endpoints con JWT?

**No siempre**. Endpoints públicos comunes:

- `/api/auth/login` - Login endpoint
- `/api/auth/register` - Registro
- `/actuator/health` - Health check (Kubernetes lo usa)
- `/api/public/**` - Recursos públicos

**Configuración**:
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**").permitAll()
    .requestMatchers("/actuator/health").permitAll()
    .anyRequest().authenticated()
)
```

---

### ¿Qué endpoints de Actuator debería exponer en producción?

**Regla general**: Solo lo necesario.

**Desarrollo**:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,loggers
```

**Producción**:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  # Proteger con autenticación si es posible
```

**Riesgo**: `/actuator/env` expone variables de entorno (incluidos secretos si no se ocultan).

---

### ¿Cómo ocultar secretos en /actuator/env?

**Solución**:

Spring Boot automáticamente oculta propiedades que contienen:
- `password`
- `secret`
- `key`
- `token`
- `credential`

**Verificar**:
```bash
curl http://localhost:8080/actuator/env | grep -i password
# Debería mostrar: "******"
```

---

### ¿Dónde guardo el JWT secret en producción?

**Nunca** en `application.yml` commiteado a Git.

**Opciones**:

1. **Variable de entorno** (simple):
```bash
export JWT_SECRET=myVeryLongSecretKeyThatIsAtLeast256Bits
```

2. **Kubernetes Secret**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
data:
  jwt-secret: bXlTZWNyZXRLZXk=  # Base64
```

3. **AWS Secrets Manager**
4. **HashiCorp Vault**

---

### ¿Cuál es la diferencia entre application.yml y application.properties?

**Funcionalmente**: Ninguna, es preferencia de sintaxis.

**application.yml** (YAML):
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/db
    username: postgres
```

**application.properties**:
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/db
spring.datasource.username=postgres
```

**Recomendación**: YAML es más legible para configuraciones complejas.

---

### ¿Cómo manejo JWT refresh tokens?

**Concepto**: Token de corta duración (15 min) + refresh token de larga duración (7 días).

**Flujo**:
1. Login → retorna access token (15 min) + refresh token (7 días)
2. Cliente usa access token
3. Al expirar, usa refresh token para obtener nuevo access token
4. Si refresh expira, hacer login de nuevo

**Implementación** (fuera del alcance de esta clase):
- Guardar refresh tokens en base de datos
- Endpoint `/api/auth/refresh`
- Validar refresh token y generar nuevo access token

---

### ¿Puedo usar JWT con microservicios?

**Sí**, es una práctica común.

**Estrategias**:

1. **JWT compartido** (este curso):
   - Mismo secret en todos los servicios
   - Cada servicio valida el JWT independientemente
   - Simple pero menos seguro

2. **API Gateway centralizado** (producción):
   - Gateway valida JWT
   - Servicios internos confían en Gateway
   - Más seguro, requiere infraestructura

3. **Service-to-service tokens**:
   - JWT diferente para comunicación entre servicios
   - Más complejo, mayor seguridad

---

### ¿Cómo debugging logs en producción sin cambiar código?

**Solución**: Cambiar nivel de log dinámicamente con Actuator.

**Verificar nivel actual**:
```bash
curl http://localhost:8080/actuator/loggers/dev.alefiengo.productservice
```

**Cambiar a DEBUG**:
```bash
curl -X POST http://localhost:8080/actuator/loggers/dev.alefiengo.productservice \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

**Volver a INFO**:
```bash
curl -X POST http://localhost:8080/actuator/loggers/dev.alefiengo.productservice \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "INFO"}'
```

---

### ¿Qué hacer si los 4 microservicios no levantan?

**Checklist de troubleshooting**:

1. **Verificar infraestructura**:
```bash
docker compose ps  # Kafka y Postgres deben estar "Up"
```

2. **Verificar puertos**:
```bash
# Cada servicio debe usar puerto diferente
product-service: 8080
order-service: 8081
inventory-service: 8082
analytics-service: 8083
```

3. **Verificar bases de datos existen**:
```bash
docker exec -it postgres psql -U postgres -c "\l"
# Debe mostrar: ecommerce, ecommerce_orders, ecommerce_inventory
```

4. **Revisar logs**:
```bash
# Ver logs del servicio que falla
mvn spring-boot:run
# Buscar errores de conexión a BD o Kafka
```

5. **Levantar uno a uno**:
```bash
# Levantar en orden:
cd product-service && mvn spring-boot:run &
cd order-service && mvn spring-boot:run &
cd inventory-service && mvn spring-boot:run &
cd analytics-service && mvn spring-boot:run &
```

---

### ¿Cómo sé si mi aplicación está lista para producción?

**Checklist básico**:

- [ ] Perfiles configurados (dev, prod)
- [ ] Variables de entorno para secretos
- [ ] Actuator health check funciona
- [ ] JWT implementado
- [ ] Logging estructurado
- [ ] HTTPS configurado (en producción)
- [ ] Connection pooling configurado
- [ ] Error handling completo
- [ ] Métricas expuestas
- [ ] Tests automatizados
- [ ] CI/CD pipeline
- [ ] Monitoring (Prometheus/Grafana)
- [ ] Backups automatizados

**Nota**: Este curso cubre los primeros 5 ítems. Los demás son temas avanzados.

---

## Troubleshooting avanzado

### Ver todas las variables de entorno que Spring Boot usa

```bash
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.name | contains("systemEnvironment"))'
```

### Ver configuración activa

```bash
curl http://localhost:8080/actuator/env | jq '.activeProfiles'
```

### Verificar que JWT es válido (decodificar)

Sitio web: https://jwt.io/

Pegar el token y verificar:
- Header: Algoritmo HS512
- Payload: Username, exp (expiration)
- Signature: Debe estar verificada

---

## Recursos adicionales

- [Spring Boot Production Best Practices](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment.html)
- [Spring Security JWT Tutorial](https://www.baeldung.com/spring-security-jwt)
- [12-Factor App Methodology](https://12factor.net/)
- [Spring Boot Actuator Endpoints](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints)
- [Logback Configuration](https://logback.qos.ch/manual/configuration.html)
