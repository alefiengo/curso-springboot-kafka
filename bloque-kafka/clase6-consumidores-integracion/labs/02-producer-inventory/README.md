# Lab 02: Producer en inventory-service

Implementar productores Kafka en inventory-service para publicar eventos OrderConfirmedEvent (cuando hay stock disponible) y OrderCancelledEvent (cuando no hay stock suficiente), completando así la comunicación asíncrona bidireccional entre order-service e inventory-service.

---

## Objetivo

Crear eventos OrderConfirmedEvent y OrderCancelledEvent, implementar InventoryEventProducer con KafkaTemplate, configurar producer en application.yml, crear topics en Kafka e integrar publicación de eventos en el método processOrderPlaced del InventoryService.

---

## Comandos a ejecutar

### Paso 1: Crear topics en Kafka

```bash
docker exec -it kafka bash

# Crear topic ecommerce.orders.confirmed
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.confirmed \
  --partitions 3 \
  --replication-factor 1

# Crear topic ecommerce.orders.cancelled
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.cancelled \
  --partitions 3 \
  --replication-factor 1

# Verificar topics creados
kafka-topics --bootstrap-server localhost:9092 --list | grep orders

# Salir del contenedor
exit
```

**Output esperado**:
```
Created topic ecommerce.orders.confirmed.
Created topic ecommerce.orders.cancelled.

ecommerce.orders.cancelled
ecommerce.orders.confirmed
ecommerce.orders.placed
```

### Paso 2: Configurar Kafka producer en application.yml

```bash
cd ~/workspace/inventory-service/src/main/resources

# Editar application.yml para agregar configuración de producer
cat >> application.yml << 'EOF'

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
EOF
```

**Output esperado**: Sin salida, archivo modificado.

**Resultado en application.yml**:
```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

    consumer:
      group-id: inventory-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.type.mapping: orderPlacedEvent:dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent
        spring.json.trusted.packages: dev.alefiengo.*

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### Paso 3: Crear OrderConfirmedEvent

```bash
cd ~/workspace/inventory-service/src/main/java/dev/alefiengo/inventoryservice/kafka/event

cat > OrderConfirmedEvent.java << 'EOF'
package dev.alefiengo.inventoryservice.kafka.event;

import java.time.Instant;

/**
 * Evento publicado cuando inventory-service confirma que hay stock disponible
 * y reserva exitosamente el inventario para una orden.
 * Este evento será consumido por order-service para actualizar el estado de la orden.
 */
public class OrderConfirmedEvent {

    private Long orderId;
    private Long productId;
    private Integer quantity;
    private Integer availableStockAfterReservation;
    private Integer reservedStockAfterReservation;
    private Instant timestamp;

    // Constructor vacío (requerido para serialización)
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

### Paso 4: Crear OrderCancelledEvent

```bash
cat > OrderCancelledEvent.java << 'EOF'
package dev.alefiengo.inventoryservice.kafka.event;

import java.time.Instant;

/**
 * Evento publicado cuando inventory-service NO puede confirmar una orden
 * porque no hay stock suficiente.
 * Este evento será consumido por order-service para cancelar la orden.
 */
public class OrderCancelledEvent {

    private Long orderId;
    private Long productId;
    private Integer requestedQuantity;
    private Integer availableStock;
    private String reason;
    private Instant timestamp;

    // Constructor vacío (requerido para serialización)
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

### Paso 5: Crear InventoryEventProducer

```bash
cd ~/workspace/inventory-service/src/main/java/dev/alefiengo/inventoryservice/kafka

mkdir -p producer

cat > producer/InventoryEventProducer.java << 'EOF'
package dev.alefiengo.inventoryservice.kafka.producer;

import dev.alefiengo.inventoryservice.kafka.event.OrderConfirmedEvent;
import dev.alefiengo.inventoryservice.kafka.event.OrderCancelledEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

/**
 * Producer de eventos de inventario.
 * Publica OrderConfirmedEvent y OrderCancelledEvent a Kafka.
 */
@Component
public class InventoryEventProducer {

    private static final Logger log = LoggerFactory.getLogger(InventoryEventProducer.class);

    private static final String TOPIC_ORDERS_CONFIRMED = "ecommerce.orders.confirmed";
    private static final String TOPIC_ORDERS_CANCELLED = "ecommerce.orders.cancelled";

