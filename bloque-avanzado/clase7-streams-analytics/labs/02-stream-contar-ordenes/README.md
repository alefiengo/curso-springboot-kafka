# Lab 02: Stream básico - Contar órdenes por estado

---

## 1. Objetivo

Implementar el primer Kafka Stream para contar órdenes confirmadas y canceladas en tiempo real usando la operación stateful `count()`, y exponer los contadores vía REST API consultando state stores.

---

## 2. Comandos a ejecutar

```bash
# Terminal 1: Iniciar analytics-service
cd ~/workspace/analytics-service
mvn spring-boot:run

# Terminal 2: Crear órdenes (usando order-service)
# Orden con stock (será confirmada)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 5,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com",
    "totalAmount": 7500.00
  }'

# Orden sin stock (será cancelada)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 3,
    "quantity": 100,
    "customerName": "Maria Lopez",
    "customerEmail": "maria@example.com",
    "totalAmount": 8999.00
  }'

# Esperar 2-3 segundos para propagación
sleep 3

# Terminal 3: Consultar estadísticas
curl http://localhost:8083/api/analytics/orders/stats
```

**Salida esperada**:

```json
{
  "confirmedOrders": 1,
  "cancelledOrders": 1
}
```

---

## 3. Desglose del comando

| Comando | Descripción |
|---------|-------------|
| `mvn spring-boot:run` | Inicia analytics-service en puerto 8083 |
| `POST /api/orders` | Crea órdenes en order-service (8081) |
| `GET /api/analytics/orders/stats` | Consulta contadores de órdenes en analytics-service |
| `sleep 3` | Espera propagación de eventos a través de Kafka |

---

## 4. Explicación detallada

### 4.1 Crear OrderAnalyticsStream.java

Crea `src/main/java/dev/alefiengo/analyticsservice/streams/OrderAnalyticsStream.java`:

```java
package dev.alefiengo.analyticsservice.streams;

import dev.alefiengo.analyticsservice.config.SerdeFactory;
import dev.alefiengo.analyticsservice.model.event.OrderCancelledEvent;
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
public class OrderAnalyticsStream {

    private final SerdeFactory serdeFactory;

    public OrderAnalyticsStream(SerdeFactory serdeFactory) {
        this.serdeFactory = serdeFactory;
    }

    @Bean
    public KStream<String, OrderConfirmedEvent> orderConfirmedStream(StreamsBuilder builder) {
        JsonSerde<OrderConfirmedEvent> orderConfirmedSerde =
            serdeFactory.jsonSerde(OrderConfirmedEvent.class);

        KStream<String, OrderConfirmedEvent> stream = builder.stream(
            "ecommerce.orders.confirmed",
            Consumed.with(Serdes.String(), orderConfirmedSerde)
        );

        stream.groupBy(
                (key, value) -> "confirmed",
                Grouped.with(Serdes.String(), orderConfirmedSerde)
            )
            .count(
                Materialized.<String, Long>as(
                        Stores.inMemoryKeyValueStore("order-confirmed-count-store"))
                    .withKeySerde(Serdes.String())
                    .withValueSerde(Serdes.Long())
            );

        return stream;
    }

    @Bean
    public KStream<String, OrderCancelledEvent> orderCancelledStream(StreamsBuilder builder) {
        JsonSerde<OrderCancelledEvent> orderCancelledSerde =
            serdeFactory.jsonSerde(OrderCancelledEvent.class);

        KStream<String, OrderCancelledEvent> stream = builder.stream(
            "ecommerce.orders.cancelled",
            Consumed.with(Serdes.String(), orderCancelledSerde)
        );

        stream
            .groupBy(
                (key, value) -> "cancelled",
                Grouped.with(Serdes.String(), orderCancelledSerde)
            )
            .count(
                Materialized.<String, Long>as(
                        Stores.inMemoryKeyValueStore("order-cancelled-count-store"))
                    .withKeySerde(Serdes.String())
                    .withValueSerde(Serdes.Long())
            );

        return stream;
    }
}
```

