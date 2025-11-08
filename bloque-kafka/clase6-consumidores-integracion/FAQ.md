# FAQ - Clase 6: Consumidores e Integración

Preguntas frecuentes y soluciones a problemas comunes al trabajar con consumidores Kafka y múltiples microservicios.

---

## Configuración de Consumers

### ¿Por qué mi consumer no recibe mensajes?

**Síntoma**: El consumer inicia sin errores pero no procesa ningún mensaje.

**Causas posibles**:

1. **El topic no existe**:
```bash
# Verificar que el topic existe
kafka-topics --bootstrap-server localhost:9092 --list | grep orders.placed
```

2. **auto-offset-reset está en `latest`** y no hay mensajes nuevos:
```yaml
# Cambiar a earliest para leer mensajes históricos
spring:
  kafka:
    consumer:
      auto-offset-reset: earliest
```

3. **group-id ya tiene offsets committidos**:
```bash
# Ver offsets del grupo
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe

# Resetear offsets si es necesario
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --reset-offsets \
  --to-earliest \
  --topic ecommerce.orders.placed \
  --execute
```

4. **Kafka no está corriendo**:
```bash
docker compose ps | grep kafka
# Si no está "Up", ejecutar:
cd ~/workspace/kafka-infrastructure
docker compose up -d
```

---

### ¿Cómo cambio el consumer group de un listener?

**Problema**: Quiero que mi consumer lea mensajes desde el principio con un grupo nuevo.

**Solución**:

```java
// Antes
@KafkaListener(
    topics = "ecommerce.orders.placed",
    groupId = "inventory-service"  // Grupo viejo
)

// Después
@KafkaListener(
    topics = "ecommerce.orders.placed",
    groupId = "inventory-service-v2"  // Grupo nuevo
)
```

Cambiar el `groupId` crea un nuevo consumer group sin offsets previos, leyendo desde `auto-offset-reset`.

---

### ¿Puedo tener múltiples consumers del mismo grupo?

**Sí**, pero solo UNO recibirá cada mensaje (load balancing).

**Ejemplo**:

```bash
# Terminal 1: inventory-service instancia 1
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8082

# Terminal 2: inventory-service instancia 2
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8083
```

Ambas instancias pertenecen al grupo `inventory-service`. Kafka distribuye particiones entre ellas:

```
Partition 0 → Instancia 1
Partition 1 → Instancia 2
Partition 2 → Instancia 1
```

---

## Deserialización

### Error: "Could not resolve type id"

**Síntoma completo**:
```
Could not resolve type id 'dev.alefiengo.orderservice.kafka.event.OrderPlacedEvent'
into a subtype of [simple type, class java.lang.Object]
```

**Causa**: Falta configuración de `spring.json.type.mapping`.

**Solución**:

```yaml
spring:
  kafka:
    consumer:
      properties:
        spring.json.type.mapping: orderPlacedEvent:dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent
```

**Nota**: El paquete debe coincidir con TU servicio, no con el productor.

---

### Error: "Package not in trusted packages"

**Síntoma completo**:
```
The class 'dev.alefiengo.orderservice.kafka.event.OrderPlacedEvent' is not in the trusted packages
```

**Causa**: `spring.json.trusted.packages` no incluye el paquete del evento.

**Solución**:

```yaml
spring:
  kafka:
    consumer:
      properties:
        spring.json.trusted.packages: dev.alefiengo.*
```

El wildcard `.*` confía en todos los subpaquetes de `dev.alefiengo`.

---

### Error: "No default constructor"

**Síntoma**: Consumer no puede deserializar el evento.

**Causa**: La clase del evento no tiene constructor sin argumentos.

**Solución**:

```java
public class OrderPlacedEvent {

    // ✅ REQUERIDO: Constructor vacío
    public OrderPlacedEvent() {
    }

    // Constructor con argumentos (opcional)
    public OrderPlacedEvent(Long orderId, Long productId, Integer quantity) {
        this.orderId = orderId;
        this.productId = productId;
        this.quantity = quantity;
    }

    // Getters y setters (REQUERIDOS)
}
```

---

## Problemas con Múltiples Microservicios

### ¿Por qué tengo que duplicar las clases de eventos?

**Problema**: `OrderPlacedEvent` existe tanto en order-service como en inventory-service.

