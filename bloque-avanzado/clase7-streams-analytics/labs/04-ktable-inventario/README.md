# Lab 04: CQRS - Read model de inventario con KTable

---

## 1. Objetivo

Implementar el patrón CQRS (Command Query Responsibility Segregation) creando un read model de inventario usando KTable. El read model se actualiza automáticamente desde Kafka sin consultar la base de datos, demostrando eventual consistency.

---

## 2. Comandos a ejecutar

```bash
# Terminal 1: Crear topic con log compaction
docker exec -it kafka bash

kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.inventory.updated \
  --partitions 3 \
  --replication-factor 1 \
  --config cleanup.policy=compact

exit

# Terminal 2: Reiniciar analytics-service (para leer nuevo topic)
cd ~/workspace/analytics-service
mvn spring-boot:run

# Terminal 3: Crear órdenes hasta agotar stock
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 45,
    "customerName": "Cliente A",
    "customerEmail": "clientea@example.com",
    "totalAmount": 67500.00
  }'

# Esperar propagación
sleep 3

# Consultar productos con stock bajo
curl http://localhost:8083/api/analytics/inventory/low-stock?threshold=10
```

**Salida esperada**:

```json
[
  {
    "productId": 1,
    "availableStock": 5,
    "reservedStock": 45,
    "totalStock": 50
  }
]
```

---

## 3. Desglose del comando

| Comando/Parámetro | Descripción |
|-------------------|-------------|
| `kafka-topics --create` | Crear nuevo topic |
| `--topic ecommerce.inventory.updated` | Topic para eventos de inventario |
| `--partitions 3` | 3 particiones para paralelismo |
| `--config cleanup.policy=compact` | **Log compaction** - Solo retiene último mensaje por key |
| `GET /api/analytics/inventory/low-stock?threshold=10` | Consulta productos con stock < 10 |

---

## 4. Explicación detallada

### 4.1 Prerequisito: Modificar inventory-service

**IMPORTANTE**: Antes de crear el KTable, debemos modificar `inventory-service` (del Bloque Kafka, Clase 6) para publicar eventos de inventario.

**Ruta del proyecto**: `~/workspace/inventory-service/` (o donde hayas creado inventory-service en Clase 6)

#### 4.1.1 Crear InventoryUpdatedEvent.java en inventory-service

Navega al proyecto inventory-service y crea `src/main/java/dev/alefiengo/inventoryservice/kafka/event/InventoryUpdatedEvent.java`:

```java
package dev.alefiengo.inventoryservice.kafka.event;

import java.time.Instant;

public class InventoryUpdatedEvent {
    private Long productId;
    private Integer availableStock;
    private Integer reservedStock;
    private Integer totalStock;
    private Instant timestamp;

    public InventoryUpdatedEvent() {}

    public InventoryUpdatedEvent(Long productId, Integer availableStock,
                                  Integer reservedStock, Integer totalStock) {
        this.productId = productId;
        this.availableStock = availableStock;
        this.reservedStock = reservedStock;
        this.totalStock = totalStock;
        this.timestamp = Instant.now();
    }

    // Getters y Setters
    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public Integer getAvailableStock() {
        return availableStock;
    }

    public void setAvailableStock(Integer availableStock) {
        this.availableStock = availableStock;
    }

    public Integer getReservedStock() {
        return reservedStock;
    }

    public void setReservedStock(Integer reservedStock) {
        this.reservedStock = reservedStock;
    }

    public Integer getTotalStock() {
        return totalStock;
    }

    public void setTotalStock(Integer totalStock) {
        this.totalStock = totalStock;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }
}
```

#### 4.1.2 Modificar InventoryEventProducer.java

Agrega el siguiente método en `InventoryEventProducer.java`:

```java
public void publishInventoryUpdated(InventoryUpdatedEvent event) {
    log.info("Publishing InventoryUpdatedEvent: productId={}, availableStock={}",
             event.getProductId(), event.getAvailableStock());

    kafkaTemplate.send("ecommerce.inventory.updated",
                       event.getProductId().toString(),
                       event);
}
```

#### 4.1.3 Modificar InventoryService.processOrderPlaced()

Agrega la publicación del evento DESPUÉS de confirmar la orden:

```java
// 5. Publicar OrderConfirmedEvent
eventProducer.publishOrderConfirmed(confirmedEvent);

// 6. Publicar InventoryUpdatedEvent (NUEVO)
InventoryUpdatedEvent inventoryEvent = new InventoryUpdatedEvent(
    item.getProductId(),
    item.getAvailableStock(),
    item.getReservedStock(),
    item.getTotalStock()
);
eventProducer.publishInventoryUpdated(inventoryEvent);
```

**Reiniciar inventory-service** para aplicar cambios.

### 4.2 Crear topic con log compaction

```bash
docker exec -it kafka bash

kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.inventory.updated \
  --partitions 3 \
  --replication-factor 1 \
  --config cleanup.policy=compact

exit
```

