# Lab 01: Consumer en inventory-service

Implementar un consumer Kafka en inventory-service usando @KafkaListener para escuchar eventos OrderPlacedEvent del topic ecommerce.orders.placed, validar disponibilidad de stock y reservar inventario cuando llegan nuevas órdenes.

---

## Objetivo

Configurar Spring Kafka consumer en inventory-service, crear la clase OrderPlacedEvent (copia del evento de order-service), implementar OrderEventConsumer con @KafkaListener y desarrollar la lógica de negocio para procesar órdenes validando y reservando stock.

---

## Comandos a ejecutar

### Paso 1: Configurar Kafka consumer en application.yml

```bash
cd ~/workspace/inventory-service/src/main/resources

# Editar application.yml para agregar configuración de consumer
cat >> application.yml << 'EOF'

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
EOF
```

**Output esperado**: Sin salida, archivo modificado.

### Paso 2: Crear estructura de paquetes Kafka

```bash
cd ~/workspace/inventory-service/src/main/java/dev/alefiengo/inventoryservice

mkdir -p kafka/event
mkdir -p kafka/consumer
```

**Output esperado**: Sin salida, directorios creados.

### Paso 3: Crear OrderPlacedEvent (copia del evento de order-service)

```bash
cat > kafka/event/OrderPlacedEvent.java << 'EOF'
package dev.alefiengo.inventoryservice.kafka.event;

import java.time.Instant;

/**
 * Evento recibido cuando se crea una nueva orden en order-service.
 * IMPORTANTE: Esta clase es una COPIA del evento de order-service.
 * En arquitecturas de microservicios, cada servicio tiene su propia copia
 * de los eventos que consume (no hay librería compartida).
 */
public class OrderPlacedEvent {

    private Long orderId;
    private Long productId;
    private Integer quantity;
    private String customerName;
    private String customerEmail;
    private Instant timestamp;

    // Constructor vacío (REQUERIDO para deserialización JSON)
    public OrderPlacedEvent() {
    }

    // Constructor completo
    public OrderPlacedEvent(Long orderId, Long productId, Integer quantity,
                            String customerName, String customerEmail, Instant timestamp) {
        this.orderId = orderId;
        this.productId = productId;
        this.quantity = quantity;
        this.customerName = customerName;
        this.customerEmail = customerEmail;
        this.timestamp = timestamp;
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

    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }

    public String getCustomerEmail() {
        return customerEmail;
    }

    public void setCustomerEmail(String customerEmail) {
        this.customerEmail = customerEmail;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }

    @Override
    public String toString() {
        return "OrderPlacedEvent{" +
                "orderId=" + orderId +
                ", productId=" + productId +
                ", quantity=" + quantity +
                ", customerName='" + customerName + '\'' +
                ", customerEmail='" + customerEmail + '\'' +
                ", timestamp=" + timestamp +
                '}';
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 4: Crear OrderEventConsumer

```bash
cat > kafka/consumer/OrderEventConsumer.java << 'EOF'
package dev.alefiengo.inventoryservice.kafka.consumer;

import dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent;
import dev.alefiengo.inventoryservice.service.InventoryService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

/**
 * Consumer de eventos OrderPlacedEvent.
 * Escucha el topic ecommerce.orders.placed y procesa las órdenes
 * validando y reservando stock en el inventario.
 */
