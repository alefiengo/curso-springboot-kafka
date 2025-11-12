# Lab 01: Spring Boot Actuator

---

## 1. Objetivo

Agregar Spring Boot Actuator a los microservicios para exponer endpoints de monitoreo, health checks y métricas que permitan supervisar el estado de las aplicaciones en tiempo real.

---

## 2. Comandos a ejecutar

```bash
# Agregar dependencia a product-service
cd ~/workspace/product-service

# Editar pom.xml y agregar (después de otras dependencias)
cat >> pom.xml << 'EOF'
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
EOF

# Configurar Actuator en application.yml
cat >> src/main/resources/application.yml << 'EOF'

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
      enabled: true
EOF

# Reconstruir y ejecutar
mvn clean install
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Probar endpoints de Actuator
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/metrics
curl http://localhost:8080/actuator/info
curl http://localhost:8080/actuator/env
```

**Salida esperada** (health):

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
        "total": 250685575168,
        "free": 100000000000,
        "threshold": 10485760
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

---

## 3. Desglose del comando

| Elemento | Descripción |
|----------|-------------|
| `spring-boot-starter-actuator` | Dependencia que agrega endpoints de monitoreo |
| `management.endpoints.web.exposure.include` | Lista de endpoints a exponer públicamente |
| `health,info,metrics,env` | Endpoints básicos de monitoreo |
| `show-details: always` | Mostrar detalles completos del health check |
| `/actuator/health` | Estado de salud de la aplicación |
| `/actuator/metrics` | Métricas de la aplicación |

---

## 4. Explicación detallada

### 4.1 ¿Qué es Spring Boot Actuator?

Spring Boot Actuator proporciona **endpoints listos para producción** que permiten monitorear y administrar aplicaciones. Es esencial para:

- **Health checks**: Verificar si la aplicación está funcionando
- **Métricas**: CPU, memoria, requests HTTP, queries a BD
- **Información**: Versión, configuración, entorno
- **Debugging**: Variables de entorno, propiedades

### 4.2 Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Incluye automáticamente:
- Endpoints HTTP bajo `/actuator`
- Métricas con Micrometer
- Health indicators para DB, Kafka, disk space

### 4.3 Configuración en application.yml

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env  # Endpoints a exponer
  endpoint:
    health:
      show-details: always  # Mostrar detalles completos
  health:
    kafka:
      enabled: true  # Health check para Kafka
```

**IMPORTANTE**:
- Por defecto, solo `/health` está expuesto
- `include: '*'` expone TODOS los endpoints (NO recomendado en producción)
- En producción, proteger con JWT o IP whitelist

### 4.4 Endpoints Principales

#### /actuator/health

Verifica el estado de la aplicación y sus componentes:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL"
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

**Estados posibles**:
- `UP`: Todo funcionando correctamente
- `DOWN`: Componente crítico falló (BD, Kafka)
- `OUT_OF_SERVICE`: Servicio deshabilitado
- `UNKNOWN`: Estado desconocido

#### /actuator/metrics

Lista todas las métricas disponibles:

```json
{
  "names": [
    "jvm.memory.used",
    "jvm.memory.max",
    "http.server.requests",
    "jdbc.connections.active",
    "kafka.producer.count"
  ]
}
```

Para ver métrica específica:
```bash
curl http://localhost:8080/actuator/metrics/jvm.memory.used
```

#### /actuator/info

Información de la aplicación (requiere configuración adicional):

```yaml
info:
  app:
    name: Product Service
    description: Microservicio de gestión de productos
    version: 1.0.0
```

#### /actuator/env

Variables de entorno y propiedades de configuración.

**PRECAUCIÓN**: Puede exponer información sensible. Proteger en producción.

### 4.5 Health Check con Kafka

Para servicios que usan Kafka, Actuator verifica automáticamente la conexión:

```yaml
management:
  health:
    kafka:
      enabled: true
