# Tarea - Clase 7: Kafka Streams y Analytics

---

## Objetivo

Implementar un endpoint de analytics que muestre los **top 5 clientes** por cantidad de órdenes confirmadas, utilizando Kafka Streams para procesar eventos en tiempo real.

---

## Descripción del desafío

Debes agregar un nuevo endpoint a `analytics-service` que permita consultar los clientes más activos del sistema basándose en la cantidad de órdenes que han realizado.

### Endpoint requerido

```
GET /api/analytics/customers/top
```

### Respuesta esperada

```json
[
  {
    "customerName": "Juan Perez",
    "ordersCount": 15
  },
  {
    "customerName": "Maria Lopez",
    "ordersCount": 12
  },
  {
    "customerName": "Pedro Gomez",
    "ordersCount": 8
  },
  {
    "customerName": "Ana Martinez",
    "ordersCount": 5
  },
  {
    "customerName": "Carlos Rodriguez",
    "ordersCount": 3
  }
]
```

---

## Requisitos técnicos

### 1. Crear CustomerStatsResponse.java

Crea el DTO para la respuesta en `model/dto/`:

```java
package dev.alefiengo.analyticsservice.model.dto;

public class CustomerStatsResponse {
    private String customerName;
    private Long ordersCount;

    public CustomerStatsResponse() {}

    public CustomerStatsResponse(String customerName, Long ordersCount) {
        this.customerName = customerName;
        this.ordersCount = ordersCount;
    }

    // Getters y Setters
    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }

    public Long getOrdersCount() {
        return ordersCount;
    }

    public void setOrdersCount(Long ordersCount) {
        this.ordersCount = ordersCount;
    }
}
```

### 2. Crear CustomerAnalyticsStream.java

Crea el stream en `streams/`:

```java
package dev.alefiengo.analyticsservice.streams;

import dev.alefiengo.analyticsservice.model.event.OrderConfirmedEvent;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.support.serializer.JsonSerde;

import java.util.Map;

@Configuration
public class CustomerAnalyticsStream {

    @Bean
    public KStream<String, OrderConfirmedEvent> customerOrdersStream(StreamsBuilder builder) {
        // TODO: Implementar
        // Pistas:
        // 1. Leer stream desde "ecommerce.orders.confirmed"
        // 2. Re-key por customerName usando selectKey()
        // 3. Agrupar por key con groupByKey()
        // 4. Contar con count(Materialized.as("customer-orders-count-store"))

        return null; // Reemplazar con implementación
    }

    private static <T> JsonSerde<T> jsonSerde(Class<T> targetClass) {
        JsonSerde<T> serde = new JsonSerde<>(targetClass);
        serde.configure(Map.of(
            "spring.json.trusted.packages", "dev.alefiengo.*"
        ), false);
        return serde;
    }
}
```

### 3. Agregar método en AnalyticsQueryService.java

```java
public List<CustomerStatsResponse> getTopCustomers(int limit) {
    // TODO: Implementar
    // Pistas:
    // 1. Obtener state store "customer-orders-count-store"
    // 2. Iterar todos los valores con store.all()
    // 3. Crear lista de CustomerStatsResponse
    // 4. Ordenar por ordersCount descendente
    // 5. Retornar solo los primeros 'limit' elementos

    return null; // Reemplazar con implementación
}
```

### 4. Agregar endpoint en AnalyticsController.java

```java
@GetMapping("/customers/top")
public ResponseEntity<List<CustomerStatsResponse>> getTopCustomers(
    @RequestParam(defaultValue = "5") int limit
) {
    return ResponseEntity.ok(queryService.getTopCustomers(limit));
}
```

---

## Pistas de implementación

### Paso 1: Re-keying

Necesitas cambiar la key del mensaje de `orderId` a `customerName`:

```java
stream.selectKey((orderId, event) -> event.getCustomerName())
```

### Paso 2: Contar por customerName

```java
.groupByKey()
.count(Materialized.as("customer-orders-count-store"))
```

### Paso 3: Iterar state store

```java
ReadOnlyKeyValueStore<String, Long> store =
    kafkaStreams.store(
        StoreQueryParameters.fromNameAndType(
            "customer-orders-count-store",
            QueryableStoreTypes.keyValueStore()
        )
    );

List<CustomerStatsResponse> customers = new ArrayList<>();

try (KeyValueIterator<String, Long> iterator = store.all()) {
    while (iterator.hasNext()) {
        KeyValue<String, Long> entry = iterator.next();
        customers.add(new CustomerStatsResponse(
            entry.key,
            entry.value
        ));
    }
}
```

### Paso 4: Ordenar y limitar

```java
customers.sort((a, b) -> Long.compare(b.getOrdersCount(), a.getOrdersCount()));

return customers.stream()
    .limit(limit)
    .collect(Collectors.toList());
```

---

## Validación

### 1. Crear órdenes con diferentes clientes

