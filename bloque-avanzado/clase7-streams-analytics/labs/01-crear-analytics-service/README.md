# Lab 01: Crear analytics-service

---

## 1. Objetivo

Crear el cuarto microservicio (`analytics-service`) con dependencia `kafka-streams`, estructura básica, y configuración necesaria para procesar streams de Kafka. Este servicio NO usará `@KafkaListener`, sino la API de Kafka Streams.

---

## 2. Comandos a ejecutar

```bash
# Generar proyecto con Spring Initializr
curl https://start.spring.io/starter.zip \
  -d dependencies=web,kafka,kafka-streams \
  -d groupId=dev.alefiengo \
  -d artifactId=analytics-service \
  -d name=AnalyticsService \
  -d packageName=dev.alefiengo.analyticsservice \
  -d javaVersion=17 \
  -o analytics-service.zip

# Descomprimir y navegar al proyecto
unzip analytics-service.zip
cd analytics-service

# Compilar proyecto
mvn clean install

# Ejecutar servicio
mvn spring-boot:run
```

**Salida esperada**:

```
Started AnalyticsServiceApplication in 3.2 seconds
Kafka Streams version: 3.6.1
StreamThread started
State transition from CREATED to RUNNING
```

---

## 3. Desglose del comando

| Parámetro | Descripción |
|-----------|-------------|
| `dependencies=web,kafka,kafka-streams` | Incluye `spring-boot-starter-web`, `spring-kafka` y `kafka-streams` |
| `groupId=dev.alefiengo` | Identificador del grupo (estándar del curso) |
| `artifactId=analytics-service` | Nombre del artefacto (JAR generado) |
| `name=AnalyticsService` | Nombre de la clase principal |
| `packageName=dev.alefiengo.analyticsservice` | Paquete base de Java |
| `javaVersion=17` | Versión de Java (LTS) |

---

## 4. Explicación detallada

### 4.1 Verificar dependencias en pom.xml

Abre `pom.xml` y verifica que contiene:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
</dependency>
```

**Importante**: `kafka-streams` es diferente de `spring-kafka`. Ambas son necesarias.

### 4.2 Configurar application.yml

Crea/edita `src/main/resources/application.yml`:

```yaml
spring:
  application:
    name: analytics-service

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    streams:
      application-id: analytics-service
      properties:
        default.key.serde: org.apache.kafka.common.serialization.Serdes$StringSerde
        default.value.serde: org.apache.kafka.common.serialization.Serdes$StringSerde
        commit.interval.ms: 1000
        state.dir: /tmp/kafka-streams

server:
  port: 8083

logging:
  level:
    dev.alefiengo: INFO
    org.apache.kafka.streams: INFO
```

**Puntos críticos**:

- **`application-id`**: Identifica el consumer group y state stores. **DEBE ser único**.
- **`default.key.serde`**: Serializador/deserializador por defecto para keys (String).
- **`default.value.serde`**: Serializador por defecto para values (sobrescribiremos para JSON).
- **`commit.interval.ms`**: Frecuencia de commit de offsets (1 segundo).
- **`state.dir`**: Directorio local donde se guardan state stores.
- **`port: 8083`**: Cuarto microservicio usa puerto 8083 (8080, 8081, 8082 ya ocupados).

### 4.3 Crear estructura de paquetes

```bash
cd src/main/java/dev/alefiengo/analyticsservice/

mkdir -p config
mkdir -p streams
mkdir -p model/event
mkdir -p model/dto
mkdir -p controller
mkdir -p service
```

**Estructura completa**:

```
src/main/java/dev/alefiengo/analyticsservice/
├── AnalyticsServiceApplication.java
├── config/
├── streams/             # Kafka Streams (KStream, KTable)
├── model/
│   ├── event/           # Eventos de Kafka (OrderConfirmedEvent, etc.)
│   └── dto/             # DTOs para REST API
├── controller/          # REST controllers para consultar analytics
└── service/             # Servicios para consultar state stores
```

### 4.4 Copiar eventos desde otros microservicios

Crea `src/main/java/dev/alefiengo/analyticsservice/model/event/OrderConfirmedEvent.java`:

```java
package dev.alefiengo.analyticsservice.model.event;

import java.time.Instant;

public class OrderConfirmedEvent {
    private Long orderId;
    private Long productId;
    private Integer quantity;
    private Integer availableStockAfterReservation;
    private Integer reservedStockAfterReservation;
    private Instant timestamp;

    // Constructor vacío OBLIGATORIO para deserialización
    public OrderConfirmedEvent() {}

    public OrderConfirmedEvent(Long orderId, Long productId, Integer quantity,
                                Integer availableStockAfterReservation,
                                Integer reservedStockAfterReservation, Instant timestamp) {
        this.orderId = orderId;
        this.productId = productId;
        this.quantity = quantity;
        this.availableStockAfterReservation = availableStockAfterReservation;
        this.reservedStockAfterReservation = reservedStockAfterReservation;
        this.timestamp = timestamp;
    }