    private final KafkaTemplate<String, OrderConfirmedEvent> confirmedKafkaTemplate;
    private final KafkaTemplate<String, OrderCancelledEvent> cancelledKafkaTemplate;

    // Constructor injection
    public InventoryEventProducer(
            KafkaTemplate<String, OrderConfirmedEvent> confirmedKafkaTemplate,
            KafkaTemplate<String, OrderCancelledEvent> cancelledKafkaTemplate) {
        this.confirmedKafkaTemplate = confirmedKafkaTemplate;
        this.cancelledKafkaTemplate = cancelledKafkaTemplate;
    }

    /**
     * Publica OrderConfirmedEvent cuando el stock fue reservado exitosamente
     */
    public void publishOrderConfirmed(OrderConfirmedEvent event) {
        log.info("Publishing OrderConfirmedEvent: orderId={}, productId={}, quantity={}",
                event.getOrderId(), event.getProductId(), event.getQuantity());

        String key = event.getOrderId().toString();

        confirmedKafkaTemplate.send(TOPIC_ORDERS_CONFIRMED, key, event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Failed to publish OrderConfirmedEvent: orderId={}, error={}",
                                event.getOrderId(), ex.getMessage(), ex);
                    } else {
                        log.info("OrderConfirmedEvent published successfully: orderId={}, partition={}, offset={}",
                                event.getOrderId(),
                                result.getRecordMetadata().partition(),
                                result.getRecordMetadata().offset());
                    }
                });
    }

    /**
     * Publica OrderCancelledEvent cuando no hay stock suficiente
     */
    public void publishOrderCancelled(OrderCancelledEvent event) {
        log.info("Publishing OrderCancelledEvent: orderId={}, productId={}, reason={}",
                event.getOrderId(), event.getProductId(), event.getReason());

        String key = event.getOrderId().toString();

        cancelledKafkaTemplate.send(TOPIC_ORDERS_CANCELLED, key, event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Failed to publish OrderCancelledEvent: orderId={}, error={}",
                                event.getOrderId(), ex.getMessage(), ex);
                    } else {
                        log.info("OrderCancelledEvent published successfully: orderId={}, partition={}, offset={}",
                                event.getOrderId(),
                                result.getRecordMetadata().partition(),
                                result.getRecordMetadata().offset());
                    }
                });
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 6: Modificar InventoryService para publicar eventos

Edita `src/main/java/dev/alefiengo/inventoryservice/service/InventoryService.java`:

**Agregar import del producer al inicio del archivo**:
```java
import dev.alefiengo.inventoryservice.kafka.event.OrderConfirmedEvent;
import dev.alefiengo.inventoryservice.kafka.event.OrderCancelledEvent;
import dev.alefiengo.inventoryservice.kafka.producer.InventoryEventProducer;
```

**Agregar field en la clase** (después de `private final InventoryRepository inventoryRepository;`):
```java
private final InventoryEventProducer eventProducer;
```

**Modificar constructor** para inyectar el producer:
```java
// Constructor injection
public InventoryService(InventoryRepository inventoryRepository,
                        InventoryEventProducer eventProducer) {
    this.inventoryRepository = inventoryRepository;
    this.eventProducer = eventProducer;
}
```

