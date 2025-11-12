# Lab 04: Logging Estructurado

---

## 1. Objetivo

Implementar logging estructurado con diferentes configuraciones para desarrollo y producción, facilitando el debugging en desarrollo y la trazabilidad en producción mediante logs organizados y persistentes.

---

## 2. Comandos a ejecutar

```bash
# Crear configuración de Logback
cd ~/workspace/product-service/src/main/resources

# Crear logback-spring.xml
cat > logback-spring.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- Perfil de desarrollo: logs a consola -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>

        <root level="INFO">
            <appender-ref ref="CONSOLE" />
        </root>

        <logger name="dev.alefiengo" level="DEBUG" />
        <logger name="org.springframework.web" level="DEBUG" />
    </springProfile>

    <!-- Perfil de producción: logs a archivo con rotación -->
    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/application.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/application-%d{yyyy-MM-dd}.log</fileNamePattern>
                <maxHistory>30</maxHistory>
                <totalSizeCap>1GB</totalSizeCap>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>

        <root level="INFO">
            <appender-ref ref="FILE" />
        </root>

        <logger name="dev.alefiengo" level="INFO" />
        <logger name="org.springframework.web" level="WARN" />
    </springProfile>

</configuration>
EOF

# Crear directorio de logs
mkdir -p logs

# Agregar logs/.gitkeep para mantener directorio en git
touch logs/.gitkeep

# Agregar logs al .gitignore
cat >> .gitignore << 'EOF'

# Logs
logs/*.log
EOF

# Ejecutar con perfil dev (logs a consola)
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Ejecutar con perfil prod (logs a archivo)
mvn spring-boot:run -Dspring-boot.run.profiles=prod

# Ver logs en tiempo real
tail -f logs/application.log
```

**Salida esperada** (consola dev):

```
2025-11-12 10:30:15 [main] INFO  d.a.productservice.ProductServiceApplication - Starting ProductServiceApplication
2025-11-12 10:30:16 [main] DEBUG d.a.productservice.config.SecurityConfig - Configuring security
2025-11-12 10:30:18 [main] INFO  d.a.productservice.ProductServiceApplication - Started ProductServiceApplication in 3.2 seconds
```

---

## 3. Desglose del comando

| Elemento | Descripción |
|----------|-------------|
| `logback-spring.xml` | Configuración de Logback con soporte para Spring Profiles |
| `<springProfile name="dev">` | Configuración específica para perfil dev |
| `ConsoleAppender` | Escribe logs a consola (stdout) |
| `RollingFileAppender` | Escribe logs a archivo con rotación automática |
| `TimeBasedRollingPolicy` | Rotación diaria de archivos de log |
| `maxHistory` | Número de días de logs a conservar |
| `%d{yyyy-MM-dd HH:mm:ss}` | Formato de fecha y hora |
| `%-5level` | Nivel de log (INFO, DEBUG, WARN, ERROR) |

---

## 4. Explicación detallada

### 4.1 ¿Qué es Logging Estructurado?

**Logging estructurado** organiza los logs de forma consistente con:

- **Formato estandarizado**: Timestamp, nivel, clase, mensaje
- **Niveles apropiados**: DEBUG, INFO, WARN, ERROR
- **Contexto relevante**: Usuario, transacción, request ID
- **Persistencia**: Archivos rotados automáticamente

**Beneficios**:
- Debugging más rápido en desarrollo
- Trazabilidad en producción
- Análisis de logs con herramientas (ELK, Splunk)
- Cumplimiento de auditoría

### 4.2 Niveles de Logging

| Nivel | Uso | Ejemplo |
|-------|-----|---------|
| **TRACE** | Información muy detallada | Parámetros de métodos, valores internos |
| **DEBUG** | Información de debugging | "Entering method", "Query executed" |
| **INFO** | Eventos informativos | "User logged in", "Order created" |
| **WARN** | Situaciones anómalas pero recuperables | "Retry attempt 3", "Deprecated API used" |
| **ERROR** | Errores que afectan funcionalidad | "Failed to connect to DB", "Exception" |

### 4.3 Configuración de Logback

#### Perfil Dev (logback-spring.xml)

```xml
<springProfile name="dev">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>

    <logger name="dev.alefiengo" level="DEBUG" />
    <logger name="org.springframework.web" level="DEBUG" />
</springProfile>
```

