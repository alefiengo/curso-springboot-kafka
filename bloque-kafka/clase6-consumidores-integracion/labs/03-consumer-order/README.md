# Lab 03: Consumer en order-service

Implementar consumers Kafka en order-service para escuchar eventos OrderConfirmedEvent y OrderCancelledEvent publicados por inventory-service, y actualizar automáticamente el estado de las órdenes de PENDING a CONFIRMED o CANCELLED, completando así el ciclo de comunicación asíncrona bidireccional.

---

## Objetivo

Configurar Spring Kafka consumer en order-service, crear copias de OrderConfirmedEvent y OrderCancelledEvent, implementar OrderEventConsumer con @KafkaListener, y desarrollar lógica de negocio en OrderService para actualizar el estado de órdenes basándose en eventos de inventario.

---

## Comandos a ejecutar

### Paso 1: Configurar Kafka consumer en application.yml de order-service

```bash
cd ~/workspace/order-service/src/main/resources

# Editar application.yml para agregar configuración de consumer
cat >> application.yml << 'EOF'

    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.type.mapping: |
          orderConfirmedEvent:dev.alefiengo.orderservice.kafka.event.OrderConfirmedEvent,
          orderCancelledEvent:dev.alefiengo.orderservice.kafka.event.OrderCancelledEvent
        spring.json.trusted.packages: dev.alefiengo.*
EOF
```

**Output esperado**: Sin salida, archivo modificado.

**IMPORTANTE**: La sintaxis `|` en YAML permite escribir múltiples mappings en líneas separadas.

### Paso 2: Verificar configuración completa de application.yml

El archivo `application.yml` debe tener ahora:

```yaml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce_orders}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.type.mapping: |
          orderConfirmedEvent:dev.alefiengo.orderservice.kafka.event.OrderConfirmedEvent,
          orderCancelledEvent:dev.alefiengo.orderservice.kafka.event.OrderCancelledEvent
        spring.json.trusted.packages: dev.alefiengo.*

server:
  port: 8081

logging:
  level:
    dev.alefiengo.orderservice: DEBUG
    org.springframework.web: INFO
```

### Paso 3: Crear estructura de directorios

```bash
cd ~/workspace/order-service/src/main/java/dev/alefiengo/orderservice/kafka

# Crear directorio consumer (si no existe)
mkdir -p consumer
```

**Output esperado**: Sin salida, directorio creado.

### Paso 4: Crear OrderConfirmedEvent en order-service

```bash
cd ~/workspace/order-service/src/main/java/dev/alefiengo/orderservice/kafka/event

cat > OrderConfirmedEvent.java << 'EOF'
package dev.alefiengo.orderservice.kafka.event;

import java.time.Instant;

/**
 * Evento recibido de inventory-service cuando una orden fue confirmada
 * (había stock disponible y fue reservado exitosamente).
 * IMPORTANTE: Esta es una COPIA del evento de inventory-service.
 */
public class OrderConfirmedEvent {

    private Long orderId;
    private Long productId;
    private Integer quantity;
    private Integer availableStockAfterReservation;
    private Integer reservedStockAfterReservation;
    private Instant timestamp;

    // Constructor vacío (REQUERIDO para deserialización)
    public OrderConfirmedEvent() {
    }

    // Constructor completo
    public OrderConfirmedEvent(Long orderId, Long productId, Integer quantity,
                               Integer availableStockAfterReservation,
                               Integer reservedStockAfterReservation) {
        this.orderId = orderId;
        this.productId = productId;
        this.quantity = quantity;
        this.availableStockAfterReservation = availableStockAfterReservation;
        this.reservedStockAfterReservation = reservedStockAfterReservation;
        this.timestamp = Instant.now();
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
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 5: Crear OrderCancelledEvent en order-service

```bash
cat > OrderCancelledEvent.java << 'EOF'
package dev.alefiengo.orderservice.kafka.event;

import java.time.Instant;