**Reemplazar el método processOrderPlaced completo**:
```java
/**
 * Procesa un evento OrderPlacedEvent:
 * 1. Busca el item de inventario por productId
 * 2. Valida que haya stock disponible
 * 3. SI HAY STOCK: Reserva y publica OrderConfirmedEvent
 * 4. SI NO HAY STOCK: Publica OrderCancelledEvent
 */
@Transactional
public void processOrderPlaced(dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent event) {
    log.info("Processing order: orderId={}, productId={}, quantity={}",
            event.getOrderId(), event.getProductId(), event.getQuantity());

    try {
        // 1. Buscar item de inventario
        InventoryItem item = inventoryRepository.findByProductId(event.getProductId())
                .orElseThrow(() -> new RuntimeException(
                        "Inventory item not found for productId: " + event.getProductId()
                ));

        log.debug("Current stock before reservation: availableStock={}, reservedStock={}",
                item.getAvailableStock(), item.getReservedStock());

        // 2. Validar stock disponible
        if (!item.hasAvailableStock(event.getQuantity())) {
            log.warn("Insufficient stock for order: orderId={}, productId={}, requested={}, available={}",
                    event.getOrderId(), event.getProductId(), event.getQuantity(), item.getAvailableStock());

            // Publicar OrderCancelledEvent
            OrderCancelledEvent cancelledEvent = new OrderCancelledEvent(
                    event.getOrderId(),
                    event.getProductId(),
                    event.getQuantity(),
                    item.getAvailableStock(),
                    "Insufficient stock"
            );
            eventProducer.publishOrderCancelled(cancelledEvent);

            return; // NO lanzar excepción, solo publicar evento de cancelación
        }

        // 3. Reservar stock (usa método de negocio de la entidad)
        item.reserveStock(event.getQuantity());

        // 4. Persistir cambios
        inventoryRepository.save(item);

        log.info("Stock reserved successfully: orderId={}, productId={}, newAvailableStock={}, newReservedStock={}",
                event.getOrderId(), event.getProductId(), item.getAvailableStock(), item.getReservedStock());

        // 5. Publicar OrderConfirmedEvent
        OrderConfirmedEvent confirmedEvent = new OrderConfirmedEvent(
                event.getOrderId(),
                event.getProductId(),
                event.getQuantity(),
                item.getAvailableStock(),
                item.getReservedStock()
        );
        eventProducer.publishOrderConfirmed(confirmedEvent);

    } catch (Exception e) {
        log.error("Error processing order: orderId={}, error={}", event.getOrderId(), e.getMessage(), e);

        // Publicar OrderCancelledEvent en caso de error inesperado
        OrderCancelledEvent cancelledEvent = new OrderCancelledEvent(
                event.getOrderId(),
                event.getProductId(),
                event.getQuantity(),
                0, // No tenemos el stock disponible en caso de error
                "Error processing order: " + e.getMessage()
        );
        eventProducer.publishOrderCancelled(cancelledEvent);
    }
}
```

### Paso 7: Compilar y verificar

```bash
cd ~/workspace/inventory-service

# Compilar
mvn clean install
```

**Output esperado**:
```
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

### Paso 8: Ejecutar inventory-service

```bash
mvn spring-boot:run
```

**Output esperado** (en logs de inicio):
```
2025-11-08T11:00:00.123  INFO 12345 --- [main] d.a.i.InventoryServiceApplication        : Started InventoryServiceApplication in 3.456 seconds
```

### Paso 9: Preparar consumers CLI para monitorear eventos

**Terminal 2** (consumer de OrderConfirmedEvent):

```bash
docker exec -it kafka bash

kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"
```

**Terminal 3** (consumer de OrderCancelledEvent):

```bash
docker exec -it kafka bash

kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.cancelled \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"
```

### Paso 10: Publicar OrderPlacedEvent con stock suficiente

**Terminal 4** (producer manual):

```bash
docker exec -it kafka bash

kafka-console-producer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --property "parse.key=true" \
  --property "key.separator=:"