**Características**:
- Logs a consola (visible en terminal)
- Nivel DEBUG para código propio
- Nivel DEBUG para Spring Web (ver requests HTTP)
- Formato simple y legible

#### Perfil Prod (logback-spring.xml)

```xml
<springProfile name="prod">
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="FILE" />
    </root>

    <logger name="dev.alefiengo" level="INFO" />
</springProfile>
```

**Características**:
- Logs a archivo (`logs/application.log`)
- Rotación diaria (nuevo archivo cada día)
- Conserva 30 días de logs
- Máximo 1GB total
- Nivel INFO (menos verbose)

### 4.4 Patrón de Log Explicado

```
%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
```

| Elemento | Descripción | Ejemplo |
|----------|-------------|---------|
| `%d{yyyy-MM-dd HH:mm:ss}` | Fecha y hora | `2025-11-12 10:30:15` |
| `[%thread]` | Nombre del thread | `[http-nio-8080-exec-1]` |
| `%-5level` | Nivel de log (5 chars) | `INFO ` |
| `%logger{36}` | Nombre de la clase (max 36 chars) | `d.a.p.service.ProductService` |
| `%msg` | Mensaje del log | `Product created: id=1` |
| `%n` | Nueva línea | |

**Salida**:
```
2025-11-12 10:30:15 [http-nio-8080-exec-1] INFO  d.a.p.service.ProductService - Product created: id=1
```

### 4.5 Implementar Logging en Código

#### ProductService.java

```java
package dev.alefiengo.productservice.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    private static final Logger log = LoggerFactory.getLogger(ProductService.class);

    public ProductResponse createProduct(ProductRequest request) {
        log.info("Creating product: name={}, price={}", request.getName(), request.getPrice());

        try {
            // Lógica de creación...
            Product product = new Product();
            product.setName(request.getName());
            product.setPrice(request.getPrice());

            Product saved = productRepository.save(product);

            log.info("Product created successfully: id={}, name={}", saved.getId(), saved.getName());

            return mapper.toResponse(saved);

        } catch (Exception e) {
            log.error("Failed to create product: name={}, error={}",
                request.getName(), e.getMessage(), e);
            throw e;
        }
    }

    public ProductResponse getProductById(Long id) {
        log.debug("Fetching product by id: {}", id);

        Product product = productRepository.findById(id)
            .orElseThrow(() -> {
                log.warn("Product not found: id={}", id);
                return new ResourceNotFoundException("Product not found: " + id);
            });

        log.debug("Product found: id={}, name={}", product.getId(), product.getName());

        return mapper.toResponse(product);
    }
}
```

#### OrderEventProducer.java

```java
package dev.alefiengo.orderservice.kafka.producer;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class OrderEventProducer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventProducer.class);

    public void publishOrderPlaced(OrderPlacedEvent event) {
        log.info("Publishing order placed event: orderId={}, productId={}, quantity={}",
            event.getOrderId(), event.getProductId(), event.getQuantity());

        try {
            kafkaTemplate.send("ecommerce.orders.placed", event.getOrderId(), event);
            log.info("Order placed event published successfully: orderId={}", event.getOrderId());

        } catch (Exception e) {
            log.error("Failed to publish order placed event: orderId={}, error={}",
                event.getOrderId(), e.getMessage(), e);
            throw e;
        }
    }
}
```

### 4.6 Logs en Diferentes Escenarios

#### Escenario 1: Request HTTP exitoso

```
2025-11-12 10:30:15 [http-nio-8080-exec-1] DEBUG o.s.web.servlet.DispatcherServlet - POST "/api/products"
2025-11-12 10:30:15 [http-nio-8080-exec-1] INFO  d.a.p.service.ProductService - Creating product: name=Laptop, price=1500.0
2025-11-12 10:30:15 [http-nio-8080-exec-1] DEBUG o.h.SQL - insert into products (name, price) values (?, ?)
2025-11-12 10:30:15 [http-nio-8080-exec-1] INFO  d.a.p.service.ProductService - Product created successfully: id=1, name=Laptop
2025-11-12 10:30:15 [http-nio-8080-exec-1] DEBUG o.s.web.servlet.DispatcherServlet - Completed 201 CREATED
```

#### Escenario 2: Producto no encontrado