**Puntos críticos**:

1. **Constructor injection con `SerdeFactory`**: Inyecta el factory centralizado creado en Lab 01.
2. **`serdeFactory.jsonSerde(OrderConfirmedEvent.class)`**: Crea JsonSerde con configuración desde `application.yml`.
3. **`builder.stream()`**: Crea un KStream desde el topic.
4. **`groupBy((key, value) -> "confirmed")`**: Agrupa TODOS los eventos bajo una sola key ("confirmed"). Esto permite contar todos los eventos juntos.
5. **`count()`**: Operación stateful que crea un state store automáticamente.
6. **`Stores.inMemoryKeyValueStore("order-confirmed-count-store")`**: Crea state store en memoria explícitamente.
7. **`Materialized.<String, Long>as(...)`**: Define tipos genéricos explícitos (String key, Long value).
8. **`.withKeySerde()` / `.withValueSerde()`**: Especifica serializadores explícitos para el state store.
9. **NO usamos método estático `jsonSerde()`**: Configuración viene de `SerdeFactory` component.

### 4.2 Crear OrderStatsResponse.java

Crea `src/main/java/dev/alefiengo/analyticsservice/model/dto/OrderStatsResponse.java`:

```java
package dev.alefiengo.analyticsservice.model.dto;

public class OrderStatsResponse {
    private Long confirmedOrders;
    private Long cancelledOrders;

    public OrderStatsResponse() {}

    public OrderStatsResponse(Long confirmedOrders, Long cancelledOrders) {
        this.confirmedOrders = confirmedOrders;
        this.cancelledOrders = cancelledOrders;
    }

    // Getters y Setters
    public Long getConfirmedOrders() {
        return confirmedOrders;
    }

    public void setConfirmedOrders(Long confirmedOrders) {
        this.confirmedOrders = confirmedOrders;
    }

    public Long getCancelledOrders() {
        return cancelledOrders;
    }

    public void setCancelledOrders(Long cancelledOrders) {
        this.cancelledOrders = cancelledOrders;
    }
}
```

### 4.3 Crear AnalyticsQueryService.java

Crea `src/main/java/dev/alefiengo/analyticsservice/service/AnalyticsQueryService.java`:

```java
package dev.alefiengo.analyticsservice.service;

import dev.alefiengo.analyticsservice.model.dto.OrderStatsResponse;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StoreQueryParameters;
import org.apache.kafka.streams.errors.InvalidStateStoreException;
import org.apache.kafka.streams.state.QueryableStoreTypes;
import org.apache.kafka.streams.state.ReadOnlyKeyValueStore;
import org.springframework.http.HttpStatus;
import org.springframework.kafka.config.StreamsBuilderFactoryBean;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class AnalyticsQueryService {

    private final StreamsBuilderFactoryBean factoryBean;

    public AnalyticsQueryService(StreamsBuilderFactoryBean factoryBean) {
        this.factoryBean = factoryBean;
    }

    public OrderStatsResponse getOrderStats() {
        KafkaStreams kafkaStreams = requireKafkaStreams();

        try {
            ReadOnlyKeyValueStore<String, Long> confirmedStore =
                kafkaStreams.store(
                    StoreQueryParameters.fromNameAndType(
                        "order-confirmed-count-store",
                        QueryableStoreTypes.keyValueStore()
                    )
                );

            ReadOnlyKeyValueStore<String, Long> cancelledStore =
                kafkaStreams.store(
                    StoreQueryParameters.fromNameAndType(
                        "order-cancelled-count-store",
                        QueryableStoreTypes.keyValueStore()
                    )
                );

            Long confirmedCount = confirmedStore.get("confirmed");
            Long cancelledCount = cancelledStore.get("cancelled");

            return new OrderStatsResponse(
                confirmedCount != null ? confirmedCount : 0L,
                cancelledCount != null ? cancelledCount : 0L
            );
        } catch (InvalidStateStoreException ex) {
            throw stateStoreUnavailable(ex);
        }
    }

    private KafkaStreams requireKafkaStreams() {
        KafkaStreams kafkaStreams = factoryBean.getKafkaStreams();
        if (kafkaStreams == null || !kafkaStreams.state().isRunningOrRebalancing()) {
            throw new ResponseStatusException(
                HttpStatus.SERVICE_UNAVAILABLE,
                "Kafka Streams no está listo para procesar consultas"
            );
        }
        return kafkaStreams;
    }

    private ResponseStatusException stateStoreUnavailable(RuntimeException cause) {
        return new ResponseStatusException(
            HttpStatus.SERVICE_UNAVAILABLE,
            "Las stores locales aún no están disponibles",
            cause
        );
    }
}
```