# Publicar evento (asegúrate de tener item de inventario con productId=1)
1:{"orderId":10,"productId":1,"quantity":5,"customerName":"Juan Perez","customerEmail":"juan@example.com","timestamp":"2025-11-08T11:05:00Z"}
```

**Logs esperados en Terminal 1** (inventory-service):

```
2025-11-08T11:05:01.123  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Received OrderPlacedEvent: OrderPlacedEvent{orderId=10, productId=1, quantity=5...}
2025-11-08T11:05:01.150  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Processing order: orderId=10, productId=1, quantity=5
2025-11-08T11:05:01.200  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Stock reserved successfully: orderId=10, productId=1, newAvailableStock=45, newReservedStock=5
2025-11-08T11:05:01.234  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.p.InventoryEventProducer         : Publishing OrderConfirmedEvent: orderId=10, productId=1, quantity=5
2025-11-08T11:05:01.280  INFO 12345 --- [ad | producer-1] d.a.i.k.p.InventoryEventProducer         : OrderConfirmedEvent published successfully: orderId=10, partition=0, offset=0
```

**Output en Terminal 2** (consumer de confirmed):

```
10:{"orderId":10,"productId":1,"quantity":5,"availableStockAfterReservation":45,"reservedStockAfterReservation":5,"timestamp":"2025-11-08T11:05:01.234Z"}
```

### Paso 11: Publicar OrderPlacedEvent sin stock suficiente

**Terminal 4** (producer manual):

```bash
# Publicar evento que requiere más stock del disponible
2:{"orderId":11,"productId":1,"quantity":1000,"customerName":"Maria Lopez","customerEmail":"maria@example.com","timestamp":"2025-11-08T11:10:00Z"}
```

**Logs esperados en Terminal 1** (inventory-service):

```
2025-11-08T11:10:01.123  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Received OrderPlacedEvent: OrderPlacedEvent{orderId=11, productId=1, quantity=1000...}
2025-11-08T11:10:01.150  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Processing order: orderId=11, productId=1, quantity=1000
2025-11-08T11:10:01.160  WARN 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Insufficient stock for order: orderId=11, productId=1, requested=1000, available=45
2025-11-08T11:10:01.190  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.p.InventoryEventProducer         : Publishing OrderCancelledEvent: orderId=11, productId=1, reason=Insufficient stock
2025-11-08T11:10:01.220  INFO 12345 --- [ad | producer-1] d.a.i.k.p.InventoryEventProducer         : OrderCancelledEvent published successfully: orderId=11, partition=1, offset=0
```

**Output en Terminal 3** (consumer de cancelled):

```
11:{"orderId":11,"productId":1,"requestedQuantity":1000,"availableStock":45,"reason":"Insufficient stock","timestamp":"2025-11-08T11:10:01.190Z"}
```

### Paso 12: Verificar stock NO cambió en caso de cancelación

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
  "totalStock": 50
}
```

**Observar**: Stock NO cambió (orden 11 fue rechazada).

---

## Desglose del comando

### Configuración producer en application.yml

| Propiedad | Valor | Descripción |
|-----------|-------|-------------|
| `producer.key-serializer` | `StringSerializer` | Serializa keys (orderId) como String |
| `producer.value-serializer` | `JsonSerializer` | Serializa events como JSON automáticamente |

**IMPORTANTE**: NO necesitas configurar `spring.json.type.mapping` en producer (solo en consumer).

### Topics creados

| Topic | Particiones | Producer | Consumer (Lab 03) | Propósito |
|-------|-------------|----------|-------------------|-----------|
| `ecommerce.orders.confirmed` | 3 | inventory-service | order-service | Notificar que orden fue confirmada (hay stock) |
| `ecommerce.orders.cancelled` | 3 | inventory-service | order-service | Notificar que orden fue cancelada (sin stock) |

### KafkaTemplate con tipos específicos

```java
private final KafkaTemplate<String, OrderConfirmedEvent> confirmedKafkaTemplate;
private final KafkaTemplate<String, OrderCancelledEvent> cancelledKafkaTemplate;
```

**Por qué dos KafkaTemplates**:
- Cada uno tiene tipo genérico específico para el evento
- Spring autowiring los crea automáticamente basándose en la configuración
- Provee type-safety en tiempo de compilación

**Alternativa** (NO recomendada):
```java
KafkaTemplate<String, Object>  // Pierde type-safety
```

### Estructura de OrderConfirmedEvent

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `orderId` | `Long` | ID de la orden confirmada |
| `productId` | `Long` | ID del producto |
| `quantity` | `Integer` | Cantidad reservada |
| `availableStockAfterReservation` | `Integer` | Stock disponible DESPUÉS de reservar (para auditoría) |
| `reservedStockAfterReservation` | `Integer` | Stock reservado DESPUÉS de reservar (para auditoría) |
| `timestamp` | `Instant` | Cuándo se confirmó |

**Campos de auditoría**: Incluir el estado del stock DESPUÉS de la operación permite:
- Debugging (saber exactamente qué stock había en ese momento)
- Analytics (analizar tendencias de inventario)
- Reconciliación (verificar consistencia entre servicios)

### Estructura de OrderCancelledEvent

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `orderId` | `Long` | ID de la orden cancelada |
| `productId` | `Long` | ID del producto |
| `requestedQuantity` | `Integer` | Cantidad solicitada (que NO se pudo reservar) |
| `availableStock` | `Integer` | Stock disponible en el momento del rechazo |
| `reason` | `String` | Razón de cancelación ("Insufficient stock", "Error processing order: ...") |
| `timestamp` | `Instant` | Cuándo se canceló |

