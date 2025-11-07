# Lab 03: Producer en order-service

## Objetivo

Integrar spring-kafka en order-service, crear el evento OrderPlacedEvent, implementar OrderEventProducer y publicar eventos al topic ecommerce.orders.placed cada vez que se crea una orden, completando así la implementación de productores en ambos microservicios.

---

## Comandos a ejecutar

```bash
# 1. Navegar al proyecto order-service
cd order-service

# 2. Agregar dependencia spring-kafka a pom.xml
# (Ver sección "Desglose del comando")

# 3. Recargar dependencias
mvn clean install

# 4. Crear topic ecommerce.orders.placed
docker exec -it kafka bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.placed \
  --partitions 3 \
  --replication-factor 1

# 5. Verificar topic creado
kafka-topics --bootstrap-server localhost:9092 --list

# 6. Salir del contenedor
exit

# 7. Configurar Kafka en application.yml
# (Ver sección "Desglose del comando")

# 8. Crear estructura de paquetes para Kafka
mkdir -p src/main/java/dev/alefiengo/orderservice/kafka/event
mkdir -p src/main/java/dev/alefiengo/orderservice/kafka/producer

# 9. Crear clases Java (ver sección "Desglose del comando")
# - OrderPlacedEvent.java
# - OrderEventProducer.java

# 10. Modificar OrderService para publicar eventos

# 11. Ejecutar order-service en puerto 8081
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081

# 12. En otra terminal, consumir eventos desde CLI
docker exec -it kafka bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning \
  --property print.key=true

# 13. Crear una orden
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 3,
    "customerName": "Maria Garcia",
    "customerEmail": "maria@example.com",
    "totalAmount": 450.00
  }'

# Deberías ver el evento OrderPlacedEvent en el consumer
```

---

## Desglose del comando

### 1. Dependencia Maven (pom.xml)

Agregar dentro de `<dependencies>`:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

**Mismo proceso que en product-service** (Lab 00).

### 2. Configuración Kafka (application.yml)

Agregar al final de `src/main/resources/application.yml`:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

**Nota**: Configuración idéntica a product-service. En ambos casos usamos JsonSerializer.

### 3. Evento OrderPlacedEvent

Crear `src/main/java/dev/alefiengo/orderservice/kafka/event/OrderPlacedEvent.java`:

```java
package dev.alefiengo.orderservice.kafka.event;

import java.math.BigDecimal;
import java.time.Instant;

public class OrderPlacedEvent {

    private Long orderId;
    private Long productId;
    private Integer quantity;
    private String customerName;
    private String customerEmail;
    private BigDecimal totalAmount;
    private Instant timestamp;

    public OrderPlacedEvent() {
        // Constructor vacío para deserialización JSON
    }

    public OrderPlacedEvent(Long orderId, Long productId, Integer quantity,
                            String customerName, String customerEmail,
                            BigDecimal totalAmount) {
        this.orderId = orderId;
        this.productId = productId;
        this.quantity = quantity;
        this.customerName = customerName;
        this.customerEmail = customerEmail;
        this.totalAmount = totalAmount;
        this.timestamp = Instant.now();
    }

    // Getters y Setters
    public Long getOrderId() { return orderId; }
    public void setOrderId(Long orderId) { this.orderId = orderId; }

    public Long getProductId() { return productId; }
    public void setProductId(Long productId) { this.productId = productId; }

    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }

    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }

    public String getCustomerEmail() { return customerEmail; }
    public void setCustomerEmail(String customerEmail) { this.customerEmail = customerEmail; }

    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }

    public Instant getTimestamp() { return timestamp; }
    public void setTimestamp(Instant timestamp) { this.timestamp = timestamp; }

    @Override
    public String toString() {
        return "OrderPlacedEvent{" +
                "orderId=" + orderId +
                ", productId=" + productId +
                ", quantity=" + quantity +
                ", customerName='" + customerName + '\'' +
                ", totalAmount=" + totalAmount +
                ", timestamp=" + timestamp +
                '}';
    }
}
```

**Campos del evento**:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `orderId` | `Long` | ID de la orden creada |
| `productId` | `Long` | Producto solicitado |
| `quantity` | `Integer` | Cantidad solicitada |
| `customerName` | `String` | Cliente que hizo la orden |
| `customerEmail` | `String` | Email del cliente |
| `totalAmount` | `BigDecimal` | Monto total de la orden |
| `timestamp` | `Instant` | Cuándo se creó el evento |

**¿Por qué incluir customerName y customerEmail?**

En Clase 6, inventory-service necesitará esta información para notificar al cliente si la orden se confirma o cancela.