```

Si Kafka está DOWN:
```json
{
  "status": "DOWN",
  "components": {
    "kafka": {
      "status": "DOWN",
      "details": {
        "error": "Connection refused"
      }
    }
  }
}
```

### 4.6 Aplicar a Todos los Servicios

Repetir configuración en:

1. **product-service** (8080)
2. **order-service** (8081) - incluye health check de Kafka
3. **inventory-service** (8082) - incluye health check de Kafka
4. **analytics-service** (8083) - incluye health check de Kafka

**Verificar todos los health checks**:
```bash
curl http://localhost:8080/actuator/health
curl http://localhost:8081/actuator/health
curl http://localhost:8082/actuator/health
curl http://localhost:8083/actuator/health
```

---

## 5. Conceptos aprendidos

- **Spring Boot Actuator**: Framework de monitoreo y administración
- **Health checks**: Verificación del estado de componentes críticos
- **Métricas**: Información cuantitativa sobre rendimiento
- **Production-ready features**: Funcionalidades esenciales para producción
- **Health indicators**: Verificadores automáticos de BD, Kafka, disco
- **Micrometer**: Framework de métricas integrado en Spring Boot
- **Endpoint exposure**: Control de qué endpoints están disponibles públicamente

---

## 6. Troubleshooting

### Problema 1: Endpoints no disponibles

**Síntoma**: `404 Not Found` al acceder a `/actuator/health`.

**Causa**: Actuator no está configurado o no expone endpoints.

**Solución**: Verificar dependencia y configuración:

```xml
<!-- Verificar en pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# Verificar en application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

### Problema 2: Health check muestra DOWN

**Síntoma**: `"status": "DOWN"` en `/actuator/health`.

**Causa**: PostgreSQL o Kafka no están disponibles.

**Solución**: Verificar infraestructura:

```bash
# Verificar PostgreSQL
docker exec -it postgres psql -U postgres -c "SELECT 1"

# Verificar Kafka
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# Reiniciar servicios
docker compose restart postgres kafka
```

### Problema 3: Health details no se muestran

**Síntoma**: Solo muestra `{"status":"UP"}` sin detalles.

**Causa**: `show-details` no configurado.

**Solución**: Agregar en `application.yml`:

```yaml
management:
  endpoint:
    health:
      show-details: always
```

### Problema 4: Kafka health check falla

**Error**: `org.apache.kafka.common.errors.TimeoutException`

**Causa**: Kafka no está corriendo o no es accesible.

**Solución**:

```bash
# Verificar Kafka está UP
docker compose ps kafka

# Ver logs de Kafka
docker compose logs -f kafka

# Verificar conexión
docker exec -it kafka kafka-broker-api-versions --bootstrap-server localhost:9092
```

---

## 7. Desafío adicional

1. **Agrega métricas personalizadas** usando Micrometer:

```java
import io.micrometer.core.instrument.MeterRegistry;

@Service
public class ProductService {

    private final MeterRegistry meterRegistry;

    public ProductService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        meterRegistry.counter("products.created").increment(0); // Inicializar
    }

    public ProductResponse createProduct(ProductRequest request) {
        // Lógica...
        meterRegistry.counter("products.created").increment();
        return response;
    }
}
```

Verificar métrica:
```bash
curl http://localhost:8080/actuator/metrics/products.created
```

2. **Crea un health indicator personalizado** para verificar stock bajo:

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class InventoryHealthIndicator implements HealthIndicator {

    @Autowired
    private InventoryRepository inventoryRepository;

    @Override
    public Health health() {
        long lowStockCount = inventoryRepository.countByStockLessThan(10);

        if (lowStockCount > 5) {
            return Health.down()
                .withDetail("lowStockProducts", lowStockCount)
                .withDetail("message", "Too many products with low stock")
                .build();
        }

        return Health.up()
            .withDetail("lowStockProducts", lowStockCount)
            .build();
    }
}
```

3. **Integra con Prometheus** (opcional):

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

---

## 8. Recursos adicionales

- [Spring Boot Actuator Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Custom Health Indicators](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints.health.writing-custom-health-indicators)

---

**Siguiente**: [Lab 02: Variables de Entorno](../02-variables-entorno/README.md)