**Puntos críticos**:

1. **`requireKafkaStreams()`**: Método privado que valida que Kafka Streams esté en estado RUNNING antes de consultar stores.
2. **`kafkaStreams.state().isRunningOrRebalancing()`**: Verifica que el estado sea RUNNING o REBALANCING (estados válidos para queries).
3. **`ResponseStatusException` con HTTP 503**: Retorna error HTTP si Streams no está listo.
4. **`try-catch InvalidStateStoreException`**: Captura errores cuando stores no están disponibles.
5. **`stateStoreUnavailable()`**: Método helper para respuestas HTTP 503 consistentes.
6. **`StoreQueryParameters.fromNameAndType()`**: Especifica nombre y tipo de store.
7. **`confirmedStore.get("confirmed")`**: Consulta el contador por key.
8. **Validación de null**: Si la key no existe, retorna 0L (operador ternario).

**Ventajas del diseño robusto**:
- Evita `NullPointerException` si `kafkaStreams` es null
- Retorna HTTP 503 (Service Unavailable) en lugar de HTTP 500 (Internal Server Error)
- Cliente puede reintentar automáticamente con backoff
- Logs claros sobre el estado del servicio

### 4.4 Crear AnalyticsController.java

Crea `src/main/java/dev/alefiengo/analyticsservice/controller/AnalyticsController.java`:

```java
package dev.alefiengo.analyticsservice.controller;

import dev.alefiengo.analyticsservice.model.dto.OrderStatsResponse;
import dev.alefiengo.analyticsservice.service.AnalyticsQueryService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/analytics")
public class AnalyticsController {

    private final AnalyticsQueryService queryService;

    // Constructor injection - Spring 4.3+ no requiere @Autowired
    public AnalyticsController(AnalyticsQueryService queryService) {
        this.queryService = queryService;
    }

    @GetMapping("/orders/stats")
    public ResponseEntity<OrderStatsResponse> getOrderStats() {
        return ResponseEntity.ok(queryService.getOrderStats());
    }
}
```

### 4.5 Compilar y ejecutar

```bash
mvn clean install
mvn spring-boot:run
```

**Verifica en logs**:

```
StreamThread [analytics-service-stream-thread] started
State transition from CREATED to RUNNING
Materialized state store order-confirmed-count-store
Materialized state store order-cancelled-count-store
```

### 4.6 Prueba completa

**Crear órdenes**:

```bash
# Orden confirmada (product 1 tiene stock)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 5,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com",
    "totalAmount": 7500.00
  }'

# Orden cancelada (product 3 sin stock suficiente)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 3,
    "quantity": 100,
    "customerName": "Maria Lopez",
    "customerEmail": "maria@example.com",
    "totalAmount": 8999.00
  }'

# Esperar propagación
sleep 3

# Consultar estadísticas
curl http://localhost:8083/api/analytics/orders/stats
```

**Respuesta esperada**:

```json
{
  "confirmedOrders": 1,
  "cancelledOrders": 1
}
```

**Crear más órdenes y verificar que los contadores se actualizan automáticamente**.

---

## 5. Conceptos aprendidos

