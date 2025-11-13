# Lab 03: Stream con transformación - Top productos vendidos

---

## 1. Objetivo

Implementar un Kafka Stream para calcular la cantidad total vendida por producto usando la operación stateful `reduce()` con re-keying, y exponer un ranking de productos más vendidos vía REST API.

---

## 2. Comandos a ejecutar

```bash
# Crear órdenes para diferentes productos
# Product 1: 5 unidades
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 5,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com",
    "totalAmount": 7500.00
  }'

# Product 1: 3 unidades más (total: 5 + 3 = 8)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 3,
    "customerName": "Pedro Gomez",
    "customerEmail": "pedro@example.com",
    "totalAmount": 4500.00
  }'

# Product 2: 10 unidades
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 2,
    "quantity": 10,
    "customerName": "Ana Martinez",
    "customerEmail": "ana@example.com",
    "totalAmount": 999.90
  }'

# Esperar propagación
sleep 3

# Consultar top productos
curl http://localhost:8083/api/analytics/products/top
```

**Salida esperada**:

```json
[
  {
    "productId": 2,
    "totalSold": 10
  },
  {
    "productId": 1,
    "totalSold": 8
  }
]
```

---

## 3. Desglose del comando

| Comando | Descripción |
|---------|-------------|
| `POST /api/orders` (productId: 1, quantity: 5) | Primera orden de producto 1 |
| `POST /api/orders` (productId: 1, quantity: 3) | Segunda orden de producto 1 (acumula) |
| `POST /api/orders` (productId: 2, quantity: 10) | Orden de producto 2 |
| `GET /api/analytics/products/top` | Consulta ranking de productos más vendidos |

---

## 4. Explicación detallada

### 4.1 Crear ProductAnalyticsStream.java

Crea `src/main/java/dev/alefiengo/analyticsservice/streams/ProductAnalyticsStream.java`:

```java
package dev.alefiengo.analyticsservice.streams;

import dev.alefiengo.analyticsservice.config.SerdeFactory;
import dev.alefiengo.analyticsservice.model.event.OrderConfirmedEvent;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.Consumed;
import org.apache.kafka.streams.kstream.Grouped;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.state.Stores;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.support.serializer.JsonSerde;

@Configuration
public class ProductAnalyticsStream {

    private final SerdeFactory serdeFactory;

    public ProductAnalyticsStream(SerdeFactory serdeFactory) {
        this.serdeFactory = serdeFactory;
    }

    @Bean
    public KStream<String, OrderConfirmedEvent> productSalesStream(StreamsBuilder builder) {
        JsonSerde<OrderConfirmedEvent> orderConfirmedSerde =
            serdeFactory.jsonSerde(OrderConfirmedEvent.class);

        KStream<String, OrderConfirmedEvent> stream = builder.stream(
            "ecommerce.orders.confirmed",
            Consumed.with(Serdes.String(), orderConfirmedSerde)
        );

        stream
            .selectKey((orderId, event) -> event.getProductId().toString())
            .mapValues(OrderConfirmedEvent::getQuantity)
            .groupByKey(Grouped.with(Serdes.String(), Serdes.Integer()))
            .reduce(
                Integer::sum,
                Materialized.<String, Integer>as(
                        Stores.inMemoryKeyValueStore("product-sales-store"))
                    .withKeySerde(Serdes.String())
                    .withValueSerde(Serdes.Integer())
            );

        return stream;
    }
}
```

**Puntos críticos**:

1. **Constructor injection con `SerdeFactory`**: Reutiliza el factory centralizado.
2. **`selectKey((orderId, event) -> event.getProductId().toString())`**: **Re-keying** - Cambia la key del mensaje de `orderId` a `productId`. Esto es necesario para agrupar por producto.
3. **`mapValues(OrderConfirmedEvent::getQuantity)`**: Method reference que transforma el value de `OrderConfirmedEvent` a `Integer` (solo la cantidad).
4. **`groupByKey()`**: Agrupa por la nueva key (`productId`).
5. **`reduce(Integer::sum, ...)`**: Agrega valores sumándolos. Equivalente a `(oldValue, newValue) -> oldValue + newValue`.
6. **`Stores.inMemoryKeyValueStore("product-sales-store")`**: State store en memoria explícito.
7. **`Materialized.<String, Integer>as(...)`**: Tipos genéricos explícitos (String productId, Integer quantity).
8. **`.withKeySerde()` / `.withValueSerde()`**: Serializadores explícitos para el state store.

### 4.2 Crear ProductStatsResponse.java

Crea `src/main/java/dev/alefiengo/analyticsservice/model/dto/ProductStatsResponse.java`:

```java
package dev.alefiengo.analyticsservice.model.dto;

public class ProductStatsResponse {
    private Long productId;
    private Integer totalSold;

    public ProductStatsResponse() {}

    public ProductStatsResponse(Long productId, Integer totalSold) {
        this.productId = productId;
        this.totalSold = totalSold;
    }

    // Getters y Setters
    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public Integer getTotalSold() {
        return totalSold;
    }

    public void setTotalSold(Integer totalSold) {
        this.totalSold = totalSold;
    }
}
```

### 4.3 Actualizar AnalyticsQueryService.java

Agrega el siguiente método en `AnalyticsQueryService.java`:

```java
import dev.alefiengo.analyticsservice.model.dto.ProductStatsResponse;
import org.apache.kafka.streams.KeyValue;
import org.apache.kafka.streams.state.KeyValueIterator;
import java.util.ArrayList;
import java.util.List;

public List<ProductStatsResponse> getTopProducts() {
    KafkaStreams kafkaStreams = requireKafkaStreams();

    try {
        ReadOnlyKeyValueStore<String, Integer> store =
            kafkaStreams.store(
                StoreQueryParameters.fromNameAndType(
                    "product-sales-store",
                    QueryableStoreTypes.keyValueStore()
                )
            );

        List<ProductStatsResponse> products = new ArrayList<>();

        try (KeyValueIterator<String, Integer> iterator = store.all()) {
            while (iterator.hasNext()) {
                KeyValue<String, Integer> entry = iterator.next();
                products.add(new ProductStatsResponse(
                    Long.parseLong(entry.key),
                    entry.value
                ));
            }
        }

        products.sort((a, b) -> Integer.compare(b.getTotalSold(), a.getTotalSold()));

        return products;
    } catch (InvalidStateStoreException ex) {
        throw stateStoreUnavailable(ex);
    }
}
```

**Puntos críticos**:

1. **`requireKafkaStreams()`**: Valida que Kafka Streams esté RUNNING antes de consultar (método definido en Lab 02).
2. **`try-catch InvalidStateStoreException`**: Manejo robusto de errores cuando store no está disponible.
3. **`try-with-resources`**: Cierra automáticamente el iterator (evita memory leaks).
4. **`store.all()`**: Itera TODOS los registros del state store.
5. **`Long.parseLong(entry.key)`**: Convertir key (String) a Long (productId).
   - **ASUME que todas las keys son numéricas** (válido porque re-keying usa `productId.toString()`)
   - Si hubiera keys no numéricas, usar try-catch para `NumberFormatException`
6. **Ordenamiento en memoria**: Kafka Streams NO ordena, lo hacemos en Java con `List.sort()`.
7. **Reutiliza `stateStoreUnavailable()`**: Método helper definido en Lab 02.

### 4.4 Actualizar AnalyticsController.java

Agrega el siguiente endpoint en `AnalyticsController.java`:

```java
import dev.alefiengo.analyticsservice.model.dto.ProductStatsResponse;
import java.util.List;

@GetMapping("/products/top")
public ResponseEntity<List<ProductStatsResponse>> getTopProducts() {
    return ResponseEntity.ok(queryService.getTopProducts());
}
```

### 4.5 Compilar y ejecutar

```bash
mvn clean install
mvn spring-boot:run
```

**Verifica en logs**:

```
Materialized state store product-sales-store
State transition to RUNNING
```

### 4.6 Prueba completa

**Crear órdenes**:

```bash
# Product 1: 5 unidades
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 5,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com",
    "totalAmount": 7500.00
  }'

# Product 1: 3 unidades más (total: 8)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 3,
    "customerName": "Pedro Gomez",
    "customerEmail": "pedro@example.com",
    "totalAmount": 4500.00
  }'

# Product 2: 10 unidades
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 2,
    "quantity": 10,
    "customerName": "Ana Martinez",
    "customerEmail": "ana@example.com",
    "totalAmount": 999.90
  }'

# Esperar propagación
sleep 3

# Consultar top productos
curl http://localhost:8083/api/analytics/products/top
```