**Campo `reason`**: Permite distinguir entre:
- Cancelación por falta de stock (caso normal)
- Cancelación por error técnico (ej. BD caída, timeout)
- Futuras razones (validaciones de negocio, producto descontinuado, etc.)

---

## Explicación detallada

### Flujo completo del Saga pattern

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  order-service  │         │      Kafka      │         │ inventory-service│
└────────┬────────┘         └────────┬────────┘         └────────┬────────┘
         │                           │                           │
         │ 1. POST /api/orders       │                           │
         ├──────────────────────────►│                           │
         │ Order saved (PENDING)     │                           │
         │                           │                           │
         │ 2. OrderPlacedEvent       │                           │
         ├──────────────────────────►│                           │
         │                           │                           │
         │                           │ 3. Consume event          │
         │                           ├──────────────────────────►│
         │                           │                           │
         │                           │    4a. IF stock OK:       │
         │                           │        Reserve stock      │
         │                           │        Save to DB         │
         │                           │                           │
         │                           │ 5a. OrderConfirmedEvent   │
         │                           │◄──────────────────────────┤
         │                           │                           │
         │ 6a. [Lab 03]              │                           │
         │     Update Order          │                           │
         │     (PENDING → CONFIRMED) │                           │
         │                           │                           │
         │                           │    4b. IF NO stock:       │
         │                           │        (NO DB changes)    │
         │                           │                           │
         │                           │ 5b. OrderCancelledEvent   │
         │                           │◄──────────────────────────┤
         │                           │                           │
         │ 6b. [Lab 03]              │                           │
         │     Update Order          │                           │
         │     (PENDING → CANCELLED) │                           │
         │                           │                           │
```

**Patrón Saga por coreografía**:
- NO hay coordinador central
- Cada servicio escucha eventos y reacciona
- Cada servicio publica sus propios eventos
- Compensación mediante eventos (OrderCancelledEvent)

### Dos KafkaTemplates vs uno genérico

**Implementación actual** (recomendada):
```java
private final KafkaTemplate<String, OrderConfirmedEvent> confirmedKafkaTemplate;
private final KafkaTemplate<String, OrderCancelledEvent> cancelledKafkaTemplate;
```

**Ventajas**:
- Type-safety: Compilador previene errores de tipo
- Claridad: Código autodocumentado (qué template usa qué evento)
- Autowiring automático: Spring crea beans basándose en tipos genéricos

**Alternativa con KafkaTemplate genérico**:
```java
private final KafkaTemplate<String, Object> kafkaTemplate;

public void publishOrderConfirmed(OrderConfirmedEvent event) {
    kafkaTemplate.send(TOPIC, key, event);  // ← Pierde type-safety
}
```

**Desventaja**: Pierdes validación en tiempo de compilación.

**Para este curso**: Usamos dos KafkaTemplates para enseñar type-safety y buenas prácticas.

### Por qué NO lanzar excepción en caso de stock insuficiente

**Implementación anterior** (Lab 01):
```java
if (!item.hasAvailableStock(quantity)) {
    throw new RuntimeException("Insufficient stock");  // ← Problema
}
```

**Problema**: La excepción se propaga al consumer, que la loguea pero NO hace nada más.

**Implementación correcta** (Lab 02):
```java
if (!item.hasAvailableStock(quantity)) {
    // Publicar evento de cancelación
    eventProducer.publishOrderCancelled(...);
    return;  // ← Salir normalmente, SIN excepción
}
```

**Ventaja**: El flujo es explícito:
- Falta de stock NO es un error técnico
- Es un caso de negocio válido
- El evento OrderCancelledEvent comunica claramente qué pasó

### Manejo de errores técnicos

```java
try {
    // Procesar orden
} catch (Exception e) {
    log.error("Error processing order...");

    // Publicar OrderCancelledEvent con reason=error
    OrderCancelledEvent cancelledEvent = new OrderCancelledEvent(
        orderId, productId, quantity, 0,
        "Error processing order: " + e.getMessage()
    );
    eventProducer.publishOrderCancelled(cancelledEvent);
}
```

**Por qué publicar evento incluso en caso de error**:
- order-service NECESITA saber que la orden falló
- Sin evento, la orden quedaría en PENDING para siempre
- El campo `reason` distingue entre falta de stock y error técnico

### JsonSerializer automático

```yaml
spring:
  kafka:
    producer:
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