**Punto crítico**: `cleanup.policy=compact` convierte el topic en un **changelog**:

- Kafka **retiene solo el último mensaje por key** (productId)
- Mensajes antiguos se descartan automáticamente
- **Perfecto para KTable** (solo importa el estado actual)

### 4.3 Copiar evento a analytics-service

Crea `src/main/java/dev/alefiengo/analyticsservice/model/event/InventoryUpdatedEvent.java` (copia exacta del evento de inventory-service):

```java
package dev.alefiengo.analyticsservice.model.event;

import java.time.Instant;

public class InventoryUpdatedEvent {
    private Long productId;
    private Integer availableStock;
    private Integer reservedStock;
    private Integer totalStock;
    private Instant timestamp;

    public InventoryUpdatedEvent() {}

    // Constructor completo + getters/setters (mismo que inventory-service)
    // ... (copiar completo)
}
```

### 4.4 Crear InventoryReadModelStream.java

Crea `src/main/java/dev/alefiengo/analyticsservice/streams/InventoryReadModelStream.java`:

```java
package dev.alefiengo.analyticsservice.streams;

import dev.alefiengo.analyticsservice.config.SerdeFactory;
import dev.alefiengo.analyticsservice.model.event.InventoryUpdatedEvent;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.Consumed;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.state.Stores;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.support.serializer.JsonSerde;

@Configuration
public class InventoryReadModelStream {

    private final SerdeFactory serdeFactory;

    public InventoryReadModelStream(SerdeFactory serdeFactory) {
        this.serdeFactory = serdeFactory;
    }

    @Bean
    public KTable<String, InventoryUpdatedEvent> inventoryTable(StreamsBuilder builder) {
        JsonSerde<InventoryUpdatedEvent> inventorySerde =
            serdeFactory.jsonSerde(InventoryUpdatedEvent.class);

        return builder.table(
            "ecommerce.inventory.updated",
            Consumed.with(Serdes.String(), inventorySerde),
            Materialized.<String, InventoryUpdatedEvent>as(
                    Stores.inMemoryKeyValueStore("inventory-read-model-store"))
                .withKeySerde(Serdes.String())
                .withValueSerde(inventorySerde)
        );
    }
}
```

**Puntos críticos**:

1. **Constructor injection con `SerdeFactory`**: Reutiliza el factory centralizado.
2. **`builder.table()`** crea un **KTable** (NO `builder.stream()`):
   - Solo retiene el **último valor por key** (productId)
   - Ideal para **estado actual** (no historial)
   - Se sincroniza automáticamente con topic compactado
3. **`Stores.inMemoryKeyValueStore()`**: State store en memoria explícito.
4. **`Materialized.<String, InventoryUpdatedEvent>as(...)`**: Tipos genéricos explícitos.
5. **`.withValueSerde(inventorySerde)`**: Usa el mismo serde para value (reutilización).

### 4.5 Crear InventoryItemResponse.java

Crea `src/main/java/dev/alefiengo/analyticsservice/model/dto/InventoryItemResponse.java`:

```java
package dev.alefiengo.analyticsservice.model.dto;

public class InventoryItemResponse {
    private Long productId;
    private Integer availableStock;
    private Integer reservedStock;
    private Integer totalStock;

    public InventoryItemResponse() {}

    public InventoryItemResponse(Long productId, Integer availableStock,
                                  Integer reservedStock, Integer totalStock) {
        this.productId = productId;
        this.availableStock = availableStock;
        this.reservedStock = reservedStock;
        this.totalStock = totalStock;
    }

    // Getters y Setters
    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public Integer getAvailableStock() {
        return availableStock;
    }

    public void setAvailableStock(Integer availableStock) {
        this.availableStock = availableStock;
    }

    public Integer getReservedStock() {
        return reservedStock;
    }

    public void setReservedStock(Integer reservedStock) {
        this.reservedStock = reservedStock;
    }

    public Integer getTotalStock() {
        return totalStock;
    }

    public void setTotalStock(Integer totalStock) {
        this.totalStock = totalStock;
    }
}
```

### 4.6 Actualizar AnalyticsQueryService.java

Agrega el siguiente método:

```java
import dev.alefiengo.analyticsservice.model.event.InventoryUpdatedEvent;
import dev.alefiengo.analyticsservice.model.dto.InventoryItemResponse;

public List<InventoryItemResponse> getLowStockProducts(int threshold) {
    KafkaStreams kafkaStreams = factoryBean.getKafkaStreams();

    ReadOnlyKeyValueStore<String, InventoryUpdatedEvent> store =
        kafkaStreams.store(
            StoreQueryParameters.fromNameAndType(
                "inventory-read-model-store",
                QueryableStoreTypes.keyValueStore()
            )
        );

    List<InventoryItemResponse> lowStock = new ArrayList<>();

    try (KeyValueIterator<String, InventoryUpdatedEvent> iterator = store.all()) {
        while (iterator.hasNext()) {
            KeyValue<String, InventoryUpdatedEvent> entry = iterator.next();
            InventoryUpdatedEvent event = entry.value;

            if (event.getAvailableStock() < threshold) {
                lowStock.add(new InventoryItemResponse(
                    event.getProductId(),
                    event.getAvailableStock(),
                    event.getReservedStock(),
                    event.getTotalStock()
                ));
            }
        }
    }

    return lowStock;
}
```

### 4.7 Actualizar AnalyticsController.java

Agrega el siguiente endpoint:

```java
import dev.alefiengo.analyticsservice.model.dto.InventoryItemResponse;

@GetMapping("/inventory/low-stock")
public ResponseEntity<List<InventoryItemResponse>> getLowStockProducts(
    @RequestParam(defaultValue = "10") int threshold
) {
    return ResponseEntity.ok(queryService.getLowStockProducts(threshold));
}
```

### 4.8 Compilar y ejecutar

```bash
mvn clean install
mvn spring-boot:run
```

**Verifica en logs**:

```
Materialized state store inventory-read-model-store
State transition to RUNNING
```

### 4.9 Prueba completa

```bash
# Crear órdenes hasta agotar stock de product 1
# Product 1 tiene 50 unidades iniciales
# Vender 45 → quedan 5

curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 45,
    "customerName": "Cliente A",
    "customerEmail": "clientea@example.com",
    "totalAmount": 67500.00
  }'

# Esperar propagación
sleep 3

# Consultar productos con stock < 10
curl http://localhost:8083/api/analytics/inventory/low-stock?threshold=10
```

**Respuesta esperada**:

```json
[
  {
    "productId": 1,
    "availableStock": 5,
    "reservedStock": 45,
    "totalStock": 50
  }
]
```

---

## 5. Conceptos aprendidos

- **KTable vs KStream**: KTable solo retiene último valor por key (changelog)
- **Log compaction**: `cleanup.policy=compact` descarta mensajes antiguos por key
- **CQRS (Command Query Responsibility Segregation)**: Read model separado del write model
- **Eventual consistency**: Read model se actualiza asíncronamente (segundos de retraso)
- **`builder.table()`** crea KTable (NO `builder.stream()`)
- **KTable se sincroniza automáticamente** con Kafka (no requiere `count()` ni `reduce()`)
- **Read model NO consulta base de datos** - Solo Kafka + state stores
- **Changelog topic**: Topic con `cleanup.policy=compact` perfecto para KTable

---

## 6. Troubleshooting

### Problema 1: Usar builder.stream() en vez de builder.table()

**Error**: Todos los eventos se procesan (duplica contadores).

```java
// ❌ INCORRECTO
KStream<String, InventoryUpdatedEvent> stream = builder.stream("ecommerce.inventory.updated");
```

**Solución**: SIEMPRE usar `builder.table()` para changelogs:

```java
// ✅ CORRECTO
KTable<String, InventoryUpdatedEvent> table = builder.table("ecommerce.inventory.updated");
```

### Problema 2: Topic sin cleanup.policy=compact

**Error**: Kafka NO descarta eventos viejos, state store crece indefinidamente.

**Solución**: Crear topic con log compaction:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.inventory.updated \
  --config cleanup.policy=compact
```

### Problema 3: Read model desactualizado

**Síntoma**: Consulta retorna datos viejos.

**Causa**: Propagación asíncrona (eventual consistency).

**Solución**: Esperar 1-2 segundos después de crear orden. Esto es NORMAL en arquitecturas event-driven.

### Problema 4: InventoryUpdatedEvent no se publica

**Síntoma**: Read model vacío, no se procesan eventos.

**Causa**: `inventory-service` no está publicando eventos.

**Solución**: Verificar:

1. Método `publishInventoryUpdated()` existe en `InventoryEventProducer`
2. Se llama en `InventoryService.processOrderPlaced()`
3. Topic existe: `docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list`

---

## 7. Desafío adicional

1. **Agrega un endpoint para consultar inventario de un producto específico**:

```java
@GetMapping("/inventory/{productId}")
public ResponseEntity<InventoryItemResponse> getProductInventory(@PathVariable Long productId) {
    // Implementar usando store.get(productId.toString())
}
```

2. **Implementa un endpoint para productos sin stock**:

```java
@GetMapping("/inventory/out-of-stock")
public ResponseEntity<List<InventoryItemResponse>> getOutOfStockProducts() {
    // Filtrar availableStock == 0
}
```

3. **Agrega un campo `lastUpdated` a InventoryItemResponse** para mostrar cuándo se actualizó el inventario.

---

## 8. Recursos adicionales

- [Kafka Streams KTable](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html#ktable)
- [Log Compaction](https://kafka.apache.org/documentation/#compaction)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Eventual Consistency](https://en.wikipedia.org/wiki/Eventual_consistency)

---

**Siguiente**: [Lab 05: Flujo completo end-to-end con analytics](../05-flujo-completo/README.md)