### 4. Producer OrderEventProducer

Crear `src/main/java/dev/alefiengo/orderservice/kafka/producer/OrderEventProducer.java`:

```java
package dev.alefiengo.orderservice.kafka.producer;

import dev.alefiengo.orderservice.kafka.event.OrderPlacedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class OrderEventProducer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventProducer.class);
    private static final String TOPIC = "ecommerce.orders.placed";

    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publishOrderPlaced(OrderPlacedEvent event) {
        String key = event.getOrderId().toString();

        log.info("Publishing OrderPlacedEvent: orderId={}, productId={}, quantity={}",
                 event.getOrderId(), event.getProductId(), event.getQuantity());

        kafkaTemplate.send(TOPIC, key, event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish OrderPlacedEvent: orderId={}",
                              event.getOrderId(), ex);
                } else {
                    log.info("OrderPlacedEvent published successfully: orderId={}, partition={}",
                             event.getOrderId(),
                             result.getRecordMetadata().partition());
                }
            });
    }
}
```

**Diferencias con ProductEventProducer**:

| Aspecto | ProductEventProducer | OrderEventProducer |
|---------|---------------------|-------------------|
| Topic | `ecommerce.products.created` | `ecommerce.orders.placed` |
| Event | `ProductCreatedEvent` | `OrderPlacedEvent` |
| Key | `productId` | `orderId` |
| Logging | productId, name | orderId, productId, quantity |

**Estructura idéntica**: Ambos productores siguen el mismo patrón.

### 5. Integración en OrderService

Modificar `src/main/java/dev/alefiengo/orderservice/service/OrderService.java`:

```java
package dev.alefiengo.orderservice.service;

import dev.alefiengo.orderservice.kafka.event.OrderPlacedEvent;
import dev.alefiengo.orderservice.kafka.producer.OrderEventProducer;
import dev.alefiengo.orderservice.model.dto.OrderRequest;
import dev.alefiengo.orderservice.model.dto.OrderResponse;
import dev.alefiengo.orderservice.model.entity.Order;
import dev.alefiengo.orderservice.repository.OrderRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderEventProducer orderEventProducer; // Nuevo

    public OrderService(OrderRepository orderRepository,
                        OrderEventProducer orderEventProducer) { // Nuevo
        this.orderRepository = orderRepository;
        this.orderEventProducer = orderEventProducer; // Nuevo
    }

    @Transactional
    public OrderResponse create(OrderRequest request) {
        // 1. Crear y guardar orden
        Order order = new Order();
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setCustomerName(request.getCustomerName());
        order.setCustomerEmail(request.getCustomerEmail());
        order.setTotalAmount(request.getTotalAmount());

        Order saved = orderRepository.save(order);

        // 2. Publicar evento a Kafka
        OrderPlacedEvent event = new OrderPlacedEvent(
            saved.getId(),
            saved.getProductId(),
            saved.getQuantity(),
            saved.getCustomerName(),
            saved.getCustomerEmail(),
            saved.getTotalAmount()
        );
        orderEventProducer.publishOrderPlaced(event);

        // 3. Retornar respuesta
        return mapToResponse(saved);
    }

    // Resto de métodos sin cambios...
    @Transactional(readOnly = true)
    public List<OrderResponse> findAll() {
        return orderRepository.findAll().stream()
                .map(this::mapToResponse)
                .collect(Collectors.toList());
    }

    @Transactional(readOnly = true)
    public OrderResponse findById(Long id) {
        Order order = orderRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Order not found with id: " + id));
        return mapToResponse(order);
    }

    private OrderResponse mapToResponse(Order order) {
        OrderResponse response = new OrderResponse();
        response.setId(order.getId());
        response.setProductId(order.getProductId());
        response.setQuantity(order.getQuantity());
        response.setCustomerName(order.getCustomerName());
        response.setCustomerEmail(order.getCustomerEmail());
        response.setTotalAmount(order.getTotalAmount());
        response.setStatus(order.getStatus());
        response.setCreatedAt(order.getCreatedAt());
        return response;
    }
}
```

---

## Explicación detallada

### Paso 1: Dos microservicios, dos productores

Ahora tenemos dos microservicios publicando eventos:

```
┌──────────────────┐                  ┌─────────────┐
│ product-service  │─────────────────>│ Kafka       │
│ (puerto 8080)    │ ProductCreated   │             │
└──────────────────┘                  │ Topics:     │
                                      │ - products  │
┌──────────────────┐                  │ - orders    │
│  order-service   │─────────────────>│             │
│ (puerto 8081)    │ OrderPlaced      └─────────────┘
└──────────────────┘
```