**Respuesta esperada**:

```json
[
  {
    "productId": 2,
    "totalSold": 10
  },
  {
    "productId": 1,
    "totalSold": 8
  }
]
```

**Crear más órdenes y verificar que las cantidades se acumulan**.

---

## 5. Conceptos aprendidos

- **Re-keying con `selectKey()`**: Cambiar la key del mensaje para agrupar por otro campo
- **`mapValues()`**: Transformar solo el value (mantiene la key)
- **`groupByKey()`**: Agrupar por la key actual (requiere re-keying previo si se quiere agrupar por otro campo)
- **`reduce()`**: Operación stateful para agregar valores (sum, max, min, etc.)
- **`KeyValueIterator`**: Permite iterar sobre todos los registros de un state store
- **SIEMPRE cerrar iterator**: Usar `try-finally` o `try-with-resources`
- **Ordenamiento en memoria**: Kafka Streams NO ordena, se hace en la aplicación
- **Agregaciones acumulativas**: `reduce()` suma valores nuevos al estado existente

---

## 6. Troubleshooting

### Problema 1: Re-keying sin selectKey()

**Error**: Intentas agrupar por un campo que no es la key.

```java
// ❌ INCORRECTO
stream.groupBy((key, value) -> value.getProductId().toString())
```

**Solución**: SIEMPRE usar `selectKey()` antes de `groupByKey()`:

```java
// ✅ CORRECTO
stream
    .selectKey((key, value) -> value.getProductId().toString())
    .groupByKey()
```

### Problema 2: Olvidar cerrar iterator

**Error**: Memory leak si no se cierra el iterator.

**Solución**: SIEMPRE usar `try-finally`:

```java
KeyValueIterator<String, Integer> iterator = store.all();
try {
    while (iterator.hasNext()) {
        // Procesar
    }
} finally {
    iterator.close();
}
```

O `try-with-resources` (Java 7+):

```java
try (KeyValueIterator<String, Integer> iterator = store.all()) {
    while (iterator.hasNext()) {
        // Procesar
    }
}
```

### Problema 3: ConcurrentModificationException

**Error**: Iterar mientras llegan eventos nuevos puede causar excepciones.

**Solución**: El `ReadOnlyKeyValueStore` es seguro para lectura, pero usar `try-finally` para cerrar iterator correctamente.

### Problema 4: NumberFormatException al parsear productId

**Error**:
```
java.lang.NumberFormatException: For input string: "abc"
```

**Causa**: Key no es un número válido.

**Solución**: Validar antes de `Long.parseLong()`:

```java
try {
    Long productId = Long.parseLong(entry.key);
    products.add(new ProductStatsResponse(productId, entry.value));
} catch (NumberFormatException e) {
    log.warn("Invalid productId: {}", entry.key);
}
```

---

## 7. Desafío adicional

1. **Filtra productos con ventas < 5 unidades** antes de retornar:

```java
products.stream()
    .filter(p -> p.getTotalSold() >= 5)
    .collect(Collectors.toList());
```

2. **Agrega un endpoint para consultar ventas de un producto específico**:

```java
@GetMapping("/products/{productId}/sales")
public ResponseEntity<Integer> getProductSales(@PathVariable Long productId) {
    // Implementar
}
```

3. **Implementa un límite de top N productos**:

```java
@GetMapping("/products/top")
public ResponseEntity<List<ProductStatsResponse>> getTopProducts(
    @RequestParam(defaultValue = "10") int limit
) {
    // Implementar con .stream().limit(limit)
}
```

---

## 8. Recursos adicionales

- [Kafka Streams DSL - reduce()](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html#aggregating)
- [Re-keying with selectKey()](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html#streams-developer-guide-dsl-transformations-stateless-selectkey)
- [KeyValueIterator](https://kafka.apache.org/35/javadoc/org/apache/kafka/streams/state/KeyValueIterator.html)
- [Interactive Queries](https://kafka.apache.org/35/documentation/streams/developer-guide/interactive-queries.html)

---

**Siguiente**: [Lab 04: CQRS - Read model de inventario con KTable](../04-ktable-inventario/README.md)