```bash
# Cliente 1: 3 órdenes
for i in {1..3}; do
  curl -X POST http://localhost:8081/api/orders \
    -H "Content-Type: application/json" \
    -d '{
      "productId": 1,
      "quantity": 5,
      "customerName": "Juan Perez",
      "customerEmail": "juan@example.com",
      "totalAmount": 7500.00
    }'
  sleep 1
done

# Cliente 2: 2 órdenes
for i in {1..2}; do
  curl -X POST http://localhost:8081/api/orders \
    -H "Content-Type: application/json" \
    -d '{
      "productId": 2,
      "quantity": 3,
      "customerName": "Maria Lopez",
      "customerEmail": "maria@example.com",
      "totalAmount": 299.97
    }'
  sleep 1
done

# Cliente 3: 1 orden
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 10,
    "customerName": "Pedro Gomez",
    "customerEmail": "pedro@example.com",
    "totalAmount": 15000.00
  }'
```

### 2. Consultar top clientes

```bash
# Esperar propagación
sleep 3

# Consultar endpoint
curl http://localhost:8083/api/analytics/customers/top
```

### 3. Resultado esperado

```json
[
  {
    "customerName": "Juan Perez",
    "ordersCount": 3
  },
  {
    "customerName": "Maria Lopez",
    "ordersCount": 2
  },
  {
    "customerName": "Pedro Gomez",
    "ordersCount": 1
  }
]
```

---

## Criterios de evaluación

### Funcionalidad (70 puntos)

- [ ] CustomerStatsResponse creado correctamente (10 pts)
- [ ] CustomerAnalyticsStream implementado (20 pts)
  - Re-keying con selectKey()
  - groupByKey() + count()
  - State store correctamente nombrado
- [ ] AnalyticsQueryService.getTopCustomers() implementado (20 pts)
  - Consulta state store correctamente
  - Itera todos los valores
  - Ordena por cantidad descendente
  - Limita resultados correctamente
- [ ] Endpoint /api/analytics/customers/top funciona (20 pts)

### Código y Buenas Prácticas (20 puntos)

- [ ] JsonSerde configurado con trusted.packages (5 pts)
- [ ] Iterator cerrado correctamente (try-with-resources o try-finally) (5 pts)
- [ ] Código limpio y bien organizado (5 pts)
- [ ] Manejo correcto de nulls (5 pts)

### Documentación (10 puntos)

- [ ] Screenshot de Postman mostrando resultado (5 pts)
- [ ] Breve explicación del enfoque (5 pts)

---

## Entrega

### Formato

Crear un documento (PDF, Markdown, o Word) con:

1. **Código implementado**:
   - CustomerStatsResponse.java
   - CustomerAnalyticsStream.java
   - Método getTopCustomers() en AnalyticsQueryService
   - Endpoint en AnalyticsController

2. **Screenshot de Postman**:
   - Request a `GET http://localhost:8083/api/analytics/customers/top`
   - Response mostrando al menos 3 clientes ordenados

3. **Breve explicación** (máximo 200 palabras):
   - ¿Por qué usaste selectKey()?
   - ¿Cómo funciona el state store "customer-orders-count-store"?
   - ¿Qué pasaría si reiniciaras analytics-service?

### Dónde entregar

- **Repositorio personal**: Subir código a tu repositorio en la rama `clase7-tarea`
- **Moodle**: Subir documento PDF con código + screenshots + explicación
- **Fecha límite**: [A definir por instructor]

---

## Desafío adicional (Opcional - Extra 10 puntos)

Implementa un segundo endpoint que permita **filtrar clientes** por un threshold mínimo de órdenes:

```
GET /api/analytics/customers/active?minOrders=5
```

Debe retornar solo clientes con 5 o más órdenes confirmadas.

**Pista**: Filtrar la lista antes de retornar.

---

## Troubleshooting

### State store no existe

**Error**: `InvalidStateStoreException: Cannot get state store customer-orders-count-store`

**Solución**:
1. Verificar nombre coincide en `Materialized.as()` y `store.get()`
2. Verificar Kafka Streams está en estado RUNNING
3. Verificar logs de analytics-service

### Contador siempre en 0

**Causa**: No se están procesando eventos.

**Solución**:
1. Verificar topic `ecommerce.orders.confirmed` tiene eventos:
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning
```
2. Verificar logs de analytics-service

### NullPointerException

**Causa**: Key no existe en state store.

**Solución**: Validar null antes de usar:
```java
Long count = store.get(customerName);
return count != null ? count : 0L;
```

---

## Recursos

- [Lab 02: Stream básico - Contar órdenes](../labs/02-stream-contar-ordenes/)
- [Lab 03: Stream - Top productos](../labs/03-stream-top-productos/)
- [Cheatsheet Clase 7](../cheatsheet.md)
- [FAQ Clase 7](../FAQ.md)

---

**¡Éxito con la tarea!** Si tienes dudas, consulta el FAQ o revisa los labs de la clase.