/**
 * Evento recibido de inventory-service cuando una orden fue cancelada
 * (no había stock suficiente o hubo un error al procesar).
 * IMPORTANTE: Esta es una COPIA del evento de inventory-service.
 */
public class OrderCancelledEvent {

    private Long orderId;
    private Long productId;
    private Integer requestedQuantity;
    private Integer availableStock;
    private String reason;
    private Instant timestamp;

    // Constructor vacío (REQUERIDO para deserialización)
    public OrderCancelledEvent() {
    }

    // Constructor completo
    public OrderCancelledEvent(Long orderId, Long productId, Integer requestedQuantity,
                               Integer availableStock, String reason) {
        this.orderId = orderId;
        this.productId = productId;
        this.requestedQuantity = requestedQuantity;
        this.availableStock = availableStock;
        this.reason = reason;
        this.timestamp = Instant.now();
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

    public Integer getRequestedQuantity() {
        return requestedQuantity;
    }

    public void setRequestedQuantity(Integer requestedQuantity) {
        this.requestedQuantity = requestedQuantity;
    }

    public Integer getAvailableStock() {
        return availableStock;
    }

    public void setAvailableStock(Integer availableStock) {
        this.availableStock = availableStock;
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
                ", requestedQuantity=" + requestedQuantity +
                ", availableStock=" + availableStock +
                ", reason='" + reason + '\'' +
                ", timestamp=" + timestamp +
                '}';
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 6: Crear OrderEventConsumer

```bash
cd ~/workspace/order-service/src/main/java/dev/alefiengo/orderservice/kafka/consumer

cat > OrderEventConsumer.java << 'EOF'
package dev.alefiengo.orderservice.kafka.consumer;

import dev.alefiengo.orderservice.kafka.event.OrderConfirmedEvent;
import dev.alefiengo.orderservice.kafka.event.OrderCancelledEvent;
import dev.alefiengo.orderservice.service.OrderService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

/**
 * Consumer de eventos OrderConfirmedEvent y OrderCancelledEvent.
 * Escucha eventos de inventory-service y actualiza el estado de las órdenes.
 */
@Component
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    private final OrderService orderService;

    // Constructor injection
    public OrderEventConsumer(OrderService orderService) {
        this.orderService = orderService;
    }

    /**
     * Consume OrderConfirmedEvent del topic ecommerce.orders.confirmed
     * y actualiza la orden a estado CONFIRMED
     */
    @KafkaListener(
        topics = "ecommerce.orders.confirmed",
        groupId = "order-service"
    )
    public void consumeOrderConfirmed(OrderConfirmedEvent event) {
        log.info("Received OrderConfirmedEvent: {}", event);

        try {
            // Delegar lógica de negocio al Service
            orderService.confirmOrder(event.getOrderId());

            log.info("Order confirmed successfully: orderId={}", event.getOrderId());
        } catch (Exception e) {
            log.error("Error confirming order: orderId={}, error={}",
                    event.getOrderId(), e.getMessage(), e);
        }
    }

    /**
     * Consume OrderCancelledEvent del topic ecommerce.orders.cancelled
     * y actualiza la orden a estado CANCELLED
     */
    @KafkaListener(
        topics = "ecommerce.orders.cancelled",
        groupId = "order-service"
    )
    public void consumeOrderCancelled(OrderCancelledEvent event) {
        log.info("Received OrderCancelledEvent: {}", event);

        try {
            // Delegar lógica de negocio al Service
            orderService.cancelOrder(event.getOrderId(), event.getReason());

            log.info("Order cancelled successfully: orderId={}, reason={}",
                    event.getOrderId(), event.getReason());
        } catch (Exception e) {
            log.error("Error cancelling order: orderId={}, error={}",
                    event.getOrderId(), e.getMessage(), e);
        }
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 7: Agregar métodos en OrderService

Edita `src/main/java/dev/alefiengo/orderservice/service/OrderService.java` y agrega estos métodos AL FINAL de la clase (antes del cierre `}`):

```java
    /**
     * Confirma una orden (actualiza estado de PENDING a CONFIRMED)
     * Llamado cuando inventory-service confirma que hay stock
     */
    @Transactional
    public void confirmOrder(Long orderId) {
        log.info("Confirming order: orderId={}", orderId);

        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));