@Component
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    private final InventoryService inventoryService;

    // Constructor injection
    public OrderEventConsumer(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    /**
     * Consume eventos OrderPlacedEvent del topic ecommerce.orders.placed
     *
     * @KafkaListener - Marca el método como listener de Kafka
     * topics - Topic(s) a escuchar (puede ser array)
     * groupId - Consumer group al que pertenece este consumer
     */
    @KafkaListener(
        topics = "ecommerce.orders.placed",
        groupId = "inventory-service"
    )
    public void consumeOrderPlaced(OrderPlacedEvent event) {
        log.info("Received OrderPlacedEvent: {}", event);

        try {
            // Delegar la lógica de negocio al Service
            inventoryService.processOrderPlaced(event);

            log.info("Order processed successfully: orderId={}", event.getOrderId());
        } catch (Exception e) {
            log.error("Error processing OrderPlacedEvent: orderId={}, error={}",
                    event.getOrderId(), e.getMessage(), e);
            // En este lab: Solo logging del error
            // Lab 02: Publicaremos OrderCancelledEvent si falla
        }
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 5: Agregar método processOrderPlaced en InventoryService

```bash
cat >> service/InventoryService.java << 'EOF'

    /**
     * Procesa un evento OrderPlacedEvent:
     * 1. Busca el item de inventario por productId
     * 2. Valida que haya stock disponible
     * 3. Reserva el stock (mueve de available a reserved)
     * 4. Guarda en base de datos
     */
    @Transactional
    public void processOrderPlaced(dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent event) {
        log.info("Processing order: orderId={}, productId={}, quantity={}",
                event.getOrderId(), event.getProductId(), event.getQuantity());

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
            throw new RuntimeException(
                    "Insufficient stock. Available: " + item.getAvailableStock() +
                    ", Requested: " + event.getQuantity()
            );
        }

        // 3. Reservar stock (usa método de negocio de la entidad)
        item.reserveStock(event.getQuantity());

        // 4. Persistir cambios
        inventoryRepository.save(item);

        log.info("Stock reserved successfully: orderId={}, productId={}, newAvailableStock={}, newReservedStock={}",
                event.getOrderId(), event.getProductId(), item.getAvailableStock(), item.getReservedStock());

        // Lab 02: Aquí publicaremos OrderConfirmedEvent
    }
EOF
```

**Output esperado**: Sin salida, archivo modificado.

**IMPORTANTE**: Este código se agrega AL FINAL del archivo `InventoryService.java`, antes del cierre de la clase `}`.

### Paso 6: Agregar import necesario en InventoryService

Edita `service/InventoryService.java` y agrega el import al inicio del archivo (después de los otros imports):

```java
import dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent;
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
[INFO] Total time:  5.234 s
[INFO] ------------------------------------------------------------------------
```

### Paso 8: Levantar infraestructura Kafka

**Terminal 1**:

```bash
# Verificar que Kafka está corriendo
cd ~/workspace/kafka-infrastructure
docker compose ps | grep kafka

# Si no está corriendo, levantarlo
docker compose up -d
```

**Output esperado**:
```
kafka  running  0.0.0.0:9092->9092/tcp
```

### Paso 9: Verificar que existe el topic ecommerce.orders.placed

```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list | grep orders
```

**Output esperado**:
```
ecommerce.orders.placed
```

**Si NO existe** (fue creado en Clase 5, Lab 03):
```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.placed \
  --partitions 3 \
  --replication-factor 1
```

### Paso 10: Crear datos de prueba en inventory-service

**Terminal 2**:

```bash
cd ~/workspace/inventory-service

# Ejecutar inventory-service
mvn spring-boot:run
```

**Terminal 3** (crear items de inventario):

```bash
# Crear item para producto 1
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "initialStock": 50
  }'

# Crear item para producto 2
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 2,
    "productName": "Mouse Logitech",
    "initialStock": 100
  }'

# Verificar inventario creado
curl http://localhost:8082/api/inventory
```

**Output esperado**:
```json
[
  {
    "id": 1,
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "availableStock": 50,
    "reservedStock": 0,
    "totalStock": 50,
    "createdAt": "2025-11-08T10:45:00.123Z",
    "updatedAt": "2025-11-08T10:45:00.123Z"
  },
  {
    "id": 2,
    "productId": 2,
    "productName": "Mouse Logitech",
    "availableStock": 100,
    "reservedStock": 0,
    "totalStock": 100,
    "createdAt": "2025-11-08T10:45:05.456Z",
    "updatedAt": "2025-11-08T10:45:05.456Z"
  }
]
```

### Paso 11: Publicar evento de prueba manualmente

**Terminal 4** (producer manual desde Kafka CLI):

```bash
docker exec -it kafka bash

# Producer manual (pega el JSON en una sola línea)
kafka-console-producer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ahora escribe (key:value en una sola línea):
1:{"orderId":1,"productId":1,"quantity":5,"customerName":"Juan Perez","customerEmail":"juan@example.com","timestamp":"2025-11-08T10:50:00Z"}

# Presiona Ctrl+C para salir
```

### Paso 12: Verificar logs de inventory-service

En **Terminal 2** (donde corre inventory-service), deberías ver:

```
2025-11-08T10:50:01.123  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Received OrderPlacedEvent: OrderPlacedEvent{orderId=1, productId=1, quantity=5, customerName='Juan Perez', customerEmail='juan@example.com', timestamp=2025-11-08T10:50:00Z}
2025-11-08T10:50:01.150  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Processing order: orderId=1, productId=1, quantity=5
2025-11-08T10:50:01.180 DEBUG 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Current stock before reservation: availableStock=50, reservedStock=0
2025-11-08T10:50:01.210  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Stock reserved successfully: orderId=1, productId=1, newAvailableStock=45, newReservedStock=5
2025-11-08T10:50:01.234  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Order processed successfully: orderId=1
```

### Paso 13: Verificar stock actualizado

**Terminal 3**:

```bash
# Consultar inventario actualizado
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
  "totalStock": 50,
  "createdAt": "2025-11-08T10:45:00.123Z",
  "updatedAt": "2025-11-08T10:50:01.210Z"
}
```

**Observar**:
- `availableStock`: 50 → 45 (se restaron 5)
- `reservedStock`: 0 → 5 (se agregaron 5)
- `totalStock`: 50 (se mantiene constante)
- `updatedAt`: Cambió al timestamp del procesamiento

### Paso 14: Verificar en PostgreSQL

```bash
docker exec -it postgres psql -U postgres -d ecommerce_inventory \
  -c "SELECT product_id, product_name, available_stock, reserved_stock FROM inventory_items WHERE product_id = 1;"
```

**Output esperado**:
```
 product_id |   product_name      | available_stock | reserved_stock
------------+---------------------+-----------------+----------------
          1 | Laptop Dell XPS 15  |              45 |              5
(1 row)
```

### Paso 15: Probar caso de stock insuficiente

**Terminal 4** (producer manual):

```bash
# Publicar evento que requiere 100 unidades (pero solo hay 45 disponibles)
1:{"orderId":2,"productId":1,"quantity":100,"customerName":"Maria Lopez","customerEmail":"maria@example.com","timestamp":"2025-11-08T10:55:00Z"}
```

**Logs esperados en Terminal 2** (inventory-service):

```
2025-11-08T10:55:01.123  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Received OrderPlacedEvent: OrderPlacedEvent{orderId=2, productId=1, quantity=100...}
2025-11-08T10:55:01.150  INFO 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Processing order: orderId=2, productId=1, quantity=100
2025-11-08T10:55:01.155  WARN 12345 --- [ntainer#0-0-C-1] d.a.i.service.InventoryService           : Insufficient stock for order: orderId=2, productId=1, requested=100, available=45
2025-11-08T10:55:01.160 ERROR 12345 --- [ntainer#0-0-C-1] d.a.i.k.c.OrderEventConsumer             : Error processing OrderPlacedEvent: orderId=2, error=Insufficient stock. Available: 45, Requested: 100
```

**Verificar que NO se modificó el stock**:

```bash
curl http://localhost:8082/api/inventory/product/1
```

**Output esperado** (sin cambios):
```json
{
  "id": 1,
  "productId": 1,
  "availableStock": 45,
  "reservedStock": 5,
  "totalStock": 50
}
```

---

## Desglose del comando

### Configuración consumer en application.yml

| Propiedad | Valor | Descripción |
|-----------|-------|-------------|
| `bootstrap-servers` | `localhost:9092` | Dirección del broker Kafka (con fallback para variable de entorno) |
| `consumer.group-id` | `inventory-service` | Identificador del consumer group (CRÍTICO para offset management) |
| `consumer.auto-offset-reset` | `earliest` | Al iniciar sin offset previo, lee desde el principio del topic |
| `key-deserializer` | `StringDeserializer` | Deserializa keys (orderId) como String |
| `value-deserializer` | `JsonDeserializer` | Deserializa values (eventos) como JSON → objeto Java |
| `spring.json.type.mapping` | `orderPlacedEvent:...` | Mapea tipo JSON a clase Java completa |
| `spring.json.trusted.packages` | `dev.alefiengo.*` | Paquetes confiables para deserialización (seguridad) |

### @KafkaListener

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `topics` | `"ecommerce.orders.placed"` | Topic(s) a escuchar (puede ser array: `{"topic1", "topic2"}`) |
| `groupId` | `"inventory-service"` | Consumer group (puede override el de application.yml) |

**IMPORTANTE**: El `groupId` define:
- Qué mensajes recibe el consumer (load balancing dentro del grupo)
- Dónde se guardan los offsets (progreso del consumer)
- Si cambias el `groupId`, el consumer lee desde el principio (si `auto-offset-reset=earliest`)

### Type Mapping

```yaml
spring.json.type.mapping: orderPlacedEvent:dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent
```

**Formato**: `nombreEvento:paquete.completo.ClaseJava`

**Por qué es necesario**:
1. El producer (order-service) serializa evento con metadata de tipo: `"@type":"orderPlacedEvent"`
2. El consumer (inventory-service) necesita saber qué clase Java crear
3. El mapping dice: "Si recibes @type=orderPlacedEvent, crea instancia de dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent"

**CRÍTICO**: La clase debe estar en el paquete de inventory-service, NO en order-service (microservicios no comparten código).

### Flujo de deserialización

```
JSON recibido:
{
  "@type": "orderPlacedEvent",
  "orderId": 1,
  "productId": 1,
  "quantity": 5,
  ...
}

↓ JsonDeserializer

1. Lee @type = "orderPlacedEvent"
2. Consulta type.mapping → dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent
3. Crea instancia con constructor vacío
4. Llama setters para cada campo
5. Pasa objeto al método del @KafkaListener

↓

consumeOrderPlaced(OrderPlacedEvent event)  ← Recibe objeto Java
```

---

## Explicación detallada

### Consumer Groups y offset management

Cuando inventory-service inicia por primera vez:

```
1. Se registra en Kafka como consumer del grupo "inventory-service"
2. Kafka pregunta: "¿Tienes offset previo para ecommerce.orders.placed?"
3. Respuesta: NO (primera vez)
4. Kafka aplica auto-offset-reset: earliest
5. Consumer lee TODOS los mensajes desde offset 0
6. Por cada mensaje procesado, Kafka guarda el offset
7. Si inventory-service se reinicia, lee desde último offset guardado
```

**Visualización**:

```
Topic ecommerce.orders.placed (Partition 0):
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │  5  │
└─────┴─────┴─────┴─────┴─────┴─────┘
         ↑
         └─ offset actual del grupo "inventory-service"

Si consumer crashea y se reinicia:
- Kafka sabe que procesó hasta offset 1
- Reinicia desde offset 2
- NO reprocesa 0 y 1
```

**Comando para ver offsets**:
```bash
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe
```

**Output**:
```
GROUP             TOPIC                    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
inventory-service ecommerce.orders.placed  0          2               5               3
inventory-service ecommerce.orders.placed  1          1               1               0
inventory-service ecommerce.orders.placed  2          0               0               0
```

**Interpretación**:
- Partition 0: Procesó hasta offset 2, quedan 3 mensajes (lag=3)
- Partition 1: Al día (lag=0)
- Partition 2: Sin mensajes

### Duplicación de eventos entre microservicios

**¿Por qué copiar OrderPlacedEvent?**

```
order-service/
  kafka/event/OrderPlacedEvent.java  ← Producer

inventory-service/
  kafka/event/OrderPlacedEvent.java  ← Consumer (COPIA)
```

**Razones**:

1. **Autonomía de microservicios**: inventory-service NO depende del código de order-service
2. **Versionado independiente**: order-service puede cambiar sin romper inventory-service
3. **Patrón estándar**: En microservicios, NO se comparten librerías de dominio
4. **Flexibilidad**: inventory-service puede tener campos adicionales que no le interesan a order-service

**Alternativa NO recomendada**:
- Crear módulo `common-events.jar` compartido
- Problema: Acoplamiento entre servicios (contradice principio de microservicios)

### Trusted Packages (seguridad)

```yaml
spring.json.trusted.packages: dev.alefiengo.*
```

**Por qué existe**:

JsonDeserializer puede crear CUALQUIER clase Java basándose en el JSON. Esto es un riesgo de seguridad (ataque de deserialización).

**Ejemplo de ataque** (sin trusted packages):
```json
{
  "@type": "java.lang.ProcessBuilder",
  "command": ["rm", "-rf", "/"]
}
```

JsonDeserializer intentaría crear `ProcessBuilder` y ejecutar comando destructivo.

**Solución**: `trusted.packages` limita qué paquetes pueden ser deserializados.

```yaml
dev.alefiengo.*  ← Solo clases de dev.alefiengo y subpaquetes
```

### Constructor vacío (REQUERIDO)

```java
public OrderPlacedEvent() {
}
```

**Por qué es obligatorio**:

Jackson (biblioteca JSON de Spring) necesita:
1. Crear instancia SIN argumentos
2. Luego llamar setters para cada campo

**Flujo de deserialización**:
```java
// Jackson internamente hace:
OrderPlacedEvent event = new OrderPlacedEvent();  // ← Necesita constructor vacío
event.setOrderId(1L);  // ← Necesita setter
event.setProductId(1L);
event.setQuantity(5);
// ...
```

**Si falta constructor vacío**: Error al deserializar:
```
InvalidDefinitionException: Cannot construct instance of OrderPlacedEvent
(no Creators, like default constructor, exist)
```

### Patrón Consumer → Service

```
OrderEventConsumer (Controller layer para Kafka)
    ↓
    Llama a InventoryService.processOrderPlaced()
    ↓
InventoryService (Business logic)
    ↓
    Llama a InventoryRepository.save()
    ↓
InventoryRepository (Data access)
```

**Por qué separar Consumer y Service**:

1. **Responsabilidad única**: Consumer solo maneja Kafka, Service maneja lógica de negocio
2. **Testabilidad**: Puedes testear Service sin Kafka
3. **Reusabilidad**: Mismo método Service puede ser llamado desde REST o Kafka
4. **Transacciones**: @Transactional en Service, no en Consumer

### Gestión de errores en consumer

```java
try {
    inventoryService.processOrderPlaced(event);
} catch (Exception e) {
    log.error("Error processing...");
    // Solo logging en este lab
    // Lab 02: Publicaremos OrderCancelledEvent
}
```

**Qué pasa cuando hay excepción**:

1. Consumer captura excepción
2. Log del error
3. **Kafka commitea el offset** (mensaje se marca como procesado)
4. Kafka NO reenvía el mensaje

**Alternativas** (fuera de alcance de este lab):
- NO capturar excepción → Kafka reintenta (at-least-once delivery)
- Configurar `ErrorHandlingDeserializer` para manejar errores de deserialización
- Implementar Dead Letter Queue (DLQ) para mensajes fallidos

### Auto-commit de offsets

Por defecto, Spring Kafka usa **auto-commit**:

```
1. Consumer recibe mensaje
2. Procesa mensaje (llama a tu método @KafkaListener)
3. Si NO hay excepción, Spring automáticamente commitea offset
4. Kafka guarda: "El grupo inventory-service procesó hasta offset X"
```

**Configuración** (implícita, no necesitas configurarla):
```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: true  # Default
```

**Manual commit** (avanzado, no en este lab):
```java
@KafkaListener(topics = "my-topic")
public void consume(OrderPlacedEvent event, Acknowledgment ack) {
    processEvent(event);
    ack.acknowledge();  // Commit manual explícito
}
```

### Eventual Consistency en acción

**Timeline del flujo**:

```
T0: Cliente crea orden vía REST → order-service
    - order-service: Order(id=1, status=PENDING) guardado en BD
    - Cliente recibe: HTTP 201 Created

T1: order-service publica OrderPlacedEvent → Kafka
    - Kafka persiste mensaje

T2: inventory-service consume OrderPlacedEvent
    - Valida stock
    - Reserva 5 unidades (50 → 45 available, 0 → 5 reserved)

T3: [Lab 02] inventory-service publica OrderConfirmedEvent → Kafka

T4: [Lab 03] order-service consume OrderConfirmedEvent
    - Actualiza Order(id=1, status=CONFIRMED)
```

**Ventana de inconsistencia**: Entre T0 y T4, la orden está PENDING pero el stock ya está reservado (o al revés).

**Esto es NORMAL** en arquitecturas event-driven. Se acepta eventual consistency a cambio de:
- Desacoplamiento
- Escalabilidad
- Resiliencia

---

## Conceptos aprendidos

- **@KafkaListener**: Anotación de Spring Kafka para marcar método como consumer de mensajes
- **Consumer Groups**: Identificador que agrupa consumers para load balancing y offset management
- **auto-offset-reset**: Estrategia para leer mensajes cuando no hay offset previo (earliest vs latest)
- **JsonDeserializer**: Convierte JSON a objetos Java automáticamente
- **Type mapping**: Mapeo entre nombre de evento JSON y clase Java completa
- **Trusted packages**: Seguridad para limitar qué clases pueden ser deserializadas
- **Constructor vacío**: Requerido por Jackson para crear instancias durante deserialización
- **Event duplication**: Cada microservicio tiene copia de eventos que consume (no librería compartida)
- **Offset commit**: Kafka guarda progreso del consumer para no reprocesar mensajes
- **Eventual consistency**: Ventana de tiempo donde datos no están sincronizados entre servicios
- **Consumer → Service pattern**: Separación de responsabilidades (Kafka handling vs business logic)

---

## Troubleshooting

### Error: "Could not resolve type id 'orderPlacedEvent'"

**Síntoma completo**:
```
Could not resolve type id 'orderPlacedEvent' into a subtype of [simple type, class java.lang.Object]
```

**Causa**: Falta configuración de `spring.json.type.mapping` en application.yml.

**Solución**:
```yaml
spring:
  kafka:
    consumer:
      properties:
        spring.json.type.mapping: orderPlacedEvent:dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent
```

**Verificar**: El paquete debe ser de **inventory-service**, NO de order-service.

### Error: "The class 'dev.alefiengo.orderservice.kafka.event.OrderPlacedEvent' is not in the trusted packages"

**Causa**: Usaste el paquete de order-service en lugar de inventory-service.

**Solución**: La clase debe estar en `dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent`.

**Verificar estructura**:
```bash
ls -la ~/workspace/inventory-service/src/main/java/dev/alefiengo/inventoryservice/kafka/event/
# Debe mostrar: OrderPlacedEvent.java
```

### Error: "No default constructor found"

**Síntoma**:
```
InvalidDefinitionException: Cannot construct instance of OrderPlacedEvent
(no Creators, like default constructor, exist)
```

**Causa**: Falta constructor vacío en OrderPlacedEvent.

**Solución**: Agregar en OrderPlacedEvent.java:
```java
public OrderPlacedEvent() {
}
```

### Consumer no recibe mensajes

**Verificaciones**:

1. **Kafka está corriendo**:
```bash
docker compose ps | grep kafka
# Debe mostrar "Up"
```

2. **Topic existe**:
```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list | grep orders.placed
# Debe mostrar: ecommerce.orders.placed
```

3. **Hay mensajes en el topic**:
```bash
docker exec -it kafka kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning
# Debe mostrar mensajes JSON
```

4. **Consumer group tiene offsets**:
```bash
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe
```

Si `CURRENT-OFFSET` = `LOG-END-OFFSET`, significa que ya procesó todos los mensajes.

5. **auto-offset-reset está en earliest**:
```yaml
spring:
  kafka:
    consumer:
      auto-offset-reset: earliest  # NO latest
```

### Error: "Inventory item not found for productId: 1"

**Causa**: No existe item de inventario para el producto en la orden.

**Solución**: Crear item de inventario primero:
```bash
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Test Product",
    "initialStock": 100
  }'
```

**Orden de ejecución**:
1. Crear items de inventario (POST /api/inventory)
2. Publicar eventos OrderPlacedEvent

### Warning: "Insufficient stock" pero no se publica OrderCancelledEvent

**Esto es esperado en este lab**.

En Lab 02 implementaremos:
- OrderConfirmedEvent (cuando hay stock)
- OrderCancelledEvent (cuando NO hay stock)

Por ahora, solo se loguea el error.

### Consumer procesa el mismo mensaje múltiples veces

**Causa posible**: La aplicación crashea ANTES de commitear el offset.

**Flujo**:
```
1. Consumer recibe mensaje offset 5
2. Procesa mensaje (reserva stock)
3. [CRASH antes de commit]
4. Al reiniciar, Kafka reenvía offset 5
5. Procesa OTRA VEZ (stock se reserva dos veces)
```

**Solución**: Implementar **idempotencia** (fuera de alcance de este lab).

**Opciones**:
- Guardar orderId procesados en tabla
- Verificar si ya existe `reservedStock` para esa orden
- Usar transactions de Kafka (avanzado)

### Logs muestran "Received OrderPlacedEvent" pero no "Stock reserved successfully"

**Causa**: La excepción se lanza en `processOrderPlaced()` y es capturada por el try-catch.

**Verificar logs**:
```
ERROR ... Error processing OrderPlacedEvent: orderId=X, error=...
```

**Causas comunes**:
- Inventory item no existe
- Stock insuficiente
- Error de BD (connection, constraint violation)

---

## Desafío adicional

### Desafío 1: Implementar manual offset commit

Modificar OrderEventConsumer para usar `Acknowledgment` y commitear offset manualmente solo si el procesamiento fue exitoso.

**Pistas**:
```java
@KafkaListener(topics = "ecommerce.orders.placed", groupId = "inventory-service")
public void consume(OrderPlacedEvent event, Acknowledgment ack) {
    try {
        inventoryService.processOrderPlaced(event);
        ack.acknowledge();  // Commit solo si exitoso
    } catch (Exception e) {
        log.error("Error, NOT committing offset");
        // NO llames ack.acknowledge() → Kafka reintentará
    }
}
```

**Configuración adicional**:
```yaml
spring:
  kafka:
    listener:
      ack-mode: manual
```

### Desafío 2: Agregar retry con backoff exponencial

Si `processOrderPlaced()` falla, reintentar 3 veces con espera creciente (1s, 2s, 4s).

**Pistas**:
- Usar `@Retryable` de Spring Retry
- Configurar `maxAttempts` y `backoff`

### Desafío 3: Loguear metadata de Kafka

Modificar consumer para loguear información adicional:
- Partition del mensaje
- Offset del mensaje
- Timestamp del mensaje

**Pistas**:
```java
@KafkaListener(topics = "ecommerce.orders.placed")
public void consume(
    @Payload OrderPlacedEvent event,
    @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
    @Header(KafkaHeaders.OFFSET) long offset,
    @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp
) {
    log.info("Received from partition {} at offset {} (timestamp {})",
             partition, offset, timestamp);
    // ...
}
```

### Desafío 4: Implementar health check para consumer

Agregar endpoint REST que verifique si el consumer está activo y procesando mensajes.

**Pistas**:
- Crear `@RestController` con endpoint `/actuator/kafka/consumer-status`
- Inyectar `KafkaListenerEndpointRegistry`
- Verificar si listener está corriendo

---

## Recursos adicionales

- [Spring Kafka @KafkaListener](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html)
- [Kafka Consumer Configuration](https://kafka.apache.org/documentation/#consumerconfigs)
- [JsonDeserializer Documentation](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/support/serializer/JsonDeserializer.html)
- [Consumer Groups Deep Dive](https://kafka.apache.org/documentation/#consumergroups)
- [Offset Management](https://kafka.apache.org/documentation/#offsetmanagement)
- [Event-Driven Microservices Patterns](https://www.confluent.io/blog/event-driven-microservices-with-spring-boot-and-kafka/)
