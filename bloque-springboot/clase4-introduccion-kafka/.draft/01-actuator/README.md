# Lab 01: Spring Boot Actuator

> **Nota**: Este laboratorio se verá en la **Clase 8** como parte de la preparación para producción. Si estás siguiendo el curso en orden, continúa con el [Lab 00: Conceptos de Kafka](../labs/00-conceptos-kafka/).

---

## Objetivo

Implementar Spring Boot Actuator en `product-service` para exponer endpoints de monitoreo y gestión que permitan verificar el estado de salud, métricas de rendimiento e información de la aplicación en tiempo real.

---

## Comandos a ejecutar

```bash
# 1. Navegar al directorio del proyecto
cd product-service

# 2. Agregar dependencia de Actuator al pom.xml
# Abrir pom.xml y agregar dentro de <dependencies>:
#
# <dependency>
#     <groupId>org.springframework.boot</groupId>
#     <artifactId>spring-boot-starter-actuator</artifactId>
# </dependency>

# 3. Configurar endpoints en application.yml
# Abrir application.yml y agregar al final:
#
# management:
#   endpoints:
#     web:
#       exposure:
#         include: health,info,metrics,env
#   endpoint:
#     health:
#       show-details: always

# 4. Recompilar el proyecto
mvn clean install

# 5. Ejecutar la aplicación
mvn spring-boot:run

# 6. Verificar endpoints disponibles de Actuator
curl http://localhost:8080/actuator

# 7. Consultar estado de salud (health check)
curl http://localhost:8080/actuator/health

# 8. Consultar información de la aplicación
curl http://localhost:8080/actuator/info

# 9. Consultar métricas disponibles
curl http://localhost:8080/actuator/metrics

# 10. Consultar métrica específica: memoria JVM utilizada
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# 11. Consultar métrica específica: total de requests HTTP
curl http://localhost:8080/actuator/metrics/http.server.requests

# 12. Consultar variables de entorno (cuidado en producción)
curl http://localhost:8080/actuator/env
```

**Salida esperada**:

### GET `/actuator`

```json
{
  "_links": {
    "self": { "href": "http://localhost:8080/actuator", "templated": false },
    "health": { "href": "http://localhost:8080/actuator/health", "templated": false },
    "info": { "href": "http://localhost:8080/actuator/info", "templated": false },
    "metrics": { "href": "http://localhost:8080/actuator/metrics", "templated": false },
    "env": { "href": "http://localhost:8080/actuator/env", "templated": false }
  }
}
```

### GET `/actuator/health`

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
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 499963174912,
        "free": 212055158784,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

### GET `/actuator/metrics/jvm.memory.used`

```json
{
  "name": "jvm.memory.used",
  "description": "The amount of used memory",
  "baseUnit": "bytes",
  "measurements": [
    { "statistic": "VALUE", "value": 157286400 }
  ],
  "availableTags": [
    { "tag": "area", "values": ["heap", "nonheap"] },
    { "tag": "id", "values": ["G1 Eden Space", "G1 Old Gen"] }
  ]
}
```

---

## Desglose del comando

### Comando: `curl http://localhost:8080/actuator/health`

| Componente | Descripción |
|------------|-------------|
| `curl` | Herramienta de línea de comandos para hacer requests HTTP |
| `http://localhost:8080` | URL base de la aplicación Spring Boot |
| `/actuator` | Base path de todos los endpoints de Actuator |
| `/health` | Endpoint específico que muestra el estado de salud de la aplicación |

**Otros endpoints útiles**:

| Endpoint | Descripción | Uso |
|----------|-------------|-----|
| `/actuator` | Lista todos los endpoints disponibles | Descubrimiento de funcionalidades |
| `/actuator/health` | Estado de salud (DB, disk, etc.) | Readiness/liveness probes en K8s |
| `/actuator/info` | Información de la aplicación (versión, git) | Verificar versión desplegada |
| `/actuator/metrics` | Lista de métricas disponibles | Identificar métricas a monitorear |
| `/actuator/metrics/{metric}` | Métrica específica | Obtener valores de rendimiento |
| `/actuator/env` | Variables de entorno y propiedades | Debugging de configuración |
| `/actuator/loggers` | Configuración de loggers | Cambiar nivel de log en tiempo real |

---

## Explicación detallada

### Paso 1: Agregar dependencia de Actuator

Abre `pom.xml` y agrega la siguiente dependencia dentro de `<dependencies>`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Esta dependencia incluye:

- Endpoints de monitoreo (health, metrics, info)
- Health indicators para componentes comunes (DB, disk)
- Sistema de métricas basado en Micrometer
- Soporte para integración con sistemas de monitoreo (Prometheus, Grafana)

### Paso 2: Configurar endpoints en application.yml

Por defecto, Actuator solo expone los endpoints `/health` e `/info` por razones de seguridad. Para habilitar más endpoints, agregar a `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env
  endpoint:
    health:
      show-details: always
  info:
    env:
      enabled: true
```

**Explicación de cada propiedad**:

- `management.endpoints.web.exposure.include`: Lista de endpoints a exponer vía HTTP
  - `health`: Estado de salud de la aplicación
  - `info`: Información de la aplicación
  - `metrics`: Métricas de rendimiento
  - `env`: Variables de entorno

- `management.endpoint.health.show-details: always`: Muestra detalles completos del health check (DB, disk, etc.)
  - `never`: No muestra detalles
  - `when-authorized`: Solo para usuarios autenticados
  - `always`: Siempre muestra detalles

- `management.info.env.enabled: true`: Permite agregar información personalizada desde propiedades

**ADVERTENCIA DE SEGURIDAD**: En producción, NO exponer `/env` públicamente ya que puede revelar credenciales. Usar Spring Security para proteger endpoints sensibles.

### Paso 3: Agregar información personalizada a /actuator/info

Puedes agregar información personalizada al endpoint `/info` en `application.yml`:

```yaml
info:
  app:
    name: product-service
    description: Microservicio de catálogo de productos
    version: 1.0.0
  contact:
    email: dev@iquattro.com
    team: Backend Team
```

Esta información será visible en `GET /actuator/info`:

```json
{
  "app": {
    "name": "product-service",
    "description": "Microservicio de catálogo de productos",
    "version": "1.0.0"
  },
  "contact": {
    "email": "dev@iquattro.com",
    "team": "Backend Team"
  }
}
```

### Paso 4: Entender el endpoint /actuator/health

El endpoint `/health` verifica el estado de componentes críticos automáticamente:

#### Health Indicators incorporados

| Indicator | Verifica | Estado UP cuando |
|-----------|----------|------------------|
| `db` | Conexión a base de datos | Puede ejecutar `isValid()` en conexión JDBC |
| `diskSpace` | Espacio en disco | Espacio libre > threshold (10 MB por defecto) |
| `ping` | Aplicación corriendo | Siempre UP si la aplicación responde |

**Ejemplo de salida con DB DOWN**:

```json
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN",
      "details": {
        "error": "org.postgresql.util.PSQLException: Connection refused"
      }
    }
  }
}
```

#### Códigos HTTP del health check

- `200 OK` → Status: UP
- `503 Service Unavailable` → Status: DOWN

**Uso en Kubernetes** (readiness/liveness probes):

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
```

### Paso 5: Explorar métricas disponibles

El endpoint `/actuator/metrics` lista todas las métricas que Actuator recopila:

```json
{
  "names": [
    "jvm.memory.used",
    "jvm.memory.max",
    "jvm.threads.live",
    "http.server.requests",
    "logback.events",
    "system.cpu.usage",
    "process.uptime"
  ]
}
```

#### Métricas más útiles

**Memoria JVM**:

```bash
curl http://localhost:8080/actuator/metrics/jvm.memory.used
curl http://localhost:8080/actuator/metrics/jvm.memory.max
```

**Threads**:

```bash
curl http://localhost:8080/actuator/metrics/jvm.threads.live
curl http://localhost:8080/actuator/metrics/jvm.threads.peak
```

**Requests HTTP**:

```bash
# Total de requests
curl http://localhost:8080/actuator/metrics/http.server.requests