**Razón**: Microservicios NO comparten código. Cada uno tiene su propia copia de los DTOs/eventos que consume.

**Alternativa** (NO recomendada para este curso):
- Crear un módulo compartido `common-events.jar`
- Cada microservicio importa este módulo
- **Desventaja**: Acoplamiento entre servicios

**Recomendación**: Duplicar las clases de eventos para mantener autonomía de microservicios.

---

### Los eventos no llegan en orden

**Síntoma**: Consumer recibe `OrderPlacedEvent(id=2)` antes que `OrderPlacedEvent(id=1)`.

**Causa**: Mensajes fueron publicados a diferentes particiones.

**Kafka garantiza orden SOLO dentro de una partición**.

**Solución**:

Si necesitas orden garantizado, usa la misma key para mensajes relacionados:

```java
// Productor usa orderId como key
String key = event.getOrderId().toString();
kafkaTemplate.send(TOPIC, key, event);

// Todos los mensajes con la misma key van a la misma partición
// → Orden garantizado
```

---

### order-service actualiza la orden dos veces

**Síntoma**: Logs muestran "Order confirmed" dos veces para la misma orden.

**Causa**: Kafka entregó el mensaje dos veces (at-least-once delivery).

**Explicación**: Si el consumer procesa el mensaje pero crashea ANTES de commitear el offset, Kafka lo reenvía.

**Solución** (idempotencia):

```java
@Transactional
public void confirmOrder(Long orderId) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));

    // ✅ IDEMPOTENTE: Solo actualiza si está PENDING
    if (order.getStatus() == OrderStatus.PENDING) {
        order.setStatus(OrderStatus.CONFIRMED);
        orderRepository.save(order);
        log.info("Order confirmed: orderId={}", orderId);
    } else {
        log.warn("Order already processed: orderId={}, status={}",
                 orderId, order.getStatus());
    }
}
```

---

## Bases de Datos

### Error: "database ecommerce_inventory does not exist"

**Síntoma**: inventory-service no puede iniciar.

**Causa**: La base de datos no fue creada.

**Solución**:

```bash
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_inventory;"
```

---

### ¿Cómo veo el estado actual de las órdenes?

```bash
docker exec -it postgres psql -U postgres -d ecommerce_orders \
  -c "SELECT id, product_id, quantity, status, created_at FROM customer_orders ORDER BY id DESC LIMIT 10;"
```

**Output esperado**:
```
 id | product_id | quantity |   status   |       created_at
----+------------+----------+------------+---------------------
  3 |          1 |        2 | CONFIRMED  | 2025-11-08 10:30:15
  2 |          1 |       50 | CANCELLED  | 2025-11-08 10:25:30
  1 |          1 |        5 | PENDING    | 2025-11-08 10:20:00
```

---

### ¿Cómo veo el stock actual de productos?

```bash
docker exec -it postgres psql -U postgres -d ecommerce_inventory \
  -c "SELECT product_id, product_name, available_stock, reserved_stock FROM inventory_items;"
```

**Output esperado**:
```
 product_id |   product_name    | available_stock | reserved_stock
------------+-------------------+-----------------+----------------
          1 | Laptop Dell XPS   |              45 |              5
          2 | Mouse Logitech    |             100 |              0
```

---

## Consumer Groups y Offsets

### ¿Qué significa el "lag" en un consumer group?

**Lag** = Número de mensajes pendientes de procesar.

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe
```

**Output**:
```
TOPIC                    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
ecommerce.orders.placed  0          50              55              5
ecommerce.orders.placed  1          30              30              0
ecommerce.orders.placed  2          40              45              5
```

**Interpretación**:
- **Partition 0**: Hay 5 mensajes pendientes (lag=5)
- **Partition 1**: Está al día (lag=0)
- **Partition 2**: Hay 5 mensajes pendientes

**Alto lag** puede indicar:
- Consumer lento
- Consumer caído
- Pico de tráfico

---

### ¿Cómo reseteo offsets para reprocesar mensajes?

**CUIDADO**: Esto hará que el consumer reprocese mensajes.

```bash
# 1. Detener el consumer
# Ctrl+C en la terminal donde corre

# 2. Resetear offsets a earliest
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --reset-offsets \
  --to-earliest \
  --topic ecommerce.orders.placed \
  --execute

# 3. Reiniciar el consumer
mvn spring-boot:run
```

**Opciones de reseteo**:
- `--to-earliest`: Desde el primer mensaje disponible
- `--to-latest`: Desde el último mensaje (skip pendientes)
- `--to-offset 10`: A un offset específico
- `--shift-by -5`: Retroceder 5 mensajes
- `--to-datetime 2025-11-08T10:00:00.000`: A un timestamp

---

## Arquitectura y Flujo

### ¿Por qué usar eventos en lugar de REST?

**Ventajas de eventos (Kafka)**:
- Desacoplamiento: Servicios no necesitan conocerse
- Asíncrono: No bloquea el productor
- Resiliencia: Mensajes persisten si consumer está caído
- Escalabilidad: Agregar consumers es fácil
- Auditoría: Log permanente de eventos

**Desventajas**:
- Complejidad: Más difícil de debuggear
- Eventual consistency: Datos no instantáneos
- Duplicación: Eventos duplicados entre servicios

**Cuándo usar REST**:
- Operaciones síncronas (login, consultas)
- Respuestas inmediatas requeridas
- Operaciones simples sin orquestación

**Cuándo usar Kafka**:
- Notificaciones
- Procesos largos
- Comunicación entre microservicios
- Event sourcing

---

### ¿Qué pasa si inventory-service está caído cuando se crea una orden?

**Flujo**:

1. Cliente crea orden → order-service responde 201 Created
2. order-service publica `OrderPlacedEvent` → Kafka
3. Kafka persiste el mensaje en disco
4. inventory-service está caído → mensaje queda en Kafka
5. Se levanta inventory-service → lee mensajes pendientes
6. inventory-service procesa la orden (aunque fue creada horas antes)

**Ventaja**: La orden NO se pierde. El sistema es resiliente a fallos temporales.

---

### ¿Cómo saber si un evento fue procesado correctamente?

**Opción 1**: Verificar estado en base de datos

```bash
# Ver si la orden cambió de PENDING a CONFIRMED
curl http://localhost:8081/api/orders/1
```

**Opción 2**: Revisar logs del consumer

```bash
# En la terminal donde corre inventory-service
# Buscar línea similar a:
INFO OrderEventConsumer - Received OrderPlacedEvent: orderId=1
INFO InventoryService - Order confirmed: orderId=1
```

**Opción 3**: Consumer group offsets

```bash
# Si el offset avanzó, el mensaje fue procesado
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe
```

---

## Errores Comunes de Compilación

### Error: "Cannot resolve symbol OrderPlacedEvent"

**Causa**: Falta crear la clase `OrderPlacedEvent` en el servicio.

**Solución**: Copiar la clase desde order-service a inventory-service:

```bash
# Estructura esperada:
inventory-service/src/main/java/dev/alefiengo/inventoryservice/kafka/event/OrderPlacedEvent.java
```

**Recuerda**: Cambiar el package a `dev.alefiengo.inventoryservice.kafka.event`.

---

### Error: "Bean KafkaTemplate could not be found"

**Causa**: Falta dependencia spring-kafka en pom.xml.

**Solución**:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

Luego ejecutar:
```bash
mvn clean install
```

---

## Performance

### Mi consumer es muy lento procesando mensajes

**Optimizaciones**:

1. **Aumentar número de particiones** (requiere recrear topic):
```bash
kafka-topics --bootstrap-server localhost:9092 \
  --alter \
  --topic ecommerce.orders.placed \
  --partitions 6
```

2. **Aumentar número de consumers** (mismogroup):
```bash
# Levantar múltiples instancias del servicio
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8082
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8083
```

3. **Procesar en batch** (avanzado - fuera de alcance):
```java
@KafkaListener(topics = "my-topic", containerFactory = "batchFactory")
public void consumeBatch(List<OrderPlacedEvent> events) {
    // Procesar todos juntos
}
```

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Retraso | Lag | Mensajes pendientes de procesar |
| Confirmar | Commit | Guardar progreso del consumer |
| Desplazamiento actual | Current offset | Último mensaje procesado |
| Desplazamiento final | Log-end offset | Último mensaje disponible |
| Reprocesar | Reprocess | Volver a consumir mensajes |
| Idempotencia | Idempotency | Operación segura de repetir |
| Consistencia eventual | Eventual consistency | Datos consistentes después de un tiempo |
| Duplicado | Duplicate | Mensaje procesado más de una vez |