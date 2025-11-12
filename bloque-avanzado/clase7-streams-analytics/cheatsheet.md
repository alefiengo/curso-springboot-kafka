# Cheatsheet - Clase 7: Kafka Streams

Referencia rápida de comandos, patrones, y configuraciones para Kafka Streams y analytics.

---

## Tabla de contenidos

1. [Configuración Básica](#configuración-básica)
2. [KStream - Operaciones Comunes](#kstream---operaciones-comunes)
3. [KTable - Operaciones Comunes](#ktable---operaciones-comunes)
4. [State Stores](#state-stores)
5. [Interactive Queries](#interactive-queries)
6. [Kafka CLI - Topics](#kafka-cli---topics)
7. [Troubleshooting Rápido](#troubleshooting-rápido)
8. [Patrones Comunes](#patrones-comunes)

---

## Configuración Básica

### application.yml

```yaml
spring:
  application:
    name: analytics-service
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    streams:
      application-id: analytics-service  # DEBE ser único
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

### Dependencias Maven

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

### JsonSerde Configuración

```java
private static <T> JsonSerde<T> jsonSerde(Class<T> targetClass) {
    JsonSerde<T> serde = new JsonSerde<>(targetClass);
    serde.configure(Map.of(
        "spring.json.trusted.packages", "dev.alefiengo.*"
    ), false);
    return serde;
}
```

---

## KStream - Operaciones Comunes

### Crear KStream

```java
KStream<String, OrderEvent> stream = builder.stream(
    "ecommerce.orders.placed",
    Consumed.with(Serdes.String(), jsonSerde(OrderEvent.class))
);
```

### Operaciones Stateless

```java
// Filtrar
stream.filter((key, value) -> value.getTotalAmount() > 100);

// Mapear value
stream.mapValues(order -> order.getCustomerName());

// Mapear key y value
stream.map((key, value) ->
    KeyValue.pair(value.getProductId().toString(), value.getQuantity())
);

// Cambiar solo la key (re-keying)
stream.selectKey((key, value) -> value.getProductId().toString());

// Transformar un evento en múltiples
stream.flatMapValues(order -> order.getItems());

// Ver eventos sin modificar (debugging)
stream.peek((key, value) -> log.info("Key: {}, Value: {}", key, value));
```

### Operaciones Stateful

```java
// Contar eventos
stream
    .groupBy((key, value) -> "confirmed")
    .count(Materialized.as("confirmed-count-store"));

// Sumar valores
stream
    .selectKey((key, value) -> value.getProductId().toString())
    .mapValues(order -> order.getQuantity())
    .groupByKey()
    .reduce(
        Integer::sum,
        Materialized.as("product-sales-store")
    );

// Agregación custom
stream
    .groupByKey()
    .aggregate(
        () -> 0,  // Initializer
        (key, value, aggregate) -> aggregate + value.getQuantity(),  // Aggregator
        Materialized.as("total-quantity-store")
    );
```

---

## KTable - Operaciones Comunes

### Crear KTable

```java
KTable<String, InventoryEvent> table = builder.table(
    "ecommerce.inventory.updated",
    Consumed.with(Serdes.String(), jsonSerde(InventoryEvent.class)),
    Materialized.as("inventory-read-model-store")
);
```

### Operaciones KTable

```java
// Filtrar
table.filter((key, value) -> value.getAvailableStock() < 10);

// Mapear values
table.mapValues(inventory -> inventory.getAvailableStock());

// Convertir KTable a KStream
KStream<String, InventoryEvent> stream = table.toStream();
```

### KStream a KTable

```java
// Agrupar y contar (produce KTable)
KTable<String, Long> countTable = stream
    .groupBy((key, value) -> "all")
    .count(Materialized.as("count-store"));
```

---

## State Stores

### Crear State Store

```java
// En stream processing
stream
    .groupByKey()
    .count(Materialized.as("my-store-name"));

// Con configuración específica
stream
    .groupByKey()
    .reduce(
        Integer::sum,
        Materialized.<String, Integer, KeyValueStore<Bytes, byte[]>>as("sales-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Integer())
    );
```

### Limpiar State Stores (Desarrollo)

```bash
# Borrar state stores locales
rm -rf /tmp/kafka-streams/*

# Reiniciar aplicación (reconstruye desde changelog)
mvn spring-boot:run
```

---

## Interactive Queries

### Consultar State Store - Valor Único

```java
@Service
public class AnalyticsQueryService {

    private final StreamsBuilderFactoryBean factoryBean;

    // Constructor injection - no requiere @Autowired
    public AnalyticsQueryService(StreamsBuilderFactoryBean factoryBean) {
        this.factoryBean = factoryBean;
    }

    public Long getConfirmedOrdersCount() {
        KafkaStreams kafkaStreams = factoryBean.getKafkaStreams();

        ReadOnlyKeyValueStore<String, Long> store =
            kafkaStreams.store(
                StoreQueryParameters.fromNameAndType(
                    "confirmed-count-store",
                    QueryableStoreTypes.keyValueStore()
                )
            );

        Long count = store.get("confirmed");
        return count != null ? count : 0L;
    }
}
```

### Iterar Todos los Valores

```java
public List<ProductStats> getAllProducts() {
    ReadOnlyKeyValueStore<String, Integer> store =
        kafkaStreams.store(
            StoreQueryParameters.fromNameAndType(
                "product-sales-store",
                QueryableStoreTypes.keyValueStore()
            )
        );

    List<ProductStats> products = new ArrayList<>();

    try (KeyValueIterator<String, Integer> iterator = store.all()) {
        while (iterator.hasNext()) {
            KeyValue<String, Integer> entry = iterator.next();
            products.add(new ProductStats(
                Long.parseLong(entry.key),
                entry.value
            ));
        }
    }

    return products;
}
```

### Verificar Estado del Stream

```java
public boolean isStreamsRunning() {
    KafkaStreams kafkaStreams = factoryBean.getKafkaStreams();
    return kafkaStreams.state() == KafkaStreams.State.RUNNING;
}
```

---

## Kafka CLI - Topics

### Crear Topic con Log Compaction

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

### Listar Topics

```bash
docker exec -it kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --list
```

### Describir Topic

```bash
docker exec -it kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.orders.confirmed
```

### Consumir Mensajes

```bash
# Desde el inicio
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning

# Con key
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"
```

### Ver Consumer Groups

```bash
# Listar consumer groups
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --list

# Describir consumer group
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group analytics-service \
  --describe
```

---

## Troubleshooting Rápido

### State Store No Existe

**Error**: `InvalidStateStoreException: Cannot get state store`

**Solución**:
1. Verificar nombre coincide entre `Materialized.as()` y `store.get()`
2. Verificar Kafka Streams está en estado RUNNING:
```java
kafkaStreams.state() == State.RUNNING
```

### Deserialización Falla

**Error**: `DeserializationException: class X is not in the trusted packages`

**Solución**: Agregar trusted packages en JsonSerde:
```java
serde.configure(Map.of(
    "spring.json.trusted.packages", "dev.alefiengo.*"
), false);
```

### Streams No Inicia

**Error**: `StreamThread died`

**Solución**:
1. Verificar Kafka está corriendo: `docker compose ps`
2. Verificar topics existen: `docker exec -it kafka kafka-topics --list`
3. Revisar logs: `grep ERROR logs/analytics-service.log`

### State Store Retorna Null

**Causa**: Key no existe en state store (normal si no ha llegado ningún evento).

**Solución**: Validar null:
```java
Long count = store.get("confirmed");
return count != null ? count : 0L;
```

### Limpiar State Stores Corruptos

```bash
# Detener aplicación
# Borrar state stores
rm -rf /tmp/kafka-streams/*
# Reiniciar aplicación (reconstruye desde Kafka)
mvn spring-boot:run
```

---

## Patrones Comunes

### Patrón: Contador Global

```java
@Bean
public KStream<String, OrderEvent> orderStream(StreamsBuilder builder) {
    KStream<String, OrderEvent> stream = builder.stream("ecommerce.orders.confirmed");

    stream
        .groupBy((key, value) -> "all")  // Agrupar todos bajo una key
        .count(Materialized.as("total-orders-store"));

    return stream;
}
```

### Patrón: Contador por Campo

```java
@Bean
public KStream<String, OrderEvent> productStream(StreamsBuilder builder) {
    KStream<String, OrderEvent> stream = builder.stream("ecommerce.orders.confirmed");

    stream
        .selectKey((key, value) -> value.getProductId().toString())  // Re-key
        .groupByKey()
        .count(Materialized.as("orders-per-product-store"));

    return stream;
}
```

### Patrón: Suma de Valores

```java
@Bean
public KStream<String, OrderEvent> salesStream(StreamsBuilder builder) {
    KStream<String, OrderEvent> stream = builder.stream("ecommerce.orders.confirmed");

    stream
        .selectKey((key, value) -> value.getProductId().toString())
        .mapValues(order -> order.getQuantity())
        .groupByKey(Grouped.with(Serdes.String(), Serdes.Integer()))
        .reduce(Integer::sum, Materialized.as("total-sold-store"));

    return stream;
}
```

### Patrón: Filtro + Agregación

```java
@Bean
public KStream<String, OrderEvent> highValueStream(StreamsBuilder builder) {
    KStream<String, OrderEvent> stream = builder.stream("ecommerce.orders.confirmed");

    stream
        .filter((key, value) -> value.getTotalAmount() > 1000)  // Solo high-value
        .groupBy((key, value) -> "high-value")
        .count(Materialized.as("high-value-orders-store"));

    return stream;
}
```

### Patrón: Read Model con KTable

```java
@Bean
public KTable<String, InventoryEvent> inventoryTable(StreamsBuilder builder) {
    return builder.table(
        "ecommerce.inventory.updated",
        Consumed.with(Serdes.String(), jsonSerde(InventoryEvent.class)),
        Materialized.as("inventory-read-model-store")
    );
}
```

---

## Comandos de Aplicación

### Iniciar analytics-service

```bash
cd ~/workspace/analytics-service
mvn spring-boot:run
```

### Verificar Health

```bash
curl http://localhost:8083/actuator/health
```

### Consultar Analytics

```bash
# Estadísticas de órdenes
curl http://localhost:8083/api/analytics/orders/stats

# Top productos
curl http://localhost:8083/api/analytics/products/top

# Stock bajo
curl http://localhost:8083/api/analytics/inventory/low-stock?threshold=10
```

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Flujo | Stream | Secuencia infinita de eventos |
| Tabla | Table | Vista materializada (último valor por key) |
| Almacén de estado | State Store | Almacenamiento local fault-tolerant |
| Operación sin estado | Stateless Operation | map, filter, selectKey |
| Operación con estado | Stateful Operation | count, reduce, aggregate |
| Re-keying | Re-keying | Cambiar key con selectKey() |
| Compactación de log | Log Compaction | Retener solo último valor por key |
| Consultas interactivas | Interactive Queries | Consultar state stores via REST |
| Topología | Topology | Grafo de operaciones de procesamiento |

---

## Referencias Rápidas

### Documentación Oficial

- [Kafka Streams DSL](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html)
- [Spring Kafka Streams](https://docs.spring.io/spring-kafka/reference/html/#kafka-streams)
- [State Stores](https://kafka.apache.org/35/documentation/streams/developer-guide/processor-api.html#state-stores)

### Recursos del Curso

- [Conceptos Clase 7](CONCEPTOS.md)
- [FAQ Clase 7](FAQ.md)
- [Labs Clase 7](labs/)

---

**Última Actualización**: Noviembre 2025