# Requests a un endpoint específico
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/products"

# Requests por método HTTP
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=method:GET"

# Requests con status code 200
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=status:200"
```

**CPU y Sistema**:

```bash
curl http://localhost:8080/actuator/metrics/system.cpu.usage
curl http://localhost:8080/actuator/metrics/process.uptime
```

### Paso 6: Usar métricas para troubleshooting

#### Detectar alto uso de memoria

```bash
curl http://localhost:8080/actuator/metrics/jvm.memory.used
```

Si el valor está cerca del máximo (`jvm.memory.max`), puede haber memory leaks.

#### Detectar endpoints lentos

```bash
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/products"
```

Revisar el campo `max` en la respuesta:

```json
{
  "measurements": [
    { "statistic": "COUNT", "value": 150 },
    { "statistic": "TOTAL_TIME", "value": 12.5 },
    { "statistic": "MAX", "value": 0.8 }
  ]
}
```

- `MAX: 0.8` → El request más lento tardó 0.8 segundos

#### Detectar errores HTTP

```bash
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=status:500"
```

Si hay valores en `COUNT`, significa que hubo errores 500.

### Paso 7: Cambiar nivel de logging en tiempo real

El endpoint `/actuator/loggers` permite modificar el nivel de log sin reiniciar la aplicación.

**Ver nivel actual**:

```bash
curl http://localhost:8080/actuator/loggers/dev.alefiengo
```

**Respuesta**:

```json
{
  "configuredLevel": "INFO",
  "effectiveLevel": "INFO"
}
```

**Cambiar a DEBUG**:

```bash
curl -X POST http://localhost:8080/actuator/loggers/dev.alefiengo \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

Ahora los logs de `dev.alefiengo.*` mostrarán nivel DEBUG sin reiniciar.

### Paso 8: Diferenciar configuración por perfil

Puedes tener configuraciones de Actuator diferentes por perfil:

**application-dev.yml**:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"  # Todos los endpoints en desarrollo
  endpoint:
    health:
      show-details: always
```

**application-prod.yml**:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # Solo endpoints seguros en producción
  endpoint:
    health:
      show-details: when-authorized  # Detalles solo para usuarios autenticados
```

---

## Conceptos aprendidos

- **Spring Boot Actuator**: Conjunto de endpoints para monitoreo y gestión de aplicaciones
- **Health checks**: Verificación automática del estado de componentes (DB, disk, etc.)
- **Métricas**: Recopilación de datos de rendimiento (memoria, CPU, requests HTTP)
- **Endpoint /info**: Información personalizada de la aplicación
- **Configuración por perfil**: Diferentes niveles de exposición según entorno
- **Readiness/Liveness probes**: Integración con Kubernetes para gestión de pods
- **Troubleshooting en producción**: Cambiar log levels sin reiniciar aplicación
- **Seguridad**: NO exponer endpoints sensibles públicamente en producción

---

## Troubleshooting

### Problema 1: Endpoints no disponibles (404)

**Síntoma**:

```bash
curl http://localhost:8080/actuator/metrics
# HTTP/1.1 404 Not Found
```

**Causa**: Endpoints no habilitados en configuración.

**Solución**:

Verificar `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # Asegurar que "metrics" está aquí
```

### Problema 2: /actuator/health no muestra detalles

**Síntoma**:

```json
{
  "status": "UP"
}
```

No muestra `components`.

**Causa**: `show-details` no configurado.

**Solución**:

Agregar a `application.yml`:

```yaml
management:
  endpoint:
    health:
      show-details: always
```

### Problema 3: DB health check muestra DOWN

**Síntoma**:

```json
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN",
      "details": {
        "error": "Connection refused"
      }
    }
  }
}
```

**Causa**: PostgreSQL no está corriendo o la aplicación no puede conectarse.

**Solución**:

```bash
# Verificar que PostgreSQL esté corriendo
docker compose ps

# Si no está corriendo, iniciarlo
docker compose up -d postgres

# Verificar conectividad desde el host
docker exec -it postgres psql -U postgres -c "SELECT 1;"
```