- **`builder.stream()`** crea un KStream desde un topic
- **`groupBy()`** agrupa eventos por key (crea KGroupedStream)
- **`count()`** es una operación **stateful** que crea un state store
- **`Materialized.as("name")`** nombra el state store para consultarlo después
- **`ReadOnlyKeyValueStore`** permite consultar state stores desde REST API
- **Interactive Queries**: Consultar state stores en tiempo real sin base de datos
- **Contadores se actualizan automáticamente** con cada evento nuevo
- **`trusted.packages`** es obligatorio para deserializar JSON con JsonSerde

---

## 6. Troubleshooting

### Problema 1: State store no existe

**Error**:
```
org.apache.kafka.streams.errors.InvalidStateStoreException:
Cannot get state store order-confirmed-count-store
```

**Causa**: Nombre incorrecto en query, o Kafka Streams no está en estado RUNNING.

**Solución**:

1. Verificar logs: `State transition to RUNNING`
2. Verificar nombre en `Materialized.as()` coincide con `store.get()`

### Problema 2: NullPointerException al consultar store

**Error**:
```
java.lang.NullPointerException: Cannot invoke "Long.longValue()" because the return value of "get()" is null
```

**Causa**: La key no existe en el state store (aún no ha llegado ningún evento).

**Solución**: Validar null:

```java
Long confirmedCount = confirmedStore.get("confirmed");
return confirmedCount != null ? confirmedCount : 0L;
```

### Problema 3: Deserialización falla

**Error**:
```
org.springframework.kafka.support.serializer.DeserializationException:
failed to deserialize; nested exception is java.lang.IllegalArgumentException:
The class 'dev.alefiengo.analyticsservice.model.event.OrderConfirmedEvent' is not in the trusted packages
```

**Causa**: `trusted.packages` no configurado en JsonSerde.

**Solución**: Siempre configurar en `jsonSerde()`:

```java
serde.configure(Map.of(
    "spring.json.trusted.packages", "dev.alefiengo.*"
), false);
```

### Problema 4: Warning "Could not autowire StreamsBuilderFactoryBean"

**Warning del IDE**:
```
Could not autowire. No beans of 'StreamsBuilderFactoryBean' type found.
```

**Causa**:
1. El IDE no detecta el bean porque es creado automáticamente por Spring Boot al tener `kafka-streams` en el classpath
2. El warning es falso - el bean SÍ existe en runtime

**Solución**:
- **Ignorar el warning** - el código compilará y funcionará correctamente
- Asegurarse de que `pom.xml` incluye AMBAS dependencias:
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
- Spring Boot autoconfigura `StreamsBuilderFactoryBean` cuando detecta `kafka-streams` en classpath

**Nota**: NO es necesario `@Autowired` en constructores (Spring 4.3+).

### Problema 5: Contador no se actualiza

**Error**: El contador siempre muestra 0 o valor antiguo.

**Causa**: Kafka Streams no está en estado RUNNING.

**Solución**: Verificar logs:

```
State transition from CREATED to RUNNING
```

Si no aparece, verificar:
- Kafka está corriendo: `docker compose ps`
- Topics existen: `docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list`
- Configuración de `application-id` es correcta

---

## 7. Desafío adicional

1. **Agrega un tercer contador** para órdenes PENDING (que aún no fueron confirmadas ni canceladas).

2. **Crea un endpoint** que retorne el total de órdenes (confirmed + cancelled + pending):

```java
@GetMapping("/orders/total")
public ResponseEntity<Long> getTotalOrders() {
    // Implementar
}
```

3. **Agrega logging** en OrderAnalyticsStream para ver cada evento procesado:

```java
stream.peek((key, value) ->
    log.info("Processing OrderConfirmedEvent: orderId={}", value.getOrderId())
);
```

---

## 8. Recursos adicionales

- [Kafka Streams DSL - count()](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html#aggregating)
- [Materialized State Stores](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html#materialized)
- [Interactive Queries](https://kafka.apache.org/35/documentation/streams/developer-guide/interactive-queries.html)
- [JsonSerde - Spring Kafka](https://docs.spring.io/spring-kafka/reference/html/#serdes)

---

**Siguiente**: [Lab 03: Stream con transformación - Top productos vendidos](../03-stream-top-productos/README.md)