        // Verificar que la orden está en estado PENDING
        if (order.getStatus() != OrderStatus.PENDING) {
            log.warn("Order is not PENDING, cannot confirm: orderId={}, currentStatus={}",
                    orderId, order.getStatus());
            return; // Idempotencia: Si ya fue procesada, no hacer nada
        }

        // Actualizar estado
        order.setStatus(OrderStatus.CONFIRMED);
        orderRepository.save(order);

        log.info("Order confirmed: orderId={}, newStatus={}", orderId, order.getStatus());
    }

    /**
     * Cancela una orden (actualiza estado de PENDING a CANCELLED)
     * Llamado cuando inventory-service rechaza la orden por falta de stock
     */
    @Transactional
    public void cancelOrder(Long orderId, String reason) {
        log.info("Cancelling order: orderId={}, reason={}", orderId, reason);

        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));

        // Verificar que la orden está en estado PENDING
        if (order.getStatus() != OrderStatus.PENDING) {
            log.warn("Order is not PENDING, cannot cancel: orderId={}, currentStatus={}",
                    orderId, order.getStatus());
            return; // Idempotencia: Si ya fue procesada, no hacer nada
        }

        // Actualizar estado
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);

        log.info("Order cancelled: orderId={}, newStatus={}, reason={}",
                orderId, order.getStatus(), reason);
    }
```

### Paso 8: Compilar y verificar

```bash
cd ~/workspace/order-service

# Compilar
mvn clean install
```

**Output esperado**:
```
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

### Paso 9: Ejecutar order-service

```bash
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

**Output esperado** (en logs):
```
2025-11-08T12:00:00.123  INFO 12345 --- [main] d.a.o.OrderServiceApplication            : Started OrderServiceApplication in 3.456 seconds
```

### Paso 10: Verificar que inventory-service está corriendo

**Terminal 2**:

```bash
cd ~/workspace/inventory-service
mvn spring-boot:run
```

Debe estar corriendo en puerto 8082.

### Paso 11: Crear datos de prueba

**Terminal 3** (crear item de inventario):

```bash
# Crear item de inventario con 50 unidades
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "initialStock": 50
  }'
```

**Output esperado**:
```json
{
  "id": 1,
  "productId": 1,
  "productName": "Laptop Dell XPS 15",
  "availableStock": 50,
  "reservedStock": 0,
  "totalStock": 50,
  "createdAt": "2025-11-08T12:05:00.123Z",
  "updatedAt": "2025-11-08T12:05:00.123Z"
}
```

### Paso 12: Probar flujo completo - Orden CON stock suficiente

**Terminal 3** (crear orden):

```bash
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 5,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com"
  }'
```

**Output esperado**:
```json
{
  "id": 1,
  "productId": 1,
  "quantity": 5,
  "customerName": "Juan Perez",
  "customerEmail": "juan@example.com",
  "status": "PENDING",
  "createdAt": "2025-11-08T12:10:00.123Z"
}
```

**Logs esperados en Terminal 1** (order-service):

```
2025-11-08T12:10:00.150  INFO 12345 --- [nio-8081-exec-1] d.a.o.service.OrderService               : Creating order: productId=1, quantity=5
2025-11-08T12:10:00.200  INFO 12345 --- [nio-8081-exec-1] d.a.o.k.p.OrderEventProducer             : Publishing OrderPlacedEvent: orderId=1, productId=1, quantity=5
2025-11-08T12:10:00.250  INFO 12345 --- [ad | producer-1] d.a.o.k.p.OrderEventProducer             : OrderPlacedEvent published successfully: orderId=1, partition=0, offset=0

[... 1-2 segundos después ...]