### Problema 4: Métricas vacías o con valor 0

**Síntoma**:

```bash
curl http://localhost:8080/actuator/metrics/http.server.requests
# COUNT: 0
```

**Causa**: No se han hecho requests HTTP desde que la aplicación inició.

**Solución**:

Hacer algunos requests primero:

```bash
curl http://localhost:8080/api/products
curl http://localhost:8080/api/products/1

# Ahora consultar métrica
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/products"
```

### Problema 5: /actuator/env expone credenciales

**Síntoma**: Al hacer `curl http://localhost:8080/actuator/env`, aparecen contraseñas en texto claro.

**Causa**: Actuator NO oculta propiedades sensibles por defecto en todas las versiones.

**Solución en producción**:

1. **NO exponer `/env` públicamente**:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # NO incluir "env"
```

2. **Proteger con Spring Security**:

```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        http
            .requestMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic();
        return http.build();
    }
}
```

### Problema 6: Alto uso de memoria reportado

**Síntoma**:

```bash
curl http://localhost:8080/actuator/metrics/jvm.memory.used
# VALUE: 1800000000 (1.8 GB)
```

**Causa**: Posible memory leak o configuración de heap insuficiente.

**Solución**:

1. **Aumentar heap size**:

```bash
java -Xms512m -Xmx2g -jar product-service.jar
```

2. **Generar heap dump para análisis**:

```bash
jmap -dump:live,format=b,file=heap.bin <PID>
```

3. **Analizar con herramientas**:
   - Eclipse MAT (Memory Analyzer Tool)
   - VisualVM
   - JProfiler

---

## Desafío adicional

### Desafío 1: Crear custom health indicator

Crea un health indicator personalizado que verifique si hay al menos un producto en la base de datos.

**ProductHealthIndicator.java**:

```java
package dev.alefiengo.productservice.config;

import dev.alefiengo.productservice.repository.ProductRepository;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class ProductHealthIndicator implements HealthIndicator {

    private final ProductRepository productRepository;

    public ProductHealthIndicator(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Override
    public Health health() {
        long productCount = productRepository.count();
        if (productCount > 0) {
            return Health.up()
                .withDetail("productCount", productCount)
                .withDetail("status", "Catálogo inicializado")
                .build();
        } else {
            return Health.down()
                .withDetail("productCount", 0)
                .withDetail("status", "Catálogo vacío - requiere inicialización")
                .build();
        }
    }
}
```

**Resultado en `/actuator/health`**:

```json
{
  "status": "UP",
  "components": {
    "product": {
      "status": "UP",
      "details": {
        "productCount": 15,
        "status": "Catálogo inicializado"
      }
    },
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" }
  }
}
```

### Desafío 2: Crear custom metric

Crea una métrica personalizada que cuente cuántos productos se crean.

**ProductService.java** (modificar método `create`):

```java
package dev.alefiengo.productservice.service;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final Counter productCreatedCounter;

    public ProductService(ProductRepository productRepository, MeterRegistry meterRegistry) {
        this.productRepository = productRepository;
        this.productCreatedCounter = Counter.builder("products.created")
            .description("Total de productos creados")
            .register(meterRegistry);
    }

    @Transactional
    public ProductResponse create(ProductRequest request) {
        // Lógica existente...
        Product product = productRepository.save(newProduct);

        // Incrementar contador
        productCreatedCounter.increment();

        return ProductMapper.toResponse(product);
    }
}
```

**Consultar métrica**:

```bash
curl http://localhost:8080/actuator/metrics/products.created
```

**Resultado**:

```json
{
  "name": "products.created",
  "description": "Total de productos creados",
  "measurements": [
    { "statistic": "COUNT", "value": 23.0 }
  ]
}
```

---

## Recursos adicionales

- [Spring Boot Actuator - Documentación oficial](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Baeldung - Spring Boot Actuator](https://www.baeldung.com/spring-boot-actuators)
- [Custom Health Indicators](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints.health.writing-custom-health-indicators)
- [Kubernetes Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
