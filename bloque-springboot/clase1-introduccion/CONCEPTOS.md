# Conceptos Fundamentales - Clase 1

Este documento contiene la teoría profunda sobre los conceptos clave de la Clase 1: Introducción a Microservicios y Spring Boot.

---

## Tabla de Contenidos

1. [Microservicios](#1-microservicios)
2. [Spring Framework y Spring Boot](#2-spring-framework-y-spring-boot)
3. [REST API](#3-rest-api)
4. [Arquitectura en Capas](#4-arquitectura-en-capas)
5. [Dependency Injection e IoC](#5-dependency-injection-e-ioc)
6. [Spring Boot Actuator](#6-spring-boot-actuator)

---

## 1. Microservicios

### 1.1 ¿Qué es un Microservicio?

Un **microservicio** es una aplicación pequeña, independiente y autónoma que realiza una función de negocio específica y se comunica con otros servicios mediante APIs bien definidas.

**Características clave:**
- **Pequeño y enfocado**: Una sola responsabilidad
- **Independiente**: Desarrollado, desplegado y escalado por separado
- **Autónomo**: Tiene su propia base de datos (Database per Service)
- **Comunicación por API**: REST, gRPC, mensajería (Kafka, RabbitMQ)
- **Tecnológicamente diverso**: Cada servicio puede usar diferentes tecnologías

### 1.2 Monolito vs Microservicios

**Arquitectura Monolítica:**

```
┌──────────────────────────────────────┐
│         Aplicación Monolítica        │
│  ┌────────────────────────────────┐  │
│  │  UI Layer                      │  │
│  ├────────────────────────────────┤  │
│  │  Business Logic                │  │
│  │  - Usuarios                    │  │
│  │  - Productos                   │  │
│  │  - Órdenes                     │  │
│  │  - Pagos                       │  │
│  ├────────────────────────────────┤  │
│  │  Data Access Layer             │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
          │
          v
┌──────────────────────────────────────┐
│      Base de Datos Única             │
└──────────────────────────────────────┘
```

**Ventajas del Monolito:**
- Simple de desarrollar inicialmente
- Fácil de testear (todo en un solo proceso)
- Deployment sencillo (un solo artefacto)
- Rendimiento: No hay latencia de red entre componentes

**Desventajas del Monolito:**
- Difícil de escalar (todo o nada)
- Acoplamiento: Un cambio puede afectar todo
- Deployment riesgoso (todo o nada)
- Tecnología fija (difícil cambiar stack)
- Equipos bloqueados (todos trabajan en el mismo código)

---

**Arquitectura de Microservicios:**

```
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  User Service   │   │ Product Service │   │  Order Service  │
│  (Spring Boot)  │   │  (Spring Boot)  │   │   (Node.js)     │
│                 │   │                 │   │                 │
│    [API]        │   │    [API]        │   │    [API]        │
└────────┬────────┘   └────────┬────────┘   └────────┬────────┘
         │                     │                      │
         v                     v                      v
   ┌─────────┐           ┌─────────┐            ┌─────────┐
   │ User DB │           │ Prod DB │            │Order DB │
   └─────────┘           └─────────┘            └─────────┘

         REST APIs / Message Broker (Kafka)
```

**Ventajas de Microservicios:**
- Escalabilidad independiente (escala solo lo necesario)
- Deployment independiente (menos riesgo)
- Diversidad tecnológica (mejor herramienta por problema)
- Equipos autónomos (ownership por servicio)
- Resiliencia (falla uno, no todos)

**Desventajas de Microservicios:**
- Complejidad operacional (múltiples deployments)
- Latencia de red entre servicios
- Debugging más difícil (distributed tracing)
- Consistencia de datos (eventual consistency)
- Overhead de infraestructura (múltiples DBs, servicios)

### 1.3 ¿Cuándo usar Microservicios?

**Usa Microservicios cuando:**
- Equipo grande (50+ desarrolladores)
- Dominios de negocio bien definidos
- Necesitas escalar componentes específicos
- Diferentes equipos con ownership claro
- Alta frecuencia de deployments
- Tolerancia a complejidad operacional

**Usa Monolito cuando:**
- Equipo pequeño (< 10 desarrolladores)
- Producto en fase inicial (MVP)
- Dominio de negocio no está claro
- No necesitas escalar independientemente
- Infraestructura limitada

**Estrategia recomendada:**
1. Empieza con monolito modular (bien estructurado)
2. Identifica bounded contexts claros
3. Extrae microservicios cuando haya necesidad real

### 1.4 Patrones de Microservicios

**API Gateway Pattern:**
```
┌──────────┐
│  Cliente │
└────┬─────┘
     │
     v
┌──────────────────┐
│   API Gateway    │  (Routing, Auth, Rate Limiting)
└────┬─────────────┘
     │
     ├──────────> [User Service]
     ├──────────> [Product Service]
     └──────────> [Order Service]
```

**Service Discovery Pattern:**
- Eureka (Netflix), Consul, etcd
- Los servicios se registran automáticamente
- El gateway descubre dónde están

**Circuit Breaker Pattern:**
- Resilience4j, Hystrix
- Previene cascadas de fallos
- Fallback cuando un servicio falla

**Database per Service:**
- Cada microservicio tiene su propia BD
- No acceso directo a BD de otro servicio
- Comunicación solo vía API

---

## 2. Spring Framework y Spring Boot

### 2.1 ¿Qué es Spring Framework?

**Spring Framework** es un framework de Java para desarrollo de aplicaciones empresariales, basado en:

- **Inversión de Control (IoC)**
- **Dependency Injection (DI)**
- **Aspect-Oriented Programming (AOP)**
- **Abstracción de capas** (JDBC, JPA, MVC, etc.)

**Historia:**
- **2002**: Rod Johnson publica "Expert One-on-One J2EE Design and Development"
- **2004**: Spring Framework 1.0
- **2014**: Spring Boot 1.0 (simplificación)
- **2022**: Spring Framework 6.0 (Java 17 baseline)
- **2024**: Spring Boot 3.x (actual)

### 2.2 ¿Qué es Spring Boot?

**Spring Boot** = Spring Framework + Opiniones + Auto-configuración

**Filosofía:**
- Convention over Configuration
- Starter dependencies (agrupa dependencias comunes)
- Auto-configuration (configura automáticamente según classpath)
- Embedded server (Tomcat, Jetty, Undertow)
- Production-ready features (Actuator)

**Diferencia clave:**

```xml
<!-- Spring Framework (manual) -->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
    <property name="username" value="root"/>
    <property name="password" value="password"/>
</bean>

<!-- Spring Boot (auto-configurado) -->
<!-- Solo necesitas: -->
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password
```

### 2.3 Spring Boot Starters

**¿Qué es un Starter?**

Un starter es una dependencia que agrupa múltiples librerías relacionadas.

**Ejemplos:**

```xml
<!-- spring-boot-starter-web incluye: -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Internamente incluye: -->
- spring-web
- spring-webmvc
- jackson (JSON)
- tomcat-embed (servidor)
- validation-api
```

**Starters comunes:**

| Starter | Propósito |
|---------|-----------|
| `spring-boot-starter-web` | REST APIs, Spring MVC |
| `spring-boot-starter-data-jpa` | JPA, Hibernate, base de datos |
| `spring-boot-starter-security` | Autenticación y autorización |
| `spring-boot-starter-test` | JUnit, Mockito, testing |
| `spring-boot-starter-actuator` | Monitoreo y métricas |
| `spring-kafka` | Apache Kafka |
| `spring-boot-starter-validation` | Bean Validation |

### 2.4 Auto-configuración

**¿Cómo funciona?**

Spring Boot analiza el classpath y configura beans automáticamente:

```java
@SpringBootApplication  // Combina 3 anotaciones:
    // @Configuration       - Define beans
    // @EnableAutoConfiguration - Auto-configuración mágica
    // @ComponentScan       - Escanea @Component, @Service, etc.
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

**Ejemplo de auto-configuración:**

1. Spring Boot detecta `spring-boot-starter-web` en el classpath
2. Configura automáticamente:
   - DispatcherServlet (Spring MVC)
   - Embedded Tomcat server
   - HttpMessageConverters (JSON con Jackson)
   - Error handling
3. Todo sin XML ni configuración manual

**Ver configuraciones aplicadas:**
```bash
java -jar app.jar --debug
# o
curl http://localhost:8080/actuator/conditions
```

### 2.5 Embedded Server

**Antes de Spring Boot:**
```
1. Desarrollar aplicación
2. Empaquetar como WAR
3. Instalar Tomcat/JBoss
4. Desplegar WAR en servidor
5. Configurar servidor
```

**Con Spring Boot:**
```
1. Desarrollar aplicación
2. Empaquetar como JAR (con Tomcat embebido)
3. Ejecutar: java -jar app.jar
```

**Beneficios:**
- No necesitas instalar servidor por separado
- Aplicación auto-contenida
- Consistencia entre dev/prod
- Fácil de contenerizar (Docker)

**Servers soportados:**
- Tomcat (default)
- Jetty
- Undertow

---

## 3. REST API

### 3.1 ¿Qué es REST?

**REST** (Representational State Transfer) es un estilo arquitectural para APIs basado en:

1. **Recursos**: Entidades identificadas por URLs
2. **Métodos HTTP**: Operaciones sobre recursos
3. **Stateless**: Sin estado entre peticiones
4. **Representaciones**: JSON, XML, HTML

**Principios REST:**

- **Client-Server**: Separación de concerns
- **Stateless**: Cada petición es independiente
- **Cacheable**: Respuestas pueden ser cacheadas
- **Uniform Interface**: Interfaz consistente
- **Layered System**: Puede haber intermediarios (proxies, load balancers)

### 3.2 Recursos y URLs

**Recursos = Sustantivos (no verbos)**

```
BUENO:
GET    /users          - Listar usuarios
GET    /users/123      - Obtener usuario 123
POST   /users          - Crear usuario
PUT    /users/123      - Actualizar usuario 123
DELETE /users/123      - Eliminar usuario 123

MALO:
GET    /getUsers
POST   /createUser
GET    /deleteUser?id=123
```

**Jerarquía de recursos:**

```
/users/123                    - Usuario 123
/users/123/orders             - Órdenes del usuario 123
/users/123/orders/456         - Orden 456 del usuario 123
/users/123/orders/456/items   - Items de la orden 456
```

### 3.3 Métodos HTTP (Verbos)

| Método | Operación | Idempotente | Safe | Request Body | Response Body |
|--------|-----------|-------------|------|--------------|---------------|
| **GET** | Leer | Sí | Sí | No | Sí |
| **POST** | Crear | No | No | Sí | Sí |
| **PUT** | Actualizar (completo) | Sí | No | Sí | Sí |
| **PATCH** | Actualizar (parcial) | No | No | Sí | Sí |
| **DELETE** | Eliminar | Sí | No | No | Opcional |

**Idempotente**: Llamar N veces = mismo resultado que llamar 1 vez

**Safe**: No modifica recursos (solo lectura)

**Ejemplos:**

```bash
# GET - Leer (idempotente, safe)
GET /users/123
GET /users/123  # Llamar 2 veces retorna lo mismo

# POST - Crear (NO idempotente)
POST /users + {"name":"Juan"}  # Crea usuario ID 1
POST /users + {"name":"Juan"}  # Crea usuario ID 2 (diferente)

# PUT - Actualizar completo (idempotente)
PUT /users/123 + {"name":"Juan","email":"j@example.com"}
PUT /users/123 + {"name":"Juan","email":"j@example.com"}  # Mismo resultado

# DELETE - Eliminar (idempotente)
DELETE /users/123  # Usuario eliminado
DELETE /users/123  # Usuario ya no existe (mismo estado final)
```

### 3.4 Códigos de Estado HTTP

**2xx - Éxito:**

| Código | Significado | Uso |
|--------|-------------|-----|
| 200 OK | Operación exitosa | GET, PUT, PATCH exitosos |
| 201 Created | Recurso creado | POST exitoso |
| 204 No Content | Exitoso sin contenido | DELETE exitoso |

**4xx - Error del Cliente:**

| Código | Significado | Uso |
|--------|-------------|-----|
| 400 Bad Request | Request inválido | Validación falla, JSON inválido |
| 401 Unauthorized | No autenticado | Sin token, token inválido |
| 403 Forbidden | No autorizado | Autenticado pero sin permisos |
| 404 Not Found | Recurso no existe | GET/PUT/DELETE de ID inexistente |
| 409 Conflict | Conflicto | Email duplicado, constraint violation |

**5xx - Error del Servidor:**

| Código | Significado | Uso |
|--------|-------------|-----|
| 500 Internal Server Error | Error inesperado | Exception no manejado |
| 503 Service Unavailable | Servicio no disponible | Base de datos caída |

### 3.5 Formato de Respuesta

**Respuesta exitosa:**

```json
// GET /users/123
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "name": "Juan Pérez",
  "email": "juan@example.com",
  "createdAt": "2025-10-20T10:30:00Z"
}
```

**Respuesta de error:**

```json
// POST /users con datos inválidos
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "timestamp": "2025-10-20T10:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Email inválido"
    },
    {
      "field": "name",
      "message": "Nombre es obligatorio"
    }
  ],
  "path": "/users"
}
```

### 3.6 Filtrado, Paginación, Ordenamiento

**Query Parameters para operaciones:**

```bash
# Filtrado
GET /products?category=electronics&minPrice=100&maxPrice=500

# Ordenamiento
GET /products?sort=price,asc
GET /products?sort=name,desc

# Paginación
GET /products?page=0&size=20

# Combinado
GET /products?category=books&sort=price,asc&page=1&size=10
```

**Respuesta paginada:**

```json
{
  "content": [
    {"id": 1, "name": "Producto 1"},
    {"id": 2, "name": "Producto 2"}
  ],
  "page": 0,
  "size": 20,
  "totalElements": 100,
  "totalPages": 5,
  "first": true,
  "last": false
}
```

### 3.7 Versionado de API

**Opciones:**

```bash
# 1. URL Path (más común)
GET /api/v1/users
GET /api/v2/users

# 2. Query Parameter
GET /api/users?version=1

# 3. Header
GET /api/users
Accept: application/vnd.myapi.v1+json

# 4. Content Negotiation
GET /api/users
Accept: application/vnd.myapi+json;version=1
```

**Recomendación:** URL Path (más explícito y visible)

---

## 4. Arquitectura en Capas

### 4.1 Patrón Layered Architecture

```
┌────────────────────────────────────┐
│     Presentation Layer             │  @RestController
│  (Controllers / Endpoints)         │  - Manejo de HTTP
│                                    │  - Validación de input
│                                    │  - Serialización JSON
└───────────────┬────────────────────┘
                │
┌───────────────▼────────────────────┐
│    Business Logic Layer            │  @Service
│  (Services)                        │  - Lógica de negocio
│                                    │  - Validaciones complejas
│                                    │  - Transacciones
└───────────────┬────────────────────┘
                │
┌───────────────▼────────────────────┐
│    Data Access Layer               │  @Repository
│  (Repositories)                    │  - Queries a BD
│                                    │  - CRUD operations
│                                    │  - Mapeo OR/M
└───────────────┬────────────────────┘
                │
┌───────────────▼────────────────────┐
│      Data Source                   │
│   (Database)                       │
└────────────────────────────────────┘
```

### 4.2 Responsabilidades de Cada Capa

**Controller Layer:**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // RESPONSABILIDADES:
    // Manejar HTTP (request/response)
    // Validar formato de entrada
    // Mapear DTOs a entidades
    // Retornar códigos HTTP apropiados

    // NO DEBE:
    // Lógica de negocio
    // Acceso directo a BD
    // Cálculos complejos

    @PostMapping
    public ResponseEntity<UserDTO> createUser(@Valid @RequestBody UserCreateRequest request) {
        User user = userService.create(request);
        UserDTO dto = userMapper.toDTO(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(dto);
    }
}
```

**Service Layer:**

```java
@Service
public class UserService {

    // RESPONSABILIDADES:
    // Lógica de negocio
    // Validaciones de negocio
    // Orquestar operaciones
    // Transacciones

    // NO DEBE:
    // Conocer sobre HTTP/JSON
    // Queries SQL directas
    // Manejo de excepciones HTTP

    @Transactional
    public User create(UserCreateRequest request) {
        // Validar reglas de negocio
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException("Email ya existe");
        }

        // Crear usuario
        User user = new User();
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));

        return userRepository.save(user);
    }
}
```

**Repository Layer:**

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // RESPONSABILIDADES:
    // Queries a base de datos
    // CRUD operations
    // Mapeo objeto-relacional

    // NO DEBE:
    // Lógica de negocio
    // Validaciones
    // Transacciones (eso es del service)

    boolean existsByEmail(String email);

    Optional<User> findByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.active = true")
    List<User> findAllActive();
}
```

### 4.3 DTOs vs Entities

**Entity (modelo de BD):**

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String password;  // Nunca exponer en API

    @OneToMany(mappedBy = "user")
    private List<Order> orders;

    // Campos técnicos
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

**DTO (para API):**

```java
public class UserDTO {
    private Long id;
    private String email;
    // NO incluye password
    // NO incluye relaciones complejas
    // NO incluye campos técnicos
}

public class UserCreateRequest {
    @NotBlank
    @Email
    private String email;

    @NotBlank
    @Size(min = 8)
    private String password;
}
```

**¿Por qué separar?**

1. **Seguridad**: No expones campos sensibles (password)
2. **Desacoplamiento**: Cambios en BD no afectan API
3. **Versionado**: Puedes tener múltiples DTOs por entidad
4. **Validación**: Validaciones diferentes para create/update
5. **Performance**: Evitas lazy loading issues

### 4.4 Mapper Pattern

```java
@Component
public class UserMapper {

    public UserDTO toDTO(User user) {
        UserDTO dto = new UserDTO();
        dto.setId(user.getId());
        dto.setEmail(user.getEmail());
        return dto;
    }

    public User toEntity(UserCreateRequest request) {
        User user = new User();
        user.setEmail(request.getEmail());
        return user;
    }
}

// Uso en controller:
UserDTO dto = userMapper.toDTO(user);
```

**Alternativas:**
- MapStruct (generación en compile-time)
- ModelMapper (reflexión en runtime)
- Manual (más control)

---

## 5. Dependency Injection e IoC

### 5.1 Inversión de Control (IoC)

**Sin IoC (control manual):**

```java
public class UserController {
    private UserService userService = new UserService();  // Tú creas
}

public class UserService {
    private UserRepository userRepository = new UserRepository();  // Tú creas
}
```

**Con IoC (control delegado a Spring):**

```java
@RestController
public class UserController {
    private final UserService userService;  // Spring inyecta

    public UserController(UserService userService) {
        this.userService = userService;
    }
}

@Service
public class UserService {
    private final UserRepository userRepository;  // Spring inyecta

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**Ventajas:**
- Desacoplamiento (no dependes de implementaciones concretas)
- Testabilidad (puedes inyectar mocks)
- Reutilización (Spring gestiona ciclo de vida)
- Configuración centralizada

### 5.2 Dependency Injection

**Tres tipos de inyección:**

```java
// 1. Constructor Injection (RECOMENDADO)
@Service
public class UserService {
    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
// Ventajas: Inmutable (final), testable, obligatorio

// 2. Field Injection (NO RECOMENDADO)
@Service
public class UserService {
    @Autowired
    private UserRepository repository;
}
// Desventajas: Mutable, difícil de testear, dependencias ocultas

// 3. Setter Injection (ocasional)
@Service
public class UserService {
    private UserRepository repository;

    @Autowired
    public void setRepository(UserRepository repository) {
        this.repository = repository;
    }
}
// Uso: Dependencias opcionales
```

### 5.3 Bean Lifecycle

```
1. Bean Definition
   @Component, @Service, @Repository, @Configuration

2. Bean Instantiation
   Spring crea instancia (usando constructor)

3. Dependency Injection
   Spring inyecta dependencias

4. Post-Initialization
   @PostConstruct methods

5. Bean Ready
   Bean disponible para uso

6. Pre-Destruction
   @PreDestroy methods

7. Bean Destroyed
   Aplicación se cierra
```

**Ejemplo:**

```java
@Service
public class UserService {

    @PostConstruct
    public void init() {
        System.out.println("UserService inicializado");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("UserService destruido");
    }
}
```

### 5.4 Bean Scopes

```java
// 1. Singleton (default) - Una instancia compartida
@Service
public class UserService { }

// 2. Prototype - Nueva instancia cada vez
@Service
@Scope("prototype")
public class ReportGenerator { }

// 3. Request - Una instancia por HTTP request
@Service
@Scope("request")
public class RequestContext { }

// 4. Session - Una instancia por HTTP session
@Service
@Scope("session")
public class ShoppingCart { }
```

**Uso más común:**
- Singleton: Services, Repositories (99% de los casos)
- Prototype: Objetos con estado mutable
- Request/Session: Web-specific beans

---

## 6. Spring Boot Actuator

### 6.1 ¿Qué es Actuator?

Actuator proporciona endpoints HTTP para:
- **Monitoreo**: Métricas de la aplicación
- **Health checks**: Estado de la aplicación
- **Management**: Administración en runtime

### 6.2 Endpoints Principales

| Endpoint | Descripción | Uso |
|----------|-------------|-----|
| `/actuator/health` | Estado de salud | Kubernetes liveness/readiness |
| `/actuator/info` | Información de la app | Versión, build info |
| `/actuator/metrics` | Métricas | CPU, memoria, requests |
| `/actuator/env` | Variables de entorno | Config values |
| `/actuator/loggers` | Configuración de logs | Cambiar nivel en runtime |
| `/actuator/threaddump` | Thread dump | Debugging |
| `/actuator/heapdump` | Heap dump | Memory analysis |

### 6.3 Health Indicators

**Built-in indicators:**

```yaml
# Health check respuesta
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
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500000000000,
        "free": 250000000000,
        "threshold": 10485760
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

**Custom health indicator:**

```java
@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            // Check external service
            boolean isUp = checkExternalService();

            if (isUp) {
                return Health.up()
                        .withDetail("service", "ExternalAPI")
                        .withDetail("responseTime", "50ms")
                        .build();
            } else {
                return Health.down()
                        .withDetail("service", "ExternalAPI")
                        .withDetail("error", "Timeout")
                        .build();
            }
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }

    private boolean checkExternalService() {
        // Implementation
        return true;
    }
}
```

### 6.4 Métricas con Micrometer

**Micrometer** = SLF4J para métricas (abstracción)

**Backends soportados:**
- Prometheus
- Grafana
- New Relic
- Datadog
- CloudWatch

**Tipos de métricas:**

```java
// 1. Counter - Solo incrementa
Counter counter = Counter.builder("tasks.created")
        .description("Total de tareas creadas")
        .register(registry);

counter.increment();

// 2. Gauge - Valor actual
Gauge.builder("tasks.active", () -> getActiveTasksCount())
        .register(registry);

// 3. Timer - Duración
Timer timer = Timer.builder("task.processing.time")
        .register(registry);

timer.record(() -> {
    // Operación a medir
    processTask();
});

// 4. Distribution Summary - Distribución de valores
DistributionSummary summary = DistributionSummary.builder("task.size")
        .register(registry);

summary.record(task.getSize());
```

### 6.5 Observabilidad

**Los 3 pilares:**

1. **Logs**: Qué pasó
2. **Metrics**: Cuánto/cuántas veces
3. **Traces**: Cómo fluye una request

**Stack completo:**
- Logs: ELK (Elasticsearch, Logstash, Kibana)
- Metrics: Prometheus + Grafana
- Traces: Zipkin, Jaeger (distributed tracing)

---

## Resumen

**Clase 1 cubrió:**

1. **Microservicios**: Arquitectura, ventajas, desafíos
2. **Spring Boot**: Auto-configuración, starters, embedded server
3. **REST API**: Recursos, métodos HTTP, códigos de estado
4. **Arquitectura en Capas**: Controller, Service, Repository
5. **Dependency Injection**: IoC, constructor injection, bean scopes
6. **Actuator**: Monitoreo, health checks, métricas

**Próximas clases:**
- **Clase 2**: Spring Data JPA, PostgreSQL, persistencia
- **Clase 3**: Validación, exception handling, profiles
- **Clase 4-6**: Apache Kafka, mensajería, event-driven

---

[← Volver a Clase 1](README.md)