    // Getters y Setters
    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public Integer getQuantity() {
        return quantity;
    }

    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }

    public Integer getAvailableStockAfterReservation() {
        return availableStockAfterReservation;
    }

    public void setAvailableStockAfterReservation(Integer availableStockAfterReservation) {
        this.availableStockAfterReservation = availableStockAfterReservation;
    }

    public Integer getReservedStockAfterReservation() {
        return reservedStockAfterReservation;
    }

    public void setReservedStockAfterReservation(Integer reservedStockAfterReservation) {
        this.reservedStockAfterReservation = reservedStockAfterReservation;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }

    @Override
    public String toString() {
        return "OrderConfirmedEvent{" +
                "orderId=" + orderId +
                ", productId=" + productId +
                ", quantity=" + quantity +
                ", availableStockAfterReservation=" + availableStockAfterReservation +
                ", reservedStockAfterReservation=" + reservedStockAfterReservation +
                ", timestamp=" + timestamp +
                '}';
    }
}
```

Crea `src/main/java/dev/alefiengo/analyticsservice/model/event/OrderCancelledEvent.java`:

```java
package dev.alefiengo.analyticsservice.model.event;

import java.time.Instant;

public class OrderCancelledEvent {
    private Long orderId;
    private Long productId;
    private Integer quantity;
    private String reason;
    private Instant timestamp;

    // Constructor vacío OBLIGATORIO
    public OrderCancelledEvent() {}

    public OrderCancelledEvent(Long orderId, Long productId, Integer quantity,
                                String reason, Instant timestamp) {
        this.orderId = orderId;
        this.productId = productId;
        this.quantity = quantity;
        this.reason = reason;
        this.timestamp = timestamp;
    }

    // Getters y Setters
    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public Integer getQuantity() {
        return quantity;
    }

    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }

    public String getReason() {
        return reason;
    }

    public void setReason(String reason) {
        this.reason = reason;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }

    @Override
    public String toString() {
        return "OrderCancelledEvent{" +
                "orderId=" + orderId +
                ", productId=" + productId +
                ", quantity=" + quantity +
                ", reason='" + reason + '\'' +
                ", timestamp=" + timestamp +
                '}';
    }
}
```

**IMPORTANTE**: Las clases de eventos deben ser **exactamente iguales** a las de `order-service` e `inventory-service` para que la deserialización funcione correctamente.

### 4.5 Compilar y ejecutar

```bash
mvn clean install
mvn spring-boot:run
```

**Verifica en los logs**:

```
Started AnalyticsServiceApplication in 3.2 seconds (JVM running for 3.8)
Kafka Streams version: 3.6.1
StreamThread [analytics-service-stream-thread] started
State transition from CREATED to RUNNING
```

Si ves `State transition to RUNNING`, el servicio está correctamente configurado.

### 4.6 Verificar puerto (opcional)

```bash
curl http://localhost:8083/actuator/health
```

**Salida esperada**:

```json
{"status":"UP"}
```

Si agregaste `spring-boot-starter-actuator` al `pom.xml`, este endpoint funcionará. **Opcional para este lab**.

---

## 5. Conceptos aprendidos

- **analytics-service** NO usa `@KafkaListener`, usa **Kafka Streams API**
- **`application-id`** identifica el consumer group y los state stores
- **`state.dir`** es el directorio local donde se guardan los state stores
- **Eventos deben ser copias exactas** de otros microservicios para deserialización correcta
- **Constructor vacío es obligatorio** para que Jackson pueda deserializar JSON
- **Puerto 8083** para el cuarto microservicio (product:8080, order:8081, inventory:8082)
- **Kafka Streams inicia en estado RUNNING** cuando se conecta correctamente a Kafka

---

## 6. Troubleshooting

### Problema 1: Puerto 8083 ya está en uso

**Error**:
```
Web server failed to start. Port 8083 was already in use.
```

**Solución**:

```bash
# Linux/Mac
lsof -ti:8083 | xargs kill -9

# Windows
netstat -ano | findstr :8083
taskkill /PID <PID> /F
```

O cambiar puerto en `application.yml`:

```yaml
server:
  port: 8084
```

### Problema 2: Dependencia kafka-streams falta en pom.xml

**Error**:
```
Cannot resolve symbol 'StreamsBuilder'
```

**Solución**: Agregar manualmente en `pom.xml`:

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
</dependency>
```

Luego ejecutar:

```bash
mvn clean install
```

### Problema 3: application-id no configurado

**Error**:
```
StreamsConfig values:
	application.id = [null]
```

**Solución**: SIEMPRE configurar `application-id` en `application.yml`:

```yaml
spring:
  kafka:
    streams:
      application-id: analytics-service
```

### Problema 4: Kafka no está disponible

**Error**:
```
org.apache.kafka.common.errors.TimeoutException:
Failed to get offsets by times in 30000ms
```

**Solución**: Verificar que Kafka está corriendo:

```bash
docker compose ps

# Si no está corriendo
docker compose up -d
```

---

## 7. Desafío adicional

1. **Agrega Spring Boot Actuator** al proyecto:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2. **Expón endpoints adicionales** en `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

3. **Consulta métricas de Kafka Streams**:

```bash
curl http://localhost:8083/actuator/metrics/kafka.stream.state
```

---

## 8. Recursos adicionales

- [Spring for Apache Kafka - Streams](https://docs.spring.io/spring-kafka/reference/html/#kafka-streams)
- [Kafka Streams Configuration](https://kafka.apache.org/documentation/#streamsconfigs)
- [Spring Initializr](https://start.spring.io/)
- [Kafka Streams Application ID](https://kafka.apache.org/35/documentation/streams/developer-guide/config-streams.html#application-id)

---

**Siguiente**: [Lab 02: Stream básico - Contar órdenes por estado](../02-stream-contar-ordenes/README.md)
