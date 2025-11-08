# Cheatsheet - Clase 6: Consumidores e Integración

Comandos y configuraciones esenciales para consumidores Kafka y gestión de consumer groups.

---

## Comandos Kafka CLI

### Crear Topics

```bash
# Entrar al contenedor de Kafka
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

# Listar todos los topics
kafka-topics --bootstrap-server localhost:9092 --list

# Describir un topic específico
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.orders.placed
```

### Consumer Groups

```bash
# Listar todos los consumer groups
kafka-consumer-groups --bootstrap-server localhost:9092 --list

# Describir un consumer group específico
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe

# Ver lag (mensajes pendientes) de un grupo
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-service \
  --describe

# Resetear offsets de un grupo (CUIDADO en producción!)
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --reset-offsets \
  --to-earliest \
  --topic ecommerce.orders.placed \
  --execute

# Resetear a offset específico
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --reset-offsets \
  --to-offset 10 \
  --topic ecommerce.orders.placed \
  --execute
```

### Consumir Mensajes desde CLI

```bash
# Consumer simple
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning

# Consumer con keys
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"

# Consumer con headers
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --property print.headers=true

# Consumer con timestamps
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --property print.timestamp=true

# Consumer desde offset específico (partición 0, offset 5)
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --partition 0 \
  --offset 5

# Consumer solo mensajes nuevos (desde ahora)
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed
```

---

## Configuración Spring Kafka (application.yml)

### Consumer Básico

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
```

### Consumer con Múltiples Eventos

```yaml
spring:
  kafka:
    consumer:
      properties:
        spring.json.type.mapping: |
          orderPlacedEvent:dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent,
          orderConfirmedEvent:dev.alefiengo.orderservice.kafka.event.OrderConfirmedEvent,
          orderCancelledEvent:dev.alefiengo.orderservice.kafka.event.OrderCancelledEvent
```

### Producer + Consumer en el Mismo Servicio

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

    # Configuración de producer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

    # Configuración de consumer
    consumer:
      group-id: inventory-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.type.mapping: orderPlacedEvent:dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent
        spring.json.trusted.packages: dev.alefiengo.*
```

---

## Código Java

### Consumer Básico

```java
@Component
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    @KafkaListener(
        topics = "ecommerce.orders.placed",
        groupId = "inventory-service"
    )
    public void consumeOrderPlaced(OrderPlacedEvent event) {
        log.info("Received OrderPlacedEvent: orderId={}", event.getOrderId());
        // Procesar evento
    }
}
```

### Consumer con Múltiples Topics

```java
@KafkaListener(
    topics = {"ecommerce.orders.confirmed", "ecommerce.orders.cancelled"},
    groupId = "order-service"
)
public void consumeOrderEvents(String message) {
    log.info("Received: {}", message);
}
```

### Consumer con Manual Acknowledgment

```java
@KafkaListener(
    topics = "ecommerce.orders.placed",
    groupId = "inventory-service"
)
public void consume(OrderPlacedEvent event, Acknowledgment ack) {
    try {
        // Procesar evento
        processEvent(event);

        // Confirmar manualmente
        ack.acknowledge();
    } catch (Exception e) {
        log.error("Error processing event", e);
        // No hacer acknowledge → Kafka reintentará
    }
}
```

### Consumer con Headers

```java
@KafkaListener(topics = "ecommerce.orders.placed")
public void consume(
    @Payload OrderPlacedEvent event,
    @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
    @Header(KafkaHeaders.OFFSET) long offset
) {
    log.info("Received from partition {} at offset {}", partition, offset);
}
```

---

## Comandos Maven

```bash
# Compilar inventory-service
cd ~/workspace/inventory-service
mvn clean install

# Ejecutar inventory-service (puerto 8082)
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8082

# Ejecutar con perfil específico
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Ejecutar con variables de entorno
KAFKA_BOOTSTRAP_SERVERS=kafka:9092 mvn spring-boot:run
```

---

## Comandos Docker y PostgreSQL

```bash
# Verificar servicios corriendo
docker compose ps

# Crear base de datos para inventory-service
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_inventory;"

# Listar bases de datos
docker exec -it postgres psql -U postgres -c "\l"

# Conectar a base de datos específica
docker exec -it postgres psql -U postgres -d ecommerce_inventory

# Ver tablas de inventory
docker exec -it postgres psql -U postgres -d ecommerce_inventory -c "\dt"

# Ver contenido de inventory_items
docker exec -it postgres psql -U postgres -d ecommerce_inventory -c "SELECT * FROM inventory_items;"

# Ver estado de órdenes
docker exec -it postgres psql -U postgres -d ecommerce_orders -c "SELECT id, product_id, quantity, status, created_at FROM customer_orders;"
```

---

## Comandos cURL

### inventory-service

```bash
# Crear item de inventario
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "initialStock": 50
  }'

# Listar items de inventario
curl http://localhost:8082/api/inventory

# Consultar stock de un producto
curl http://localhost:8082/api/inventory/product/1
```

### order-service

```bash
# Crear orden (trigger flujo completo)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 2,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com",
    "totalAmount": 3000.00
  }'

# Ver estado de una orden
curl http://localhost:8081/api/orders/1

# Listar todas las órdenes
curl http://localhost:8081/api/orders
```

---

## Verificación de Flujo Completo

```bash
# Terminal 1: Monitorear logs de inventory-service
cd ~/workspace/inventory-service
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8082

# Terminal 2: Monitorear logs de order-service
cd ~/workspace/order-service
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081

# Terminal 3: Consumer de orders.placed
docker exec -it kafka bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning \
  --property print.key=true

# Terminal 4: Consumer de orders.confirmed
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --property print.key=true

# Terminal 5: Consumer de orders.cancelled
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.cancelled \
  --from-beginning \
  --property print.key=true

# Terminal 6: Crear órdenes y verificar
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":2,"customerName":"Test","customerEmail":"test@example.com","totalAmount":100}'
```

---

## Troubleshooting Rápido

```bash
# Verificar que Kafka está corriendo
docker compose ps | grep kafka

# Verificar que PostgreSQL está corriendo
docker compose ps | grep postgres

# Ver logs de Kafka
docker compose logs -f kafka | grep -i error

# Ver logs de PostgreSQL
docker compose logs -f postgres | grep -i error

# Verificar que un topic existe
kafka-topics --bootstrap-server localhost:9092 --list | grep orders

# Ver consumer groups activos
kafka-consumer-groups --bootstrap-server localhost:9092 --list

# Ver lag de un consumer group
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe

# Borrar un topic (SOLO EN DESARROLLO!)
kafka-topics --bootstrap-server localhost:9092 \
  --delete \
  --topic ecommerce.orders.placed
```

---

## Glosario Técnico

| Término Español | English Term | Comando/Concepto |
|-----------------|--------------|------------------|
| Grupo de consumidores | Consumer Group | `--group inventory-service` |
| Desplazamiento | Offset | `--reset-offsets --to-offset 10` |
| Retraso | Lag | Mensajes pendientes de procesar |
| Desde el inicio | From beginning | `--from-beginning` |
| Desde ahora | From latest | (sin flag, por defecto) |
| Reiniciar desplazamientos | Reset offsets | `--reset-offsets` |
| Confirmar manualmente | Manual acknowledge | `ack.acknowledge()` |
| Deserialización | Deserialization | JsonDeserializer |
| Paquetes confiables | Trusted packages | `spring.json.trusted.packages` |