# FAQ - Clase 7: Kafka Streams

Preguntas frecuentes y troubleshooting para Kafka Streams y analytics-service.

---

## Tabla de contenidos

1. [Conceptos Generales](#conceptos-generales)
2. [Errores de Configuración](#errores-de-configuración)
3. [Errores de State Stores](#errores-de-state-stores)
4. [Errores de Serialización](#errores-de-serialización)
5. [Performance y Escalabilidad](#performance-y-escalabilidad)
6. [Debugging y Monitoreo](#debugging-y-monitoreo)

---

## Conceptos Generales

### ¿Cuándo usar @KafkaListener vs Kafka Streams?

**Usa @KafkaListener cuando**:
- Necesitas actualizar base de datos (CRUD)
- Lógica de negocio transaccional
- Procesamiento evento por evento
- Enviar notificaciones
- Ejemplo: order-service, inventory-service

**Usa Kafka Streams cuando**:
- Analytics en tiempo real
- Agregaciones (count, sum, avg)
- Transformaciones complejas
- Top N elementos
- Ejemplo: analytics-service

**Regla práctica**: Si necesitas mantener contadores o agregados → Kafka Streams. Si necesitas escribir a BD → @KafkaListener.

---

### ¿Cuál es la diferencia entre KStream y KTable?

**KStream** - Todos los eventos se procesan:
```
orderId=1, quantity=5  → Procesa
orderId=2, quantity=3  → Procesa
orderId=3, quantity=10 → Procesa
Resultado: 3 eventos procesados
```

**KTable** - Solo último valor por key:
```
productId=1, stock=50 → Almacena
productId=1, stock=45 → Reemplaza anterior
productId=1, stock=42 → Reemplaza anterior
Resultado: productId=1 → stock=42
```

**Cuándo usar cada uno**:
- KStream: Órdenes, transacciones, logs (todos los eventos importan)
- KTable: Inventario, perfiles, configuraciones (solo estado actual importa)

---

### ¿Los state stores comparten datos entre instancias?

**NO**. Cada instancia de analytics-service tiene su **propia copia local** del state store.

**Sin embargo**:
- State stores están **respaldados en Kafka** (changelog topics)
- Si una instancia falla, otra puede **reconstruir** el estado desde Kafka
- Son **consistentes** (eventual consistency)

**En este curso**: Solo una instancia de analytics-service, por lo que esto no es problema.

---

### ¿Qué pasa si reinicio analytics-service?

**Al reiniciar**:
1. Kafka Streams lee **changelog topics**
2. **Reconstruye** state stores desde Kafka
3. Puede tomar **varios segundos** dependiendo del volumen de datos
4. Una vez reconstruido, los contadores estarán **correctos**

**Logs esperados**:
```
Restoring state from changelog topic...
State restoration completed
State transition to RUNNING
```

---

### ¿Por qué hay retraso entre crear orden y ver en analytics?

**Esto es normal** - Se llama **eventual consistency**:

```
1. Usuario crea orden → PostgreSQL (INMEDIATO)
2. Evento publicado a Kafka → (milisegundos)
3. analytics-service procesa → (milisegundos)
4. Usuario consulta /api/analytics → (READ MODEL)

Total: 1-2 segundos de retraso
```

**Esto NO es un bug** - Es característica de arquitecturas event-driven.

**Solución**: Si necesitas datos 100% actualizados, consulta el write model (order-service directamente).

---

## Errores de Configuración

### Error: `StreamsConfig values: application.id = [null]`

**Causa**: `application-id` no configurado en `application.yml`.

**Solución**: Agregar en `application.yml`:

```yaml
spring:
  kafka:
    streams:
      application-id: analytics-service  # OBLIGATORIO
```

**Importante**: `application-id` debe ser **único** para cada aplicación Kafka Streams.

---

### Error: `Failed to construct kafka stream`

**Causa**: Kafka no está disponible.

**Solución**:

```bash
# Verificar Kafka está corriendo
docker compose ps

# Si no está corriendo
docker compose up -d

# Verificar logs
docker compose logs kafka
```

---

### Error: `Port 8083 already in use`

**Causa**: Otra aplicación usa puerto 8083.

**Solución 1**: Matar proceso existente:

```bash
# Linux/Mac
lsof -ti:8083 | xargs kill -9

# Windows
netstat -ano | findstr :8083
taskkill /PID <PID> /F
```

**Solución 2**: Cambiar puerto en `application.yml`:

```yaml
server:
  port: 8084
```

---

## Errores de State Stores

### Error: `InvalidStateStoreException: Cannot get state store`

**Causa**: Nombre de state store incorrecto o Streams no está en RUNNING.

**Solución 1**: Verificar nombre coincide:

```java
// En stream
.count(Materialized.as("confirmed-count-store"))

// En query
kafkaStreams.store(
    StoreQueryParameters.fromNameAndType(
        "confirmed-count-store",  // ← MISMO NOMBRE
        QueryableStoreTypes.keyValueStore()
    )
);
```

**Solución 2**: Verificar estado de Streams:

```java
if (kafkaStreams.state() != State.RUNNING) {
    throw new IllegalStateException("Kafka Streams not running");
}
```

---

### Error: `NullPointerException` al consultar store.get()

**Causa**: Key no existe en state store (aún no ha llegado ningún evento).

**Solución**: SIEMPRE validar null:

```java
Long count = store.get("confirmed");
return count != null ? count : 0L;  // ← Retornar 0 si no existe
```

**Esto es normal** cuando:
- analytics-service recién inició
- No se han creado órdenes todavía
- State store fue borrado

---

### State store vacío después de reiniciar

**Síntoma**: Todos los contadores en 0 después de reiniciar analytics-service.

**Causa**: State stores se están **reconstruyendo** desde Kafka changelog topics.

**Solución**: **Esperar 10-30 segundos** para que Kafka Streams reconstruya estado.

**Ver progreso en logs**:

```bash
grep "Restoring state" logs/analytics-service.log
grep "State transition" logs/analytics-service.log
```

**Si después de 2 minutos sigue vacío**: Ver sección "State stores corruptos" abajo.

---

### State stores corruptos

**Síntoma**: State stores nunca se reconstruyen, errores de RocksDB en logs.

**Solución**: Limpiar state stores y reiniciar:

```bash
# 1. Detener analytics-service
Ctrl+C

# 2. Borrar state stores locales
rm -rf /tmp/kafka-streams/*

# 3. Reiniciar analytics-service
mvn spring-boot:run

# 4. Esperar reconstrucción (puede tomar minutos)
```

**Logs esperados**:

```
Restoring state from changelog topic...
State restoration completed (XXX records restored)
State transition to RUNNING
```

---

## Errores de Serialización

### Error: `DeserializationException: class X is not in the trusted packages`

**Causa**: `trusted.packages` no configurado en JsonSerde.

**Solución**: Configurar JsonSerde correctamente:

```java
private static <T> JsonSerde<T> jsonSerde(Class<T> targetClass) {
    JsonSerde<T> serde = new JsonSerde<>(targetClass);
    serde.configure(Map.of(
        "spring.json.trusted.packages", "dev.alefiengo.*"  // ← OBLIGATORIO
    ), false);
    return serde;
}
```

**SIEMPRE incluir** en todos los métodos que usan JsonSerde.

---

### Error: `No suitable constructor found for type X`

**Causa**: Clase de evento no tiene constructor vacío.

**Solución**: Agregar constructor vacío:

```java
public class OrderConfirmedEvent {
    private Long orderId;
    private Long productId;

    // Constructor vacío OBLIGATORIO
    public OrderConfirmedEvent() {}

    // Constructor completo
    public OrderConfirmedEvent(Long orderId, Long productId) {
        this.orderId = orderId;
        this.productId = productId;
    }

    // Getters y setters
}
```

---

### Error: `Cannot deserialize instance of X from VALUE_STRING`

**Causa**: Evento en Kafka tiene formato diferente al esperado.

**Solución**: Verificar que eventos en analytics-service **coincidan exactamente** con los de order-service/inventory-service:

1. Mismo package path
2. Mismos campos
3. Mismos tipos

**Verificar evento en Kafka**:

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --max-messages 1
```

---

## Performance y Escalabilidad

### ¿Cuántos eventos por segundo puede procesar Kafka Streams?

**En este curso (una instancia)**:
- Eventos simples: 10,000-50,000 eventos/seg
- Agregaciones complejas: 1,000-10,000 eventos/seg

**En producción (múltiples instancias)**:
- Millones de eventos/seg

**Factores que afectan performance**:
- Complejidad de operaciones (map simple vs reduce complejo)
- Tamaño de eventos (KB vs MB)
- Número de particiones
- Hardware (CPU, memoria, disco)

---

### ¿Cómo escalar analytics-service?

**Horizontal scaling** (múltiples instancias):

```bash
# Instancia 1
mvn spring-boot:run -Dserver.port=8083

# Instancia 2 (en otro terminal)
mvn spring-boot:run -Dserver.port=8084

# Kafka Streams distribuye automáticamente particiones entre instancias
```

**IMPORTANTE**: Número de instancias <= número de particiones del topic.

**En este curso**: NO hacemos scaling (solo una instancia).

---

### ¿State stores consumen mucha memoria?

**Depende del volumen de datos**:

**En este curso (100-1000 órdenes)**:
- State stores: < 10 MB
- Memoria total: 512 MB suficiente

**En producción (millones de eventos)**:
- State stores: 100 MB - 10 GB
- Usar RocksDB (persistente en disco, menos memoria)

**Configurar límite de memoria**:

```yaml
spring:
  kafka:
    streams:
      properties:
        cache.max.bytes.buffering: 10485760  # 10 MB
```

---

## Debugging y Monitoreo

### ¿Cómo saber si Kafka Streams está procesando eventos?

**Opción 1: Logs**

```bash
# Buscar "Processing" en logs
grep "Processing" logs/analytics-service.log

# Ver últimas 50 líneas
tail -50 logs/analytics-service.log
```

**Opción 2**: Agregar logging en stream:

```java
stream.peek((key, value) ->
    log.info("Processing OrderConfirmedEvent: orderId={}", value.getOrderId())
);
```

**Opción 3**: Consultar state store y verificar que cuenta aumenta.

---

### ¿Cómo ver estado de Kafka Streams?

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

    private final StreamsBuilderFactoryBean factoryBean;

    @GetMapping("/streams/state")
    public ResponseEntity<String> getStreamsState() {
        KafkaStreams kafkaStreams = factoryBean.getKafkaStreams();
        return ResponseEntity.ok(kafkaStreams.state().toString());
    }
}
```

**Estados posibles**:
- `CREATED`: Streams creado pero no iniciado
- `REBALANCING`: Rebalanceando particiones
- `RUNNING`: Procesando eventos normalmente
- `PENDING_SHUTDOWN`: Deteniendo
- `NOT_RUNNING`: Detenido
- `ERROR`: Error fatal

**Estado esperado**: `RUNNING`

---

### ¿Cómo depurar topología de streams?

```java
@Bean
public KafkaStreams kafkaStreams(StreamsBuilder builder) {
    Topology topology = builder.build();

    // Imprimir topología
    System.out.println(topology.describe());

    return new KafkaStreams(topology, streamsConfig);
}
```

**Output ejemplo**:

```
Topologies:
   Sub-topology: 0
    Source: KSTREAM-SOURCE-0000000000 (topics: [ecommerce.orders.confirmed])
      --> KSTREAM-GROUPBY-0000000001
    Processor: KSTREAM-GROUPBY-0000000001 (stores: [])
      --> KSTREAM-AGGREGATE-0000000002
      <-- KSTREAM-SOURCE-0000000000
    Processor: KSTREAM-AGGREGATE-0000000002 (stores: [confirmed-count-store])
      --> none
      <-- KSTREAM-GROUPBY-0000000001
```

---

### ¿Cómo verificar consumer group offsets?

```bash
# Ver consumer group de Kafka Streams
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group analytics-service \
  --describe

# Output esperado:
# TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# ecommerce.orders.confirmed     0          150             150             0
# ecommerce.orders.confirmed     1          142             142             0
# ecommerce.orders.confirmed     2          138             138             0
```

**LAG = 0**: Sin retraso, procesando en tiempo real
**LAG > 0**: analytics-service atrasado, procesar más rápido

---

### ¿Cómo resetear consumer group offsets?

**WARNING**: Esto reprocesará TODOS los eventos desde el inicio.

```bash
# 1. Detener analytics-service
Ctrl+C

# 2. Resetear offsets
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group analytics-service \
  --reset-offsets \
  --to-earliest \
  --all-topics \
  --execute

# 3. Reiniciar analytics-service
mvn spring-boot:run
```

**Cuándo usar**: Nunca en producción. Solo en desarrollo si state stores están corruptos.

---

## Errores Comunes de Código

### Error: Stream operations no tienen efecto

**Código incorrecto**:

```java
KStream<String, OrderEvent> stream = builder.stream("orders");
stream.filter(...);  // ❌ No hace nada
stream.map(...);     // ❌ No hace nada
```

**Causa**: KStream es **inmutable**. Operaciones retornan **nuevo stream**.

**Código correcto**:

```java
KStream<String, OrderEvent> stream = builder.stream("orders");
KStream<String, OrderEvent> filtered = stream.filter(...);
KStream<String, Integer> mapped = filtered.map(...);
```

O encadenar:

```java
stream
    .filter(...)
    .map(...)
    .groupByKey()
    .count(Materialized.as("count-store"));
```

---

### Error: State store duplicado

**Código incorrecto**:

```java
stream1.count(Materialized.as("count-store"));
stream2.count(Materialized.as("count-store"));  // ❌ Nombre duplicado
```

**Causa**: Dos streams usan mismo nombre de state store.

**Código correcto**:

```java
stream1.count(Materialized.as("confirmed-count-store"));
stream2.count(Materialized.as("cancelled-count-store"));
```

**Nombres de state stores DEBEN ser únicos**.

---

### Error: groupBy sin re-key previo

**Código incorrecto**:

```java
stream
    .groupBy((orderId, event) -> event.getProductId().toString())  // ❌ Menos eficiente
    .count(...);
```

**Código correcto**:

```java
stream
    .selectKey((orderId, event) -> event.getProductId().toString())  // ✅ Re-key explícito
    .groupByKey()
    .count(...);
```

**Razón**: `selectKey()` + `groupByKey()` es más eficiente que `groupBy()` solo.

---

## Recursos Adicionales

### Documentación Oficial

- [Kafka Streams Developer Guide](https://kafka.apache.org/documentation/streams/developer-guide/)
- [Spring Kafka Streams](https://docs.spring.io/spring-kafka/reference/html/#kafka-streams)
- [Kafka Streams FAQ](https://kafka.apache.org/documentation/streams/faq)

### Troubleshooting Avanzado

- [Kafka Streams Debugging Guide](https://kafka.apache.org/35/documentation/streams/developer-guide/debugging.html)
- [State Store Recovery](https://kafka.apache.org/35/documentation/streams/architecture.html#streams_architecture_state)

### Comunidad

- [Stack Overflow - kafka-streams](https://stackoverflow.com/questions/tagged/kafka-streams)
- [Confluent Community](https://forum.confluent.io/)

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Flujo | Stream | Secuencia infinita de eventos |
| Tabla | Table | Vista materializada (último valor por key) |
| Almacén de estado | State Store | Almacenamiento local fault-tolerant |
| Topología | Topology | Grafo de operaciones de procesamiento |
| Re-keying | Re-keying | Cambiar key del mensaje con selectKey() |
| Consistencia eventual | Eventual Consistency | Sincronización con retraso temporal |
| Consultas interactivas | Interactive Queries | Consultar state stores via REST API |
| Rebalanceo | Rebalancing | Redistribuir particiones entre consumidores |

---

**¿No encuentras tu problema?** Consulta el [Cheatsheet](cheatsheet.md) o revisa los [Labs](labs/).
