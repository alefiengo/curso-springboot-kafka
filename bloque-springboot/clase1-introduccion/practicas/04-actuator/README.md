# Práctica 4: Spring Boot Actuator y Configuración

**Nivel:** Principiante

---

## 1. Objetivo

Integrar Spring Boot Actuator para monitoreo y health checks, y aprender a configurar aplicaciones Spring Boot usando `application.yml` con diferentes perfiles (dev, prod).

**Al finalizar esta práctica podrás:**
- Agregar Spring Boot Actuator a un proyecto existente
- Exponer endpoints de monitoreo (health, info, metrics)
- Configurar la aplicación con `application.yml`
- Usar perfiles de Spring (dev, prod) para diferentes entornos
- Personalizar información de la aplicación

---

## 2. Comandos a Ejecutar

### Paso 1: Agregar Actuator al pom.xml

Abre tu proyecto `todo-api` de la Práctica 3 y agrega la dependencia:

**Archivo:** `pom.xml`

```xml
<dependencies>
    <!-- Dependencias existentes -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- NUEVA: Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### Paso 2: Crear application.yml

Elimina `application.properties` y crea `application.yml`:

**Archivo:** `src/main/resources/application.yml`

```yaml
# Configuración del servidor
server:
  port: 8080

# Configuración de la aplicación
spring:
  application:
    name: todo-api

# Configuración de Actuator
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env
  endpoint:
    health:
      show-details: always

# Información de la aplicación
info:
  app:
    name: TODO API
    description: API REST para gestión de tareas
    version: 1.0.0
    author: Tu Nombre
    contact:
      email: tu-email@ejemplo.com
```

### Paso 3: Crear perfiles (dev y prod)

**Archivo:** `src/main/resources/application-dev.yml`

```yaml
# Perfil de Desarrollo
server:
  port: 8080

spring:
  application:
    name: todo-api-dev

management:
  endpoints:
    web:
      exposure:
        include: '*'  # Expone TODOS los endpoints en dev
  endpoint:
    health:
      show-details: always

info:
  environment: development
  debug: true
```

**Archivo:** `src/main/resources/application-prod.yml`

```yaml
# Perfil de Producción
server:
  port: 9090

spring:
  application:
    name: todo-api-prod

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # Solo endpoints esenciales
  endpoint:
    health:
      show-details: when-authorized  # Oculta detalles sensibles

info:
  environment: production
  debug: false
```

### Paso 4: Ejecutar con diferentes perfiles

```bash
# Sin perfil específico (usa application.yml default)
mvn spring-boot:run

# Con perfil dev
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Con perfil prod
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

### Paso 5: Probar los endpoints de Actuator

```bash
# Health check
curl http://localhost:8080/actuator/health

# Información de la aplicación
curl http://localhost:8080/actuator/info

# Métricas
curl http://localhost:8080/actuator/metrics

# Métricas específicas
curl http://localhost:8080/actuator/metrics/http.server.requests

# Variables de entorno (solo en dev)
curl http://localhost:8080/actuator/env
```

**Respuestas esperadas:**

```json
// GET /actuator/health
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500000000000,
        "free": 250000000000,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}

// GET /actuator/info
{
  "app": {
    "name": "TODO API",
    "description": "API REST para gestión de tareas",
    "version": "1.0.0",
    "author": "Tu Nombre",
    "contact": {
      "email": "tu-email@ejemplo.com"
    }
  },
  "environment": "development",
  "debug": true
}

// GET /actuator/metrics
{
  "names": [
    "http.server.requests",
    "jvm.memory.used",
    "jvm.memory.max",
    "system.cpu.usage",
    "process.uptime"
  ]
}
```

---

## 3. Desglose del Comando

### Estructura de application.yml

| Sección | Propósito | Ejemplo |
|---------|-----------|---------|
| `server:` | Configuración del servidor web | `port: 8080` |
| `spring:` | Configuración de Spring Boot | `application.name` |
| `management:` | Configuración de Actuator | `endpoints.web.exposure` |
| `info:` | Metadata de la aplicación | `app.name`, `app.version` |

### Endpoints de Actuator

| Endpoint | Propósito | Información |
|----------|-----------|-------------|
| `/actuator` | Lista de endpoints disponibles | Links a todos los endpoints |
| `/actuator/health` | Estado de salud de la app | UP/DOWN, diskSpace, db, etc. |
| `/actuator/info` | Información de la app | Versión, autor, descripción |
| `/actuator/metrics` | Métricas de la aplicación | CPU, memoria, requests, etc. |
| `/actuator/env` | Variables de entorno | System props, application props |
| `/actuator/loggers` | Configuración de logs | Niveles de log por paquete |
| `/actuator/beans` | Beans de Spring | Todos los beans del contexto |