**Arquitectura event-driven**: Ambos servicios publican eventos, pero NO se comunican directamente.

### Paso 2: OrderPlacedEvent - Información completa

```java
public class OrderPlacedEvent {
    private Long orderId;       // Identifica la orden
    private Long productId;     // Qué producto
    private Integer quantity;   // Cuántos
    private String customerName;
    private String customerEmail;
    private BigDecimal totalAmount;
    private Instant timestamp;
}
```

**Principio**: El evento debe ser **auto-contenido** (self-contained).

**¿Por qué?**

Los consumidores (Clase 6) NO deberían llamar a order-service para obtener más info. El evento contiene todo lo necesario.

**Comparación**:

```java
// ❌ Evento incompleto
public class OrderPlacedEvent {
    private Long orderId; // Consumidor debe llamar a GET /api/orders/{id}
}

// ✅ Evento completo
public class OrderPlacedEvent {
    private Long orderId;
    private Long productId;
    private Integer quantity;
    // ... todo lo necesario
}
```

### Paso 3: Key de particionamiento (orderId)

```java
String key = event.getOrderId().toString();
kafkaTemplate.send(TOPIC, key, event);
```

**¿Por qué usar orderId como key?**

En Clase 6 implementaremos:

- `OrderPlacedEvent` (este lab)
- `OrderConfirmedEvent` (inventory-service confirma)
- `OrderCancelledEvent` (inventory-service rechaza)

**Usando orderId como key**:

```
orderId=1 → Partition 0
  - OrderPlacedEvent(orderId=1)
  - OrderConfirmedEvent(orderId=1)  ← Misma partición
```

**Ventaja**: Orden garantizado de eventos por orden.

### Paso 4: Orden de operaciones crítico

```java
@Transactional
public OrderResponse create(OrderRequest request) {
    // 1. Guardar en BD PRIMERO
    Order saved = orderRepository.save(order);

    // 2. Publicar evento DESPUÉS
    orderEventProducer.publishOrderPlaced(event);

    // 3. Retornar respuesta
    return mapToResponse(saved);
}
```

**¿Por qué este orden?**

Escenario 1: Guardar falla

```
save() → FALLA → Exception
publicar() → NO se ejecuta
cliente → recibe error 500
```

✅ Correcto: No se publicó evento de orden que no existe.

Escenario 2: Publicar falla

```
save() → OK → orden existe en BD
publicar() → FALLA → log de error
cliente → recibe 201 Created
```

⚠️ Problema menor: Orden existe pero evento no publicado. Puede manejarse con retry o monitoreo.

**Escenario inverso** (publicar primero):

```
publicar() → OK → evento en Kafka
save() → FALLA → rollback
cliente → recibe error 500
```

❌ Problema grave: Evento dice "orden creada" pero NO existe en BD.

### Paso 5: Testing con dos microservicios

**Terminal 1**: product-service

```bash
cd product-service
mvn spring-boot:run
# Corre en puerto 8080
```

**Terminal 2**: order-service

```bash
cd order-service
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
# Corre en puerto 8081
```

**Terminal 3**: Consumer de products

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning
```

**Terminal 4**: Consumer de orders

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning
```

**Terminal 5**: Crear datos

```bash
# Crear producto
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Mouse Logitech","price":25,"categoryId":1}'

# Crear orden
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":2,"customerName":"Juan","customerEmail":"juan@example.com","totalAmount":50}'
```

Deberías ver eventos en Terminal 3 y 4.

### Paso 6: Preparación para Clase 6

En Clase 6 agregaremos **inventory-service** que:

1. **Consume** `ecommerce.orders.placed`
2. Valida stock del producto
3. **Produce** `ecommerce.orders.confirmed` o `ecommerce.orders.cancelled`
4. order-service **consume** estos eventos y actualiza estado

```
order-service ─────> ecommerce.orders.placed ─────> inventory-service
      ↑                                                      │
      │                                                      │
      └──── ecommerce.orders.confirmed/cancelled ←──────────┘
```

**En este lab**: Solo publicamos OrderPlacedEvent. Nadie lo consume aún.

---

## Conceptos aprendidos

- Múltiples microservicios pueden publicar a **diferentes topics** en el mismo cluster Kafka
- **Eventos auto-contenidos** incluyen toda la información necesaria sin requerir llamadas adicionales
- **orderId como key** garantiza orden de eventos para la misma orden
- **Mismo patrón de producer** se aplica en todos los microservicios (coherencia arquitectónica)
- Orden de operaciones: **primero persistir, luego publicar** para evitar inconsistencias
- **Logging detallado** facilita debugging cuando múltiples servicios publican eventos
- Topics diferentes permiten **separación de concerns** (products vs orders)
- **Puerto diferente por servicio** permite ejecución simultánea en desarrollo
- Configuración Kafka es **idéntica entre servicios** (JsonSerializer, StringSerializer)
- Eventos quedan en Kafka aunque **nadie los consuma** (preparación para Clase 6)