2025-11-08T12:10:01.500  INFO 12345 --- [ntainer#0-0-C-1] d.a.o.k.c.OrderEventConsumer             : Received OrderConfirmedEvent: OrderConfirmedEvent{orderId=1, productId=1, quantity=5...}
2025-11-08T12:10:01.520  INFO 12345 --- [ntainer#0-0-C-1] d.a.o.service.OrderService               : Confirming order: orderId=1
2025-11-08T12:10:01.550  INFO 12345 --- [ntainer#0-0-C-1] d.a.o.service.OrderService               : Order confirmed: orderId=1, newStatus=CONFIRMED
2025-11-08T12:10:01.560  INFO 12345 --- [ntainer#0-0-C-1] d.a.o.k.c.OrderEventConsumer             : Order confirmed successfully: orderId=1
```

**Logs esperados en Terminal 2** (inventory-service):

```
2025-11-08T12:10:00.300  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Received OrderPlacedEvent: OrderPlacedEvent{orderId=1, productId=1, quantity=5...}
2025-11-08T12:10:00.320  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Processing order: orderId=1, productId=1, quantity=5
2025-11-08T12:10:00.380  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Stock reserved successfully: orderId=1, productId=1, newAvailableStock=45, newReservedStock=5
2025-11-08T12:10:00.400  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.p.InventoryEventProducer         : Publishing OrderConfirmedEvent: orderId=1, productId=1, quantity=5
2025-11-08T12:10:00.450  INFO 12345 --- [ad | producer-1] d.a.i.k.p.InventoryEventProducer         : OrderConfirmedEvent published successfully: orderId=1, partition=0, offset=0
```

### Paso 13: Verificar estado actualizado de la orden

**Terminal 3**:

```bash
curl http://localhost:8081/api/orders/1
```

**Output esperado**:
```json
{
  "id": 1,
  "productId": 1,
  "quantity": 5,
  "customerName": "Juan Perez",
  "customerEmail": "juan@example.com",
  "status": "CONFIRMED",
  "createdAt": "2025-11-08T12:10:00.123Z"
}
```

**Observar**: `status` cambió de `PENDING` a `CONFIRMED`.

### Paso 14: Verificar stock actualizado en inventario

```bash
curl http://localhost:8082/api/inventory/product/1
```

**Output esperado**:
```json
{
  "id": 1,
  "productId": 1,
  "productName": "Laptop Dell XPS 15",
  "availableStock": 45,
  "reservedStock": 5,
  "totalStock": 50,
  "createdAt": "2025-11-08T12:05:00.123Z",
  "updatedAt": "2025-11-08T12:10:00.380Z"
}
```

**Observar**: Stock cambió (50→45 available, 0→5 reserved).

### Paso 15: Probar flujo completo - Orden SIN stock suficiente

**Terminal 3** (crear orden con cantidad excesiva):

```bash
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 1000,
    "customerName": "Maria Lopez",
    "customerEmail": "maria@example.com"
  }'
```

**Output esperado**:
```json
{
  "id": 2,
  "productId": 1,
  "quantity": 1000,
  "customerName": "Maria Lopez",
  "customerEmail": "maria@example.com",
  "status": "PENDING",
  "createdAt": "2025-11-08T12:15:00.123Z"
}
```

**Logs esperados en Terminal 1** (order-service):

```
2025-11-08T12:15:00.150  INFO 12345 --- [nio-8081-exec-2] d.a.o.service.OrderService               : Creating order: productId=1, quantity=1000
2025-11-08T12:15:00.200  INFO 12345 --- [nio-8081-exec-2] d.a.o.k.p.OrderEventProducer             : Publishing OrderPlacedEvent: orderId=2, productId=1, quantity=1000

[... 1-2 segundos después ...]

2025-11-08T12:15:01.500  INFO 12345 --- [ntainer#1-0-C-1] d.a.o.k.c.OrderEventConsumer             : Received OrderCancelledEvent: OrderCancelledEvent{orderId=2, productId=1, requestedQuantity=1000, availableStock=45, reason='Insufficient stock'...}
2025-11-08T12:15:01.520  INFO 12345 --- [ntainer#1-0-C-1] d.a.o.service.OrderService               : Cancelling order: orderId=2, reason=Insufficient stock
2025-11-08T12:15:01.550  INFO 12345 --- [ntainer#1-0-C-1] d.a.o.service.OrderService               : Order cancelled: orderId=2, newStatus=CANCELLED, reason=Insufficient stock
2025-11-08T12:15:01.560  INFO 12345 --- [ntainer#1-0-C-1] d.a.o.k.c.OrderEventConsumer             : Order cancelled successfully: orderId=2, reason=Insufficient stock
```

**Logs esperados en Terminal 2** (inventory-service):

```
2025-11-08T12:15:00.300  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Received OrderPlacedEvent: OrderPlacedEvent{orderId=2, productId=1, quantity=1000...}
2025-11-08T12:15:00.320  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Processing order: orderId=2, productId=1, quantity=1000
2025-11-08T12:15:00.325  WARN 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Insufficient stock for order: orderId=2, productId=1, requested=1000, available=45
2025-11-08T12:15:00.350  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.p.InventoryEventProducer         : Publishing OrderCancelledEvent: orderId=2, productId=1, reason=Insufficient stock
2025-11-08T12:15:00.400  INFO 12345 --- [ad | producer-1] d.a.i.k.p.InventoryEventProducer         : OrderCancelledEvent published successfully: orderId=2, partition=1, offset=0
```

### Paso 16: Verificar estado cancelado de la orden

```bash
curl http://localhost:8081/api/orders/2
```

**Output esperado**:
```json
{
  "id": 2,
  "productId": 1,
  "quantity": 1000,
  "customerName": "Maria Lopez",
  "customerEmail": "maria@example.com",
  "status": "CANCELLED",
  "createdAt": "2025-11-08T12:15:00.123Z"
}
```

**Observar**: `status` cambió de `PENDING` a `CANCELLED`.

### Paso 17: Verificar stock NO cambió (orden rechazada)

```bash
curl http://localhost:8082/api/inventory/product/1
```

**Output esperado**:
```json
{
  "id": 1,
  "productId": 1,
  "productName": "Laptop Dell XPS 15",
  "availableStock": 45,
  "reservedStock": 5,
  "totalStock": 50,
  "createdAt": "2025-11-08T12:05:00.123Z",
  "updatedAt": "2025-11-08T12:10:00.380Z"
}
```

**Observar**: Stock NO cambió (sigue en 45 available, 5 reserved).

---

## Desglose del comando

### Configuración consumer con múltiples type mappings

```yaml
spring.json.type.mapping: |
  orderConfirmedEvent:dev.alefiengo.orderservice.kafka.event.OrderConfirmedEvent,
  orderCancelledEvent:dev.alefiengo.orderservice.kafka.event.OrderCancelledEvent
```

**Sintaxis YAML**:
- `|` indica literal string multiline
- Cada mapping separado por coma
- Formato: `nombreEvento:ClaseJavaCompleta`

**Por qué es necesario**:
- order-service consume DOS tipos de eventos diferentes
- Cada tipo debe mapearse a su clase Java correspondiente
- JsonDeserializer usa este mapping para crear instancias correctas

### Dos @KafkaListener en la misma clase

```java
@KafkaListener(topics = "ecommerce.orders.confirmed", groupId = "order-service")
public void consumeOrderConfirmed(OrderConfirmedEvent event) { }

@KafkaListener(topics = "ecommerce.orders.cancelled", groupId = "order-service")
public void consumeOrderCancelled(OrderCancelledEvent event) { }
```

**Características**:
- Ambos listeners pertenecen al mismo `groupId` (order-service)
- Pero escuchan topics DIFERENTES
- Kafka gestiona offsets independientes por topic
- Cada método recibe tipo de evento específico (type-safe)

### Timeline del flujo Saga completo

```
T0: Cliente POST /api/orders
    → order-service crea Order(id=1, status=PENDING)
    → Cliente recibe HTTP 201

T1: order-service publica OrderPlacedEvent
    → Kafka topic: ecommerce.orders.placed

T2: inventory-service consume OrderPlacedEvent
    → Valida stock
    → Reserva 5 unidades (50→45 available)
    → Guarda en BD

T3: inventory-service publica OrderConfirmedEvent
    → Kafka topic: ecommerce.orders.confirmed

T4: order-service consume OrderConfirmedEvent
    → Actualiza Order(id=1, status=CONFIRMED)
    → Guarda en BD

T5: Cliente consulta GET /api/orders/1
    → Ve status=CONFIRMED
```

**Duración típica**: T0 a T4 = 1-2 segundos

---

## Explicación detallada

### Idempotencia en confirmOrder y cancelOrder

```java
if (order.getStatus() != OrderStatus.PENDING) {
    log.warn("Order is not PENDING, cannot confirm...");
    return; // NO lanzar excepción
}
```

**Por qué es crítico**:

Kafka garantiza **at-least-once delivery**: Un mensaje puede ser entregado más de una vez.

**Escenario problemático SIN idempotencia**:
```
1. Consumer recibe OrderConfirmedEvent (orderId=1)
2. Actualiza orden: PENDING → CONFIRMED
3. Consumer crashea ANTES de commitear offset
4. Kafka reenvía OrderConfirmedEvent
5. Intenta actualizar orden OTRA VEZ: CONFIRMED → CONFIRMED (problema)
```

**Con idempotencia**:
```
4. Kafka reenvía OrderConfirmedEvent
5. Verifica: order.status == CONFIRMED (no es PENDING)
6. Return sin hacer nada (safe)
7. Commit de offset
```

**Resultado**: Operación segura de repetir.

### Consumer group order-service

```yaml
consumer:
  group-id: order-service
```

**Todos los consumers con este groupId** comparten offsets:

```
order-service instance 1 ─┐
                          ├──► Consumer Group: order-service
order-service instance 2 ─┘

Topics suscritos:
- ecommerce.orders.confirmed
- ecommerce.orders.cancelled

Kafka distribuye particiones entre instancias para load balancing
```

**Si levantas 2 instancias de order-service**:
- Kafka distribuye particiones entre ellas
- Solo UNA instancia recibe cada mensaje
- Offsets compartidos por el grupo

### Eventual consistency en acción

**Cliente ve la inconsistencia**:

```bash
# T0: Crear orden
curl -X POST http://localhost:8081/api/orders ...
# Response: status=PENDING

# T1: Consultar inmediatamente
curl http://localhost:8081/api/orders/1
# Response: status=PENDING (todavía)

# T2: Esperar 2 segundos
sleep 2

# T3: Consultar otra vez
curl http://localhost:8081/api/orders/1
# Response: status=CONFIRMED (ya actualizado)
```

**Ventana de inconsistencia**: T0 a T3 (1-2 segundos).

**Esto es NORMAL** y aceptable en arquitecturas event-driven.

### Por qué NO usar REST síncrono

**Alternativa síncrona** (NO recomendada):

```java
// En OrderService.createOrder()
Order order = orderRepository.save(newOrder);

// Llamada REST síncrona a inventory-service
RestTemplate restTemplate = new RestTemplate();
InventoryResponse response = restTemplate.getForObject(
    "http://localhost:8082/api/inventory/product/" + request.getProductId(),
    InventoryResponse.class
);

if (response.getAvailableStock() >= request.getQuantity()) {
    order.setStatus(OrderStatus.CONFIRMED);
} else {
    order.setStatus(OrderStatus.CANCELLED);
}
orderRepository.save(order);
```

**Problemas**:
1. **Acoplamiento**: order-service necesita conocer URL de inventory-service
2. **Bloqueo**: HTTP request bloquea thread hasta recibir respuesta
3. **Fallo en cascada**: Si inventory-service está caído, order-service falla
4. **No escalable**: Cada orden espera respuesta antes de procesar siguiente

**Con eventos (arquitectura actual)**:
1. **Desacoplamiento**: Servicios no se conocen, solo conocen topics
2. **No bloqueante**: Publicar evento retorna inmediatamente
3. **Resiliente**: Si inventory-service está caído, mensajes esperan en Kafka
4. **Escalable**: order-service puede crear miles de órdenes sin esperar

### Duplicación de eventos (otra vez)

```
inventory-service/kafka/event/OrderConfirmedEvent.java  ← Producer
order-service/kafka/event/OrderConfirmedEvent.java      ← Consumer (COPIA)
```

**Campos idénticos** en ambas clases, pero **paquetes diferentes**.

**Por qué es necesario**:
- Microservicios NO comparten código
- Cada servicio es independiente
- Permite evolución independiente de servicios

**Ejemplo de evolución**:

```
inventory-service agrega campo "warehouseId" a OrderConfirmedEvent
order-service NO necesita ese campo
→ order-service NO tiene que cambiar su clase
→ JsonDeserializer ignora campos desconocidos
```

### Manejo de errores en consumer

```java
try {
    orderService.confirmOrder(event.getOrderId());
} catch (Exception e) {
    log.error("Error confirming order...");
    // Solo logging, NO throw
}
```

**Por qué NO re-lanzar excepción**:

Si lanzas excepción, Spring Kafka puede:
1. Reintentar (si configuraste retry)
2. O commitear offset (mensaje se pierde)

**Estrategia actual**:
- Capturar excepción
- Loguear error
- Commitear offset (mensaje se marca como procesado)
- En producción: Enviar a Dead Letter Queue (DLQ)

### Transacciones en confirmOrder y cancelOrder

```java
@Transactional
public void confirmOrder(Long orderId) {
    Order order = orderRepository.findById(orderId) ...
    order.setStatus(OrderStatus.CONFIRMED);
    orderRepository.save(order);
}
```

**Por qué @Transactional**:
- Si algo falla después de `findById` pero antes de `save`, rollback automático
- Garantiza consistencia en base de datos
- Previene estados parciales

**IMPORTANTE**: @Transactional solo cubre PostgreSQL, NO Kafka.

---

## Conceptos aprendidos

- **Saga pattern completado**: Flujo bidireccional order-service ↔ inventory-service ↔ order-service
- **Múltiples @KafkaListener**: Una clase puede tener varios listeners para topics diferentes
- **Multiple type mappings**: Configurar deserialización para múltiples tipos de eventos
- **Idempotencia**: Verificar estado antes de actualizar (safe para reprocesar)
- **Consumer group único**: Múltiples listeners en mismo groupId comparten offsets
- **Eventual consistency observable**: Cliente ve estado PENDING luego CONFIRMED
- **Event-driven vs REST**: Desacoplamiento, no-bloqueo, resiliencia vs simplicidad
- **Duplicación de eventos**: Cada servicio tiene copia de eventos que consume
- **Error handling en consumer**: Capturar, loguear, NO re-lanzar
- **@Transactional en consumer**: Protege operaciones de BD, NO Kafka
- **Estado de orden reflejando saga**: PENDING → CONFIRMED/CANCELLED según inventario

---

## Troubleshooting

### Consumer no recibe eventos

**Verificaciones**:

1. **Topics existen**:
```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list | grep orders
# Debe mostrar: orders.confirmed, orders.cancelled, orders.placed
```

2. **order-service está corriendo**:
```bash
curl http://localhost:8081/api/orders
# Debe retornar lista (puede estar vacía)
```

3. **Configuración consumer en application.yml**:
```yaml
spring:
  kafka:
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
```

4. **Hay mensajes en topics**:
```bash
docker exec -it kafka kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning
```

### Error: "Could not resolve type id 'orderConfirmedEvent'"

**Causa**: Falta type mapping en application.yml.

**Solución**: Verificar configuración:
```yaml
spring.json.type.mapping: |
  orderConfirmedEvent:dev.alefiengo.orderservice.kafka.event.OrderConfirmedEvent,
  orderCancelledEvent:dev.alefiengo.orderservice.kafka.event.OrderCancelledEvent
```

**Nota**: Debe ser paquete de **order-service**, NO inventory-service.

### Orden se queda en PENDING para siempre

**Posibles causas**:

1. **inventory-service NO está corriendo**:
```bash
curl http://localhost:8082/api/inventory
# Debe responder
```

2. **No existe item de inventario**:
```bash
curl http://localhost:8082/api/inventory/product/1
# Debe retornar item con stock
```

3. **inventory-service NO publica eventos**:
```bash
# Ver logs de inventory-service
# Buscar: "OrderConfirmedEvent published successfully"
```

4. **order-service NO consume eventos**:
```bash
# Ver logs de order-service
# Buscar: "Received OrderConfirmedEvent"
```

### Orden se cancela pero debería confirmarse

**Verificar stock disponible**:
```bash
curl http://localhost:8082/api/inventory/product/1
```

**Verificar logs de inventory-service**:
```
WARN ... Insufficient stock for order: orderId=X, requested=Y, available=Z
```

**Si available >= requested pero aún se cancela**:
- Verificar método `hasAvailableStock()` en InventoryItem
- Verificar que NO hay excepción en `reserveStock()`

### Error: "Order not found: X"

**Causa**: order-service recibe evento para orden que no existe en su BD.

**Posibles causas**:
1. Órdenes creadas en sesión anterior fueron eliminadas
2. Mensajes en Kafka son de pruebas manuales con IDs inventados
3. Bases de datos fueron reseteadas

**Solución**:
- Crear orden primero (POST /api/orders)
- Esperar que inventory-service responda
- O resetear consumer group offsets para ignorar mensajes viejos:
```bash
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-service \
  --reset-offsets \
  --to-latest \
  --all-topics \
  --execute
```

### Orden se confirma dos veces (logs duplicados)

**Esto puede ser normal** si:
1. Restarted order-service antes de commitear offset
2. Kafka reenvió mensaje

**Verificar idempotencia funcionando**:
```
INFO ... Order is not PENDING, cannot confirm: orderId=1, currentStatus=CONFIRMED
```

Si ves este log, la idempotencia está funcionando correctamente.

---

## Desafío adicional

### Desafío 1: Agregar campo `cancellationReason` en Order entity

Modificar entidad Order para guardar la razón de cancelación.

**Pistas**:
```java
@Column(name = "cancellation_reason")
private String cancellationReason;

public void cancelOrder(Long orderId, String reason) {
    order.setStatus(OrderStatus.CANCELLED);
    order.setCancellationReason(reason);  // ← Nuevo
    orderRepository.save(order);
}
```

### Desafío 2: Implementar endpoint para consultar órdenes por estado

Crear endpoint `GET /api/orders/status/{status}` que retorne todas las órdenes con ese estado.

**Pistas**:
- Agregar método en OrderRepository: `List<Order> findByStatus(OrderStatus status);`
- Agregar método en OrderService
- Agregar endpoint en OrderController

### Desafío 3: Agregar timeout a saga

Si order-service NO recibe confirmación ni cancelación en 30 segundos, marcar orden como TIMEOUT.

**Pistas**:
- Usar `@Scheduled` task que busca órdenes PENDING más antiguas de 30s
- Cambiar estado a TIMEOUT automáticamente

### Desafío 4: Implementar compensación si inventory-service está caído

Si order-service publica OrderPlacedEvent pero NO recibe respuesta en X segundos, cancelar la orden automáticamente.

**Pistas**:
- Usar `@Scheduled` para verificar órdenes PENDING
- Si createdAt es muy antiguo (ej. > 1 minuto), cancelar automáticamente
- Loguear como "Auto-cancelled due to timeout"

---

## Recursos adicionales

- [Saga Pattern with Spring Boot and Kafka](https://www.baeldung.com/saga-pattern-microservices)
- [Spring Kafka Multiple @KafkaListener](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html#kafka-listener-annotation)
- [Idempotent Consumer Pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html)
- [Eventual Consistency](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- [Compensating Transactions](https://docs.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction)
- [At-Least-Once Delivery](https://kafka.apache.org/documentation/#semantics)