### Perfiles de Spring

| Perfil | Archivo | Uso |
|--------|---------|-----|
| Default | `application.yml` | Configuración base común |
| dev | `application-dev.yml` | Desarrollo local |
| prod | `application-prod.yml` | Producción |
| test | `application-test.yml` | Testing |

**Precedencia:** `application-{profile}.yml` sobrescribe `application.yml`

---

## 4. Explicación Detallada

### ¿Qué es Spring Boot Actuator?

Actuator proporciona endpoints HTTP para:
- **Monitoreo**: Ver métricas de la aplicación
- **Health checks**: Verificar si la app está funcionando
- **Management**: Administrar configuración en runtime
- **Observabilidad**: Integración con Prometheus, Grafana

**Casos de uso:**
- Kubernetes liveness/readiness probes
- Monitoreo con herramientas APM
- Debugging en producción
- CI/CD health checks

### YAML vs Properties

**application.properties:**
```properties
server.port=8080
spring.application.name=todo-api
management.endpoints.web.exposure.include=health,info
```

**application.yml:** (más legible, estructurado)
```yaml
server:
  port: 8080

spring:
  application:
    name: todo-api

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

**Ventajas de YAML:**
- Jerarquía visual clara
- Menos repetición
- Más legible para configuraciones complejas

### Perfiles de Spring Boot

Los perfiles permiten diferentes configuraciones por entorno:

```
application.yml          ← Base común
application-dev.yml      ← Sobrescribe para dev
application-prod.yml     ← Sobrescribe para prod
application-test.yml     ← Sobrescribe para tests
```

**Activar perfil:**

```bash
# Línea de comandos
java -jar app.jar --spring.profiles.active=prod

# Maven
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Variable de entorno
export SPRING_PROFILES_ACTIVE=prod
```

**En código:**
```java
@Profile("dev")
@Component
public class DevDataLoader {
    // Solo se carga en perfil dev
}
```

### Health Checks

```yaml
management:
  endpoint:
    health:
      show-details: always  # Muestra detalles completos
      # Opciones: never, when-authorized, always
```

**Respuesta con `show-details: always`:**
```json
{
  "status": "UP",
  "components": {
    "diskSpace": {"status": "UP", "details": {...}},
    "ping": {"status": "UP"}
  }
}
```

**Respuesta con `show-details: never`:**
```json
{
  "status": "UP"
}
```

### Exponiendo Endpoints

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'          # Todos (solo en dev)
        include: health,info  # Solo específicos (en prod)
        exclude: env,beans    # Excluir algunos
```

**Seguridad:**
- En **dev**: Expón todos (`'*'`) para debugging
- En **prod**: Solo expón los esenciales (health, info, metrics)
- Endpoints como `/env` pueden exponer secretos

### Métricas

```bash
# Listar todas las métricas
curl http://localhost:8080/actuator/metrics

# Métrica específica
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Con tags
curl http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/tasks
```

**Métricas útiles:**
- `jvm.memory.used` - Memoria usada
- `jvm.memory.max` - Memoria máxima
- `system.cpu.usage` - Uso de CPU
- `http.server.requests` - Requests HTTP
- `process.uptime` - Tiempo de ejecución

---

## 5. Conceptos Aprendidos

### Observabilidad

Los tres pilares de observabilidad:
1. **Logs** - Qué pasó (eventos)
2. **Metrics** - Cuánto/cuántas veces (números)
3. **Traces** - Cómo fluye una request (distribución)

Actuator proporciona principalmente métricas.

### Configuration Management

- **Externalización**: Configuración fuera del código
- **Perfiles**: Diferentes configs por entorno
- **Precedencia**: application-{profile}.yml > application.yml > defaults

### Health Checks en Microservicios

```
[Load Balancer]
    |
    | GET /actuator/health cada 10s
    v
[Service Instance 1] → status: UP
[Service Instance 2] → status: DOWN ← No recibe tráfico
[Service Instance 3] → status: UP
```

### Kubernetes Integration

```yaml
# Kubernetes deployment.yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
```

### Info Endpoint Personalizado

```yaml
info:
  app:
    name: @project.name@           # Desde pom.xml
    version: @project.version@     # Desde pom.xml
    description: Mi aplicación
  team:
    name: Equipo Backend
    slack: #backend-team
  git:
    branch: main
    commit: abc123
```