---

## Troubleshooting

### Problema 1: Topic ecommerce.orders.placed no existe

**Síntoma**:

```
Topic ecommerce.orders.placed not present in metadata
```

**Solución**:

```bash
docker exec -it kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --create --topic ecommerce.orders.placed \
  --partitions 3 --replication-factor 1
```

### Problema 2: Ambos servicios intentan usar puerto 8080

**Síntoma**:

```
Port 8080 was already in use
```

**Causa**: product-service ya está usando 8080.

**Solución**: Ejecutar order-service en puerto diferente:

```bash
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

O configurar en application.yml:

```yaml
server:
  port: 8081
```

### Problema 3: OrderEventProducer no se inyecta

**Síntoma**:

```
NullPointerException at OrderService.create()
```

**Causa**: Falta anotación `@Component` en OrderEventProducer.

**Solución**:

```java
@Component  // ← Importante
public class OrderEventProducer {
    // ...
}
```

### Problema 4: Evento OrderPlacedEvent se serializa como {}

**Síntoma**: Consumer muestra `{}` en lugar del evento completo.

**Causa**: Faltan getters o constructor vacío.

**Solución**: Verificar OrderPlacedEvent:

```java
public OrderPlacedEvent() {}  // Constructor vacío

// Getters
public Long getOrderId() { return orderId; }
public Long getProductId() { return productId; }
// ... todos los getters
```

### Problema 5: Error "Failed to publish" pero orden se crea

**Síntoma**: Orden existe en BD pero log muestra error de Kafka.

**Causa**: Kafka no está accesible o topic no existe.

**Solución**:

```bash
# Verificar Kafka corriendo
docker compose ps

# Verificar topic existe
docker exec -it kafka kafka-topics \
  --bootstrap-server localhost:9092 --list
```

**Implicación**: En producción, implementar retry logic o dead letter queue.

---

## Desafío adicional

### Desafío 1: Verificar eventos con diferentes claves

Crea múltiples órdenes y observa cómo se distribuyen en particiones:

```bash
# Crear 5 órdenes
for i in {1..5}; do
  curl -X POST http://localhost:8081/api/orders \
    -H "Content-Type: application/json" \
    -d "{\"productId\":1,\"quantity\":$i,\"customerName\":\"Cliente$i\",\"customerEmail\":\"cliente$i@example.com\",\"totalAmount\":$(($i*25))}"
done
```

Observa en el consumer cómo las órdenes se distribuyen entre las 3 particiones.

### Desafío 2: Implementar OrderDeletedEvent

Agrega evento para eliminación de órdenes:

1. Crear `OrderDeletedEvent` con solo `orderId` y `timestamp`
2. Agregar método `publishOrderDeleted()` en `OrderEventProducer`
3. Implementar endpoint `DELETE /api/orders/{id}` en OrderController
4. Publicar evento antes de eliminar de BD

### Desafío 3: Consumer group con múltiples instancias

Ejecuta dos instancias de order-service en puertos diferentes:

```bash
# Terminal 1
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081

# Terminal 2
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8082
```

Ambas publicarán al mismo topic. Verifica con consumer CLI que todos los eventos llegan.

### Desafío 4: Monitoreo de eventos publicados

Crea endpoint que retorne estadísticas de eventos publicados:

```java
@RestController
@RequestMapping("/api/metrics")
public class MetricsController {

    private final OrderEventProducer producer;

    @GetMapping("/events-published")
    public ResponseEntity<Map<String, Long>> getEventsPublished() {
        // Implementar contador de eventos
        Map<String, Long> stats = new HashMap<>();
        stats.put("orders_placed_total", producer.getPublishedCount());
        return ResponseEntity.ok(stats);
    }
}
```

---

## Recursos adicionales

- [Event-Driven Microservices](https://www.confluent.io/blog/event-driven-microservices-with-spring-boot-and-kafka/)
- [Kafka Partitioning Deep Dive](https://www.confluent.io/blog/apache-kafka-producer-improvements-sticky-partitioner/)
- [Microservices Communication Patterns](https://microservices.io/patterns/communication-style/messaging.html)
- [Self-Contained Events](https://martinfowler.com/eaaDev/EventCollaboration.html)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