**Qué hace JsonSerializer**:
1. Recibe objeto Java (OrderConfirmedEvent)
2. Usa Jackson para convertir a JSON
3. Agrega metadata de tipo: `"@type": "orderConfirmedEvent"`
4. Serializa a bytes
5. Envía a Kafka

**JSON producido**:
```json
{
  "@type": "orderConfirmedEvent",
  "orderId": 10,
  "productId": 1,
  "quantity": 5,
  "availableStockAfterReservation": 45,
  "reservedStockAfterReservation": 5,
  "timestamp": "2025-11-08T11:05:01.234Z"
}
```

**IMPORTANTE**: El `@type` es usado por el consumer para deserializar (via `spring.json.type.mapping`).

### CompletableFuture y whenComplete

```java
kafkaTemplate.send(TOPIC, key, event)
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("Failed to publish...");
        } else {
            log.info("Published successfully...");
        }
    });
```

**Por qué usar whenComplete**:
- `kafkaTemplate.send()` es **asíncrono** (retorna inmediatamente)
- `whenComplete()` se ejecuta cuando Kafka responde (éxito o fallo)
- NO bloquea el thread principal

**Alternativa bloqueante** (NO recomendada):
```java
kafkaTemplate.send(TOPIC, key, event).get();  // ← Bloquea hasta que Kafka responda
```

### Transaccionalidad y eventos

```java
@Transactional
public void processOrderPlaced(...) {
    // 1. Reservar stock en BD
    inventoryRepository.save(item);

    // 2. Publicar evento a Kafka
    eventProducer.publishOrderConfirmed(...);
}
```

**IMPORTANTE**: La transacción `@Transactional` solo cubre la base de datos, NO Kafka.

**Escenario problemático**:
```
1. Reserva stock en BD ✓
2. Commit de transacción en PostgreSQL ✓
3. Publica evento a Kafka ✗ (falla)

Resultado: Stock reservado pero order-service nunca recibe confirmación
```

**Solución avanzada** (fuera de alcance): Transactional Outbox Pattern:
- Guardar evento en tabla `outbox` en la misma transacción
- Proceso separado lee tabla y publica a Kafka
- Garantiza at-least-once delivery

**Para este curso**: Aceptamos que en casos extremadamente raros puede haber inconsistencia.

### Particionamiento con key

```java
String key = event.getOrderId().toString();
kafkaTemplate.send(TOPIC, key, event);
```

**Por qué usar orderId como key**:
- Todos los eventos de la misma orden van a la misma partición
- Kafka garantiza orden dentro de una partición
- Ejemplo: Orden 10 → Partition 0, Orden 11 → Partition 1, Orden 12 → Partition 0

**Visualización**:
```
ecommerce.orders.confirmed
├─ Partition 0: Order 10, Order 12, Order 15
├─ Partition 1: Order 11, Order 14
└─ Partition 2: Order 13

Dentro de cada partición: ORDEN GARANTIZADO
Entre particiones: NO hay garantía de orden
```

---

## Conceptos aprendidos

- **Saga Pattern por coreografía**: Cada servicio reacciona a eventos sin coordinador central
- **Eventos de compensación**: OrderCancelledEvent compensa la operación fallida
- **Dos KafkaTemplates**: Type-safety con tipos genéricos específicos
- **JsonSerializer**: Serialización automática de objetos Java a JSON
- **Metadata @type**: Permite al consumer deserializar correctamente
- **Campos de auditoría**: Incluir estado post-operación en eventos para debugging
- **Manejo de errores de negocio**: Publicar evento (NO lanzar excepción) cuando falta stock
- **Manejo de errores técnicos**: Publicar OrderCancelledEvent incluso en caso de error
- **CompletableFuture**: Publicación asíncrona con callback whenComplete
- **Transaccionalidad parcial**: @Transactional solo cubre BD, NO Kafka
- **Particionamiento**: Usar orderId como key garantiza orden de eventos por orden
- **Eventual consistency**: Ventana de tiempo entre reserva de stock y actualización de orden

---

## Troubleshooting

### Error: "Topic 'ecommerce.orders.confirmed' does not exist"

**Causa**: No creaste el topic antes de ejecutar.

**Solución**:
```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.confirmed \
  --partitions 3 \
  --replication-factor 1
```

### Error: "Could not autowire. No beans of type 'KafkaTemplate<String, OrderConfirmedEvent>' found"

