# Conceptos - Clase 8: Producción, Seguridad y Consolidación

## Introducción

En esta clase final aprendemos a preparar microservicios para entornos de producción. Implementamos configuración por perfiles, monitoreo, seguridad y logging estructurado, siguiendo las mejores prácticas de la industria.

---

## 12-Factor App

### ¿Qué es 12-Factor App?

**12-Factor App** es una metodología para construir aplicaciones como servicio (SaaS) modernas, creada por desarrolladores de Heroku. Define 12 principios para aplicaciones:

- Portables entre entornos
- Desplegables en plataformas cloud
- Escalables horizontalmente
- Mantenibles a largo plazo

### Principios relevantes para esta clase

#### III. Config - Configuración

**Principio**: Almacenar la configuración en el entorno, no en el código.

**Malo**:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://prod-db.company.com:5432/ecommerce
    username: admin
    password: secretPassword123
```

**Bueno**:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

**Beneficios**:
- Mismo código en todos los entornos
- Secretos no comprometidos en Git
- Fácil cambio de configuración sin recompilar

#### VI. Processes - Procesos

**Principio**: Ejecutar la aplicación como uno o más procesos stateless.

**Aplicado**:
- Microservicios no guardan estado en memoria
- Sesiones manejadas con tokens (JWT)
- Estado persistido en PostgreSQL o Kafka

#### XI. Logs - Logs

**Principio**: Tratar logs como streams de eventos.

**Aplicado**:
- Escribir logs a stdout/stderr
- No gestionar archivos de log en la aplicación
- Sistema externo recolecta y agrega logs

---

## Spring Boot Profiles

### ¿Qué son los Profiles?

**Profiles** permiten configurar la aplicación de forma diferente según el entorno donde se ejecuta (desarrollo, testing, producción).

### Casos de uso

**Desarrollo (dev)**:
- Logging verboso (DEBUG)
- SQL queries visibles
- H2 console habilitada
- Datos de prueba precargados

**Producción (prod)**:
- Logging mínimo (INFO, WARN, ERROR)
- SQL queries deshabilitados
- Conexiones a base de datos real
- Configuración optimizada

### Estructura de archivos

```
src/main/resources/
├── application.yml           # Configuración común
├── application-dev.yml       # Específico para desarrollo
└── application-prod.yml      # Específico para producción
```

### Ejemplo práctico

**application.yml** (común):
```yaml
spring:
  application:
    name: product-service
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
```

**application-dev.yml**:
```yaml
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
```

**application-prod.yml**:
```yaml
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

### Activación de perfiles

**Línea de comandos**:
```bash
# Maven
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# JAR
java -jar app.jar --spring.profiles.active=prod
```

**Variable de entorno**:
```bash
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar
```

**Docker Compose**:
```yaml
services:
  product-service:
    environment:
      - SPRING_PROFILES_ACTIVE=prod
```

---

## Spring Boot Actuator

### ¿Qué es Actuator?

**Spring Boot Actuator** proporciona endpoints listos para usar que exponen información operativa de la aplicación: health checks, métricas, información del entorno.

### Endpoints principales

| Endpoint | Descripción |
|----------|-------------|
| `/actuator/health` | Estado de la aplicación (UP/DOWN) |
| `/actuator/info` | Información de la aplicación (versión, git) |
| `/actuator/metrics` | Métricas de JVM, HTTP, Kafka |
| `/actuator/env` | Variables de entorno |
| `/actuator/loggers` | Configuración de logging |

### Health Checks

**¿Por qué son importantes?**

- Kubernetes/Docker usan health checks para saber si reiniciar un contenedor
- Load balancers verifican si un servicio puede recibir tráfico
- Sistemas de monitoreo alertan si un servicio está caído

**Ejemplo de respuesta**:
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
    }
  }
}
```

### Métricas

Actuator expone métricas sobre:

- **JVM**: Memoria heap, threads, garbage collection
- **HTTP**: Requests por segundo, latencia, errores
- **Kafka**: Mensajes enviados, errores de producción
- **Database**: Conexiones activas, queries ejecutados

**Ejemplo**:
```bash
curl http://localhost:8080/actuator/metrics/jvm.memory.used

{
  "name": "jvm.memory.used",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 268435456
    }
  ]
}
```

### Seguridad de Actuator

**Importante**: Endpoints de Actuator pueden exponer información sensible.

**Prácticas recomendadas**:
1. Exponer solo endpoints necesarios
2. Proteger con autenticación (JWT, basic auth)
3. En producción, usar puerto separado

```yaml
# Solo exponer health e info públicamente
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

---

## Variables de Entorno

### ¿Por qué externalizar configuración?

**Problemas de hardcodear valores**:
- Secretos comprometidos en Git
- Mismo código no funciona en diferentes entornos
- Cambiar configuración requiere recompilar

**Ventajas de variables de entorno**:
- Separación de código y configuración
- Secretos no en repositorio
- Mismo artefacto (JAR) desplegado en todos los entornos

### Sintaxis en Spring Boot

**Patrón**: `${VARIABLE_NAME:default_value}`

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
```

**Explicación**:
- `DB_HOST:localhost` → Usa variable `DB_HOST`, si no existe usa `localhost`
- Valor por defecto permite desarrollo sin configurar variables

### Gestión de secretos

**Desarrollo local**:
- Archivo `.env` (NO committear a Git)
- Variables exportadas en shell

**Producción**:
- Kubernetes Secrets
- AWS Secrets Manager
- HashiCorp Vault
- Variables de entorno del contenedor

---

## JWT (JSON Web Tokens)

### ¿Qué es JWT?

**JWT** es un estándar abierto (RFC 7519) para transmitir información de forma segura entre partes como un objeto JSON.

### Estructura de un JWT

Un JWT tiene 3 partes separadas por puntos:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

      HEADER            .           PAYLOAD              .        SIGNATURE
```