---

## 6. Troubleshooting

### Problema 1: "404 Not Found" en /actuator

**Causa:** Actuator no está en el classpath.

**Solución:** Verifica que la dependencia esté en `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Luego ejecuta:
```bash
mvn clean install
```

---

### Problema 2: /actuator/env no aparece

**Causa:** No está incluido en `exposure.include`

**Solución:** Agrega en `application-dev.yml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

---

### Problema 3: "Could not resolve placeholder" en @Value

**Error:**
```
Could not resolve placeholder 'app.name' in value "${app.name}"
```

**Causa:** Propiedad no definida en application.yml

**Solución:** Agrega la propiedad o usa default:
```java
@Value("${app.name:Default App Name}")
private String appName;
```

---

### Problema 4: Perfil no se activa

**Síntoma:** Los cambios en `application-dev.yml` no se aplican

**Verificar que el perfil esté activo:**
```bash
curl http://localhost:8080/actuator/env | grep "activeProfiles"
```

**Solución:** Asegúrate de pasar el parámetro:
```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

---

### Problema 5: Health check retorna DOWN

**Respuesta:**
```json
{"status":"DOWN","components":{"diskSpace":{"status":"DOWN"}}}
```

**Causa:** Espacio en disco bajo el umbral.

**Solución temporal (dev):** Aumenta el umbral
```yaml
management:
  health:
    diskspace:
      threshold: 1MB  # Muy bajo, solo para dev
```

---

## 7. Desafío Adicional

### Desafío 1: Agregar información de build

Agrega información de Git y timestamp de build:

**pom.xml:**
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>build-info</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

**Prueba:**
```bash
mvn clean package
java -jar target/todo-api-0.0.1-SNAPSHOT.jar
curl http://localhost:8080/actuator/info
```

---

### Desafío 2: Custom Health Indicator

Crea un health indicator personalizado:

```java
package dev.alefiengo.todoapi.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class CustomHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // Lógica personalizada
        int taskCount = getTaskCount();

        if (taskCount > 100) {
            return Health.down()
                    .withDetail("reason", "Demasiadas tareas")
                    .withDetail("taskCount", taskCount)
                    .build();
        }

        return Health.up()
                .withDetail("taskCount", taskCount)
                .build();
    }

    private int getTaskCount() {
        // Obtener el conteo real de tareas
        return 42;
    }
}
```

**Prueba:**
```bash
curl http://localhost:8080/actuator/health
```

---

### Desafío 3: Métricas personalizadas

Agrega un contador de tareas creadas:

```java
package dev.alefiengo.todoapi.controller;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/tasks")
public class TaskController {

    private final Counter taskCreatedCounter;

    public TaskController(MeterRegistry registry) {
        this.taskCreatedCounter = Counter.builder("tasks.created")
                .description("Total de tareas creadas")
                .register(registry);
    }

    @PostMapping
    public ResponseEntity<Task> createTask(@RequestBody Task task) {
        // ... lógica existente ...
        taskCreatedCounter.increment();
        return ResponseEntity.status(HttpStatus.CREATED).body(task);
    }
}
```

**Prueba:**
```bash
# Crear algunas tareas
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","completed":false}'

# Ver métrica
curl http://localhost:8080/actuator/metrics/tasks.created
```

---

## 8. Recursos Adicionales

### Documentación Oficial

- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints)
- [Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
- [Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)

### Herramientas de Monitoreo

- [Prometheus](https://prometheus.io/) - Métricas y alertas
- [Grafana](https://grafana.com/) - Dashboards
- [Spring Boot Admin](https://github.com/codecentric/spring-boot-admin) - UI para Actuator

### Tutoriales

- [Spring Boot Actuator Tutorial](https://www.baeldung.com/spring-boot-actuator)
- [Custom Metrics with Micrometer](https://www.baeldung.com/micrometer)

### Próximos Pasos

- **Práctica 5:** Arquitectura en capas (Service layer)
- **Clase 8:** Monitoreo avanzado con Prometheus y Grafana

---

## Código de Solución

El código completo está disponible en: `solucion/`

**Incluye:**
- Configuración completa de Actuator
- Perfiles dev y prod
- Custom health indicator
- Métricas personalizadas

---

**¡Excelente!** Tu aplicación ahora tiene monitoreo y configuración profesional.

[← Práctica 3](../03-todo-api/) | [Volver a Clase 1](../../README.md) | [Práctica 5 →](../05-arquitectura-capas/)