```
2025-11-12 10:31:20 [http-nio-8080-exec-2] DEBUG d.a.p.service.ProductService - Fetching product by id: 999
2025-11-12 10:31:20 [http-nio-8080-exec-2] WARN  d.a.p.service.ProductService - Product not found: id=999
2025-11-12 10:31:20 [http-nio-8080-exec-2] ERROR d.a.p.exception.GlobalExceptionHandler - ResourceNotFoundException: Product not found: 999
```

#### Escenario 3: Error de Kafka

```
2025-11-12 10:32:10 [http-nio-8081-exec-1] INFO  d.a.o.kafka.OrderEventProducer - Publishing order placed event: orderId=123
2025-11-12 10:32:10 [http-nio-8081-exec-1] ERROR d.a.o.kafka.OrderEventProducer - Failed to publish order placed event: orderId=123, error=Connection refused
org.apache.kafka.common.errors.TimeoutException: Topic ecommerce.orders.placed not present in metadata after 60000 ms.
```

### 4.7 Rotación de Logs en Producción

Con `TimeBasedRollingPolicy`:

```
logs/
├── application.log              # Archivo actual
├── application-2025-11-11.log   # Ayer
├── application-2025-11-10.log   # Hace 2 días
└── application-2025-11-09.log   # Hace 3 días
```

Después de 30 días, los logs más antiguos se eliminan automáticamente.

---

## 5. Conceptos aprendidos

- **Logging estructurado**: Formato consistente y organizado
- **Logback**: Framework de logging recomendado por Spring Boot
- **SLF4J**: API de logging (abstracción sobre Logback)
- **Appenders**: Destinos de logs (consola, archivo, base de datos)
- **Niveles de log**: TRACE, DEBUG, INFO, WARN, ERROR
- **Rolling policy**: Rotación automática de archivos de log
- **Pattern layout**: Formato de mensajes de log
- **Logger por clase**: `LoggerFactory.getLogger(MyClass.class)`

---

## 6. Troubleshooting

### Problema 1: Logs no aparecen en consola

**Síntoma**: No se ven logs al ejecutar aplicación.

**Causa**: Nivel de log muy restrictivo.

**Solución**: Verificar configuración en `logback-spring.xml`:

```xml
<root level="INFO">  <!-- Cambiar a DEBUG si necesario -->
    <appender-ref ref="CONSOLE" />
</root>
```

O en `application.yml`:

```yaml
logging:
  level:
    root: INFO
    dev.alefiengo: DEBUG
```

### Problema 2: Archivo de log no se crea

**Síntoma**: No existe `logs/application.log`.

**Causa 1**: Directorio `logs/` no existe.

**Solución**: Crear directorio:

```bash
mkdir -p logs
```

**Causa 2**: Perfil incorrecto activo.

**Solución**: Verificar perfil:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

### Problema 3: Demasiados logs en producción

**Síntoma**: Archivos de log crecen rápidamente.

**Solución**: Reducir nivel de log a INFO/WARN:

```xml
<logger name="dev.alefiengo" level="INFO" />
<logger name="org.springframework" level="WARN" />
<logger name="org.hibernate" level="WARN" />
```

### Problema 4: No se ven logs de Hibernate SQL

**Síntoma**: No aparecen queries SQL.

**Solución**: Habilitar en `application.yml`:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

---

## 7. Desafío adicional

1. **Implementa logs en formato JSON** para ELK Stack:

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

```xml
<springProfile name="prod">
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>
</springProfile>
```

2. **Agrega MDC (Mapped Diagnostic Context)** para request tracing:

```java
import org.slf4j.MDC;

@Component
public class RequestTracingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

Pattern:
```xml
<pattern>%d{yyyy-MM-dd HH:mm:ss} [%X{requestId}] [%thread] %-5level %logger{36} - %msg%n</pattern>
```

3. **Envía logs a sistema centralizado** (Papertrail, Loggly):

```xml
<appender name="SYSLOG" class="ch.qos.logback.classic.net.SyslogAppender">
    <syslogHost>logs.papertrailapp.com</syslogHost>
    <port>12345</port>
    <facility>USER</facility>
    <suffixPattern>[%thread] %logger %msg</suffixPattern>
</appender>
```

---

## 8. Recursos adicionales

- [Logback Manual](https://logback.qos.ch/manual/index.html)
- [SLF4J Documentation](https://www.slf4j.org/manual.html)
- [Spring Boot Logging](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)
- [Logback Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging.logback-extensions)

---

**Siguiente**: [Lab 05: Consolidación Final](../05-consolidacion/README.md)