**Causa**: Spring no puede crear KafkaTemplate con tipo específico.

**Solución**: Verifica que `spring-kafka` está en `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

Y que la configuración de producer está en `application.yml`:
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### Error: "Failed to publish... Connection refused"

**Causa**: Kafka no está corriendo.

**Solución**:
```bash
cd ~/workspace/kafka-infrastructure
docker compose ps | grep kafka
# Si no está "Up":
docker compose up -d
```

### Events publicados pero consumer CLI no muestra nada

**Verificaciones**:

1. **Consumer CLI inició DESPUÉS de publicar evento**:
```bash
# Agregar --from-beginning para leer eventos históricos
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning
```

2. **Topic correcto**:
```bash
kafka-topics --bootstrap-server localhost:9092 --list | grep confirmed
```

3. **Eventos realmente publicados**:
```bash
# Ver logs de inventory-service
# Buscar: "OrderConfirmedEvent published successfully"
```

### OrderConfirmedEvent se publica pero stock NO se reservó

**Causa**: Orden de operaciones incorrecta.

**Verificar en InventoryService.java**:
```java
// ✅ CORRECTO: Primero guardar, luego publicar
inventoryRepository.save(item);
eventProducer.publishOrderConfirmed(...);

// ❌ INCORRECTO: Publicar antes de guardar
eventProducer.publishOrderConfirmed(...);
inventoryRepository.save(item);  // Si esto falla, evento ya fue publicado
```

### OrderCancelledEvent se publica pero logs muestran excepción

**Esto es esperado** si el código anterior tenía `throw new RuntimeException`.

**Verificar** que el método processOrderPlaced NO lanza excepción en caso de stock insuficiente:
```java
if (!item.hasAvailableStock(quantity)) {
    eventProducer.publishOrderCancelled(...);
    return;  // ← Salir normalmente, NO throw
}
```

---

## Desafío adicional

### Desafío 1: Implementar evento InventoryUpdatedEvent

Crear un tercer evento `InventoryUpdatedEvent` que se publique CADA VEZ que cambie el stock (tanto en confirmación como cancelación).

**Pistas**:
- Topic: `ecommerce.inventory.updated`
- Campos: productId, previousAvailableStock, newAvailableStock, previousReservedStock, newReservedStock, reason ("ORDER_CONFIRMED", "ORDER_CANCELLED")
- Publicar en processOrderPlaced DESPUÉS de confirmar/cancelar

### Desafío 2: Agregar retry al producer

Si la publicación a Kafka falla, reintentar 3 veces con backoff exponencial.

**Pistas**:
```yaml
spring:
  kafka:
    producer:
      retries: 3
      properties:
        retry.backoff.ms: 1000
```

### Desafío 3: Implementar Transactional Outbox Pattern (avanzado)

Crear tabla `outbox` en PostgreSQL que guarde eventos antes de publicarlos a Kafka.

**Tabla outbox**:
```sql
CREATE TABLE outbox (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(255) NOT NULL,
    event_payload TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    published BOOLEAN DEFAULT FALSE
);
```

**Flujo**:
1. Guardar reserva de stock + insertar evento en `outbox` (misma transacción)
2. Proceso separado (scheduled task) lee `outbox` y publica a Kafka
3. Marca evento como `published = true`

### Desafío 4: Agregar métricas con Micrometer

Registrar métricas de:
- Número de órdenes confirmadas (counter)
- Número de órdenes canceladas (counter)
- Tiempo de procesamiento de órdenes (timer)

**Pistas**:
```java
@Autowired
private MeterRegistry meterRegistry;

public void processOrderPlaced(...) {
    Timer.Sample sample = Timer.start(meterRegistry);

    // Procesar orden

    sample.stop(Timer.builder("inventory.order.processing.time")
            .tag("status", "confirmed")
            .register(meterRegistry));

    meterRegistry.counter("inventory.orders.confirmed").increment();
}
```

---

## Recursos adicionales

- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Spring Kafka Producer](https://docs.spring.io/spring-kafka/reference/kafka/sending-messages.html)
- [JsonSerializer Documentation](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/support/serializer/JsonSerializer.html)
- [CompletableFuture Guide](https://www.baeldung.com/java-completablefuture)
- [Kafka Partitioning Strategy](https://kafka.apache.org/documentation/#producerconfigs_partitioner.class)