1. **Header**: Algoritmo y tipo de token
2. **Payload**: Claims (datos del usuario)
3. **Signature**: Firma digital para verificar autenticidad

### Claims comunes

```json
{
  "sub": "admin",           // Subject (usuario)
  "iat": 1699876800,        // Issued At (timestamp)
  "exp": 1699963200,        // Expiration (timestamp)
  "roles": ["ADMIN", "USER"]
}
```

### Flujo de autenticación con JWT

```
1. [Cliente] → POST /api/auth/login (username, password)
2. [Server] → Valida credenciales
3. [Server] → Genera JWT con claims
4. [Server] → Retorna JWT
5. [Cliente] → Guarda JWT (localStorage, cookie)
6. [Cliente] → GET /api/products con header: Authorization: Bearer <JWT>
7. [Server] → Valida JWT, extrae username
8. [Server] → Retorna datos
```

### Ventajas de JWT

- **Stateless**: No requiere sesiones en servidor
- **Escalable**: Funciona en múltiples instancias
- **Cross-domain**: CORS friendly
- **Información embebida**: Claims en el token

### Consideraciones de seguridad

1. **Secret key fuerte**: Al menos 256 bits
2. **HTTPS obligatorio**: JWT en HTTP plano es inseguro
3. **Expiración corta**: Reducir ventana de compromiso
4. **Refresh tokens**: Para sesiones largas
5. **NO guardar información sensible**: JWT es decodificable

---

## Autenticación vs Autorización

### Autenticación

**Pregunta**: ¿Quién eres?

**Proceso**: Verificar identidad (username/password, JWT, OAuth)

**Resultado**: Usuario identificado

### Autorización

**Pregunta**: ¿Qué puedes hacer?

**Proceso**: Verificar permisos (roles, scopes)

**Resultado**: Acceso permitido o denegado

### En Spring Security

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()  // No auth required
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()  // Autenticación requerida
            );
        return http.build();
    }
}
```

---

## Logging Estructurado

### ¿Qué es logging estructurado?

**Logging tradicional** (string plano):
```
2025-11-07 10:30:00 Order created: ORD-123
```

**Logging estructurado** (JSON):
```json
{
  "timestamp": "2025-11-07T10:30:00Z",
  "level": "INFO",
  "service": "order-service",
  "message": "Order created",
  "orderId": "ORD-123",
  "customerId": "CUST-456",
  "amount": 150.00
}
```

### Ventajas

1. **Parseable**: Fácil de procesar con herramientas
2. **Searchable**: Buscar por campos específicos
3. **Aggregable**: Agregar métricas desde logs
4. **Context-rich**: Información estructurada

### Niveles de log

| Nivel | Uso | Ejemplo |
|-------|-----|---------|
| TRACE | Debugging muy detallado | Entrada/salida de métodos |
| DEBUG | Información de desarrollo | Valores de variables |
| INFO | Eventos normales | "Order created", "User logged in" |
| WARN | Situaciones anormales no críticas | "Retry attempt 2/3" |
| ERROR | Errores que requieren atención | "Database connection failed" |

### Prácticas recomendadas

**Incluir contexto**:
```java
// Malo
log.info("Order created");

// Bueno
log.info("Order created: orderId={}, customerId={}, amount={}",
    order.getId(),
    order.getCustomerId(),
    order.getTotalAmount());
```

**Loguear eventos importantes**:
- Inicio y fin de la aplicación
- Publicación de eventos a Kafka
- Errores de base de datos
- Requests HTTP (entrada/salida)

**NO loguear**:
- Contraseñas o secretos
- Información personal sensible (PII)
- Datos de tarjetas de crédito

---

## Checklist de Producción

Antes de desplegar a producción, verificar:

### Configuración

- [ ] Perfiles configurados (dev, prod)
- [ ] Variables de entorno externalizadas
- [ ] Secretos NO en código
- [ ] Logging apropiado por entorno

### Monitoreo

- [ ] Actuator habilitado
- [ ] Health checks configurados
- [ ] Métricas expuestas
- [ ] Logs estructurados

### Seguridad

- [ ] Endpoints protegidos con JWT
- [ ] CORS configurado
- [ ] Actuator protegido
- [ ] HTTPS en producción

### Base de datos

- [ ] Migrations versionadas (Flyway/Liquibase)
- [ ] Connection pooling configurado
- [ ] Índices en columnas frecuentes
- [ ] Backups automatizados

### Kafka

- [ ] Topics creados con configuración correcta
- [ ] Replication factor > 1 en producción
- [ ] Serialización consistente
- [ ] Error handling implementado

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Perfil | Profile | Configuración específica por entorno |
| Verificación de salud | Health Check | Endpoint para verificar estado del servicio |
| Métrica | Metric | Medición cuantitativa del sistema |
| Variable de entorno | Environment Variable | Configuración externa al código |
| Token web JSON | JSON Web Token (JWT) | Token autofirmado para autenticación |
| Autenticación | Authentication | Verificar identidad del usuario |
| Autorización | Authorization | Verificar permisos del usuario |
| Registro estructurado | Structured Logging | Logs en formato parseable (JSON) |
| Secreto | Secret | Información sensible (passwords, keys) |
| Sin estado | Stateless | No guarda estado entre requests |
