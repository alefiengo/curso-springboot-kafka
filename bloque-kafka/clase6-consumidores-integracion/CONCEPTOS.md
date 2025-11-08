# Conceptos - Clase 6: Consumidores e Integración

Este documento complementa los laboratorios prácticos con teoría fundamental sobre consumidores Kafka y arquitecturas event-driven.

---

## Tabla de Contenidos

1. [Consumidores Kafka](#consumidores-kafka)
2. [Consumer Groups](#consumer-groups)
3. [Offset Management](#offset-management)
4. [Deserialización de Eventos](#deserialización-de-eventos)
5. [Comunicación Asíncrona](#comunicación-asíncrona)
6. [Eventual Consistency](#eventual-consistency)
7. [Patrón Saga](#patrón-saga)
8. [Idempotencia](#idempotencia)

---

## Consumidores Kafka

### ¿Qué es un consumidor?

Un **consumidor** es una aplicación que lee mensajes de topics de Kafka. A diferencia de los productores (que escriben), los consumidores leen mensajes en el orden en que fueron publicados.

**Características clave**:
- Lee mensajes de uno o más topics
- Mantiene un puntero (offset) de qué mensajes ha leído
- Puede procesar mensajes de forma paralela (particiones)
- Puede formar parte de un consumer group

### Anatomía de un Consumer en Spring Kafka

```java
@Component
public class OrderEventConsumer {

    @KafkaListener(
        topics = "ecommerce.orders.placed",  // Topic a escuchar
        groupId = "inventory-service"        // Consumer group
    )
    public void consumeOrderPlaced(OrderPlacedEvent event) {
        // Procesar evento
        log.info("Received order: {}", event.getOrderId());
    }
}
```

**Componentes**:
- `@KafkaListener`: Marca el método como listener de Kafka
- `topics`: Qué topic(s) escuchar (puede ser array)
- `groupId`: A qué consumer group pertenece
- Parámetro del método: El evento deserializado automáticamente

---

## Consumer Groups

### ¿Qué es un Consumer Group?

Un **consumer group** es un conjunto de consumidores que trabajan juntos para consumir mensajes de un topic. Kafka distribuye las particiones del topic entre los consumidores del grupo.

**Regla de oro**: Cada mensaje es consumido por **UN SOLO** consumidor dentro del grupo.

### Ejemplo Visual

```
Topic: ecommerce.orders.placed (3 particiones)

┌─── Partition 0 ───┐
│ msg1, msg4, msg7  │ ──────┐
└───────────────────┘        │
                             ├──► Consumer A (inventory-service)
┌─── Partition 1 ───┐        │
│ msg2, msg5, msg8  │ ──────┤
└───────────────────┘        │
                             ├──► Consumer B (inventory-service)
┌─── Partition 2 ───┐        │
│ msg3, msg6, msg9  │ ──────┘
└───────────────────┘

Consumer Group: inventory-service
```

**Ventajas**:
- **Paralelismo**: Múltiples consumidores procesan diferentes particiones simultáneamente
- **Load balancing**: Kafka distribuye automáticamente las particiones
- **Fault tolerance**: Si un consumidor falla, Kafka redistribuye sus particiones

### Consumer Groups vs Broadcast

**Mismo consumer group** (load balancing):
```
inventory-service-1 ─┐
                     ├──► group: inventory-service
inventory-service-2 ─┘

Resultado: Solo UNO recibe cada mensaje
```

**Diferentes consumer groups** (broadcast):
```
inventory-service ──► group: inventory-service
analytics-service ──► group: analytics-service

Resultado: AMBOS reciben cada mensaje
```

---

## Offset Management

### ¿Qué es un offset?

Un **offset** es un número secuencial que identifica cada mensaje dentro de una partición. Es como un marcador de libro que indica hasta dónde has leído.

```
Partition 0:
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │  5  │
└─────┴─────┴─────┴─────┴─────┴─────┘
                    ↑
                  offset actual
```

### Auto-offset-reset

Controla qué hacer cuando un consumidor NO tiene offset previo:

```yaml
spring:
  kafka:
    consumer:
      auto-offset-reset: earliest  # O 'latest'
```

**Opciones**:

| Valor | Comportamiento | Cuándo usar |
|-------|---------------|-------------|
| `earliest` | Lee desde el primer mensaje disponible | Quieres procesar mensajes históricos |
| `latest` | Lee solo mensajes nuevos (desde ahora) | Solo te importan mensajes futuros |

**Ejemplo práctico**:

```
Topic tiene mensajes: [0, 1, 2, 3, 4, 5]

Consumer nuevo con earliest:
- Lee desde offset 0
- Procesa: 0, 1, 2, 3, 4, 5

Consumer nuevo con latest:
- Lee desde offset 6 (siguiente)
- Procesa: solo mensajes futuros
```

### Commit de Offsets

Cuando un consumidor procesa un mensaje, debe "commitear" el offset para indicar que ya lo procesó.

**Auto-commit** (por defecto):
```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: true  # Default
```

Spring Kafka automáticamente guarda el progreso.

**Manual commit** (control fino):
```java
@KafkaListener(topics = "my-topic")
public void consume(String message, Acknowledgment ack) {
    // Procesar mensaje
    processMessage(message);

    // Confirmar manualmente
    ack.acknowledge();
}
```

---

## Deserialización de Eventos

### JsonDeserializer

Spring Kafka deserializa automáticamente JSON a objetos Java usando `JsonDeserializer`.

```yaml
spring:
  kafka:
    consumer:
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

### Type Mapping

Indica qué clase Java corresponde a cada tipo de evento:

```yaml
spring:
  kafka:
    consumer:
      properties:
        spring.json.type.mapping: |
          orderPlacedEvent:dev.alefiengo.inventoryservice.kafka.event.OrderPlacedEvent,
          orderConfirmedEvent:dev.alefiengo.orderservice.kafka.event.OrderConfirmedEvent
```

**Formato**: `nombreEvento:paquete.completo.NombreClase`

### Trusted Packages

Por seguridad, JsonDeserializer solo deserializa clases de paquetes confiables:

```yaml
spring:
  kafka:
    consumer:
      properties:
        spring.json.trusted.packages: dev.alefiengo.*
```

**Wildcard `.*`**: Confía en todos los subpaquetes de `dev.alefiengo`

### Ejemplo Completo

**Producer publica**:
```json
{
  "orderId": 1,
  "productId": 5,
  "quantity": 3,
  "timestamp": "2025-11-08T10:30:00Z"
}
```

**Consumer recibe**:
```java
@KafkaListener(topics = "ecommerce.orders.placed")
public void consume(OrderPlacedEvent event) {
    // event ya es un objeto Java
    Long orderId = event.getOrderId();  // 1
    Integer qty = event.getQuantity();  // 3
}
```

**Requisitos para deserialización**:
1. Constructor sin argumentos
2. Getters y setters
3. Type mapping configurado
4. Paquete en trusted packages

---

## Comunicación Asíncrona

### Síncrona vs Asíncrona

**Comunicación Síncrona** (REST):
```
order-service ──HTTP GET──► product-service
              ◄───response───
```

- Bloqueante: order-service espera respuesta
- Acoplamiento: order-service depende que product-service esté disponible
- Latencia: Suma de tiempos de ambos servicios

**Comunicación Asíncrona** (Kafka):
```
order-service ──OrderPlaced──► Kafka ──► inventory-service
              (no espera)
```

- No bloqueante: order-service continúa inmediatamente
- Desacoplamiento: Servicios no se conocen entre sí
- Resiliencia: Si inventory-service está caído, el mensaje espera en Kafka

### Ventajas de Asíncrona

1. **Desacoplamiento temporal**: Los servicios no necesitan estar disponibles al mismo tiempo
2. **Desacoplamiento de localización**: No necesitan conocer URLs de otros servicios
3. **Escalabilidad**: Agregar consumidores no afecta productores
4. **Resiliencia**: Mensajes persisten aunque consumidores fallen

### Desventajas

1. **Complejidad**: Más difícil de debuggear que REST síncrono
2. **Eventual consistency**: Los datos no se actualizan instantáneamente
3. **Duplicación de código**: Eventos duplicados entre servicios
4. **Orden no garantizado** (entre particiones)

---

## Eventual Consistency

### ¿Qué es Eventual Consistency?

En arquitecturas event-driven, los datos NO se sincronizan instantáneamente. Existe una ventana de tiempo donde diferentes servicios tienen vistas inconsistentes del sistema.

**Ejemplo**:

```
T0: Cliente crea orden → order-service
    - order-service: Order(id=1, status=PENDING)
    - inventory-service: No sabe nada de order 1

T1: OrderPlacedEvent publicado a Kafka
    - order-service: Order(id=1, status=PENDING)
    - inventory-service: Todavía no ha consumido el evento

T2: inventory-service consume evento
    - order-service: Order(id=1, status=PENDING)
    - inventory-service: Procesando...

T3: inventory-service publica OrderConfirmedEvent
    - order-service: Order(id=1, status=PENDING) ← aún no actualizado
    - inventory-service: Ya procesó

T4: order-service consume OrderConfirmedEvent
    - order-service: Order(id=1, status=CONFIRMED) ← actualizado
    - inventory-service: Consistente

T4+: Estado consistente
```

**Implicaciones**:

- El cliente puede ver estado PENDING por algunos milisegundos
- Dos consultas consecutivas pueden retornar estados diferentes
- La UI debe diseñarse para esto (e.g., "Procesando orden...")

### Eventual vs Strong Consistency

**Strong Consistency** (transacciones ACID):
```
BEGIN TRANSACTION
  INSERT INTO orders ...
  UPDATE inventory ...
COMMIT

Si falla cualquier paso, TODO se revierte
```

**Eventual Consistency** (eventos):
```
1. INSERT INTO orders → OK
2. Publish OrderPlacedEvent → OK
3. [tiempo pasa...]
4. Consume event → UPDATE inventory → OK

Si falla paso 3, orden existe pero inventario no actualizado
```

---

## Patrón Saga

### ¿Qué es una Saga?

Una **Saga** es un patrón para manejar transacciones distribuidas usando una secuencia de transacciones locales coordinadas por eventos.

**Problema**: No puedes tener transacciones ACID entre múltiples bases de datos (microservicios).

**Solución**: Dividir la transacción en pasos locales con compensación si algo falla.

### Saga en nuestro e-commerce

**Flujo feliz** (saga exitosa):
```
1. order-service: Crear orden (PENDING)
2. inventory-service: Reservar stock
3. inventory-service: Publicar OrderConfirmedEvent
4. order-service: Actualizar orden (CONFIRMED)
```

**Flujo de error** (saga compensada):
```
1. order-service: Crear orden (PENDING)
2. inventory-service: No hay stock
3. inventory-service: Publicar OrderCancelledEvent
4. order-service: Actualizar orden (CANCELLED) ← compensación
```

### Tipos de Saga

**Orquestación** (con coordinador central):
```
┌──────────────┐
│ Order Service│ (orquestador)
└──────┬───────┘
       │ 1. Reserve stock
       ├─────────► Inventory Service
       │ 2. Charge payment
       ├─────────► Payment Service
       │ 3. Ship order
       └─────────► Shipping Service
```

**Coreografía** (sin coordinador - **usamos este**):
```
Order Service ──OrderPlaced──► Kafka ──► Inventory Service
      ↑                                         │
      │                                         │
      └────── OrderConfirmed/Cancelled ◄────────┘
```

Cada servicio escucha eventos y reacciona, sin coordinador central.

---

## Idempotencia

### ¿Qué es Idempotencia?

Una operación es **idempotente** si ejecutarla múltiples veces produce el mismo resultado que ejecutarla una sola vez.

**Ejemplo idempotente**:
```java
// Ejecutar 1 vez o 100 veces → mismo resultado
UPDATE orders SET status = 'CONFIRMED' WHERE id = 1;
```

**Ejemplo NO idempotente**:
```java
// Ejecutar 2 veces → problema!
UPDATE inventory SET stock = stock - 5 WHERE product_id = 1;
```

### ¿Por qué importa en Kafka?

Kafka garantiza **at-least-once delivery**: Un mensaje puede ser entregado más de una vez.

**Escenario problemático**:
```
1. Consumer procesa OrderPlacedEvent
2. Reserva 5 unidades de stock (stock = 100 - 5 = 95)
3. Consumer crashea ANTES de commitear offset
4. Kafka reenvía el mismo evento
5. Consumer reserva 5 unidades OTRA VEZ (stock = 95 - 5 = 90)

Resultado: Stock incorrecto!
```

### Cómo lograr idempotencia

**Opción 1**: Guardar IDs de eventos procesados
```java
@Transactional
public void processOrder(OrderPlacedEvent event) {
    // Verificar si ya procesamos este evento
    if (processedEvents.contains(event.getOrderId())) {
        log.warn("Event already processed: {}", event.getOrderId());
        return;
    }

    // Procesar
    reserveStock(event);

    // Marcar como procesado
    processedEvents.add(event.getOrderId());
}
```

**Opción 2**: Usar operaciones idempotentes
```java
// En lugar de restar del stock actual...
inventory.setReservedStock(inventory.getReservedStock() - quantity);

// Calcular basándose en el evento
Order order = orderRepository.findById(event.getOrderId());
if (order.getStatus() == PENDING) {  // Solo si aún está pendiente
    reserveStock(event);
}
```

**Opción 3**: Usar versioning
```java
@Version
private Long version;  // JPA optimistic locking
```

**En este curso**: Mencionamos el concepto pero NO implementamos idempotencia completa (fuera del alcance).

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Consumidor | Consumer | Aplicación que lee mensajes de Kafka |
| Grupo de consumidores | Consumer Group | Conjunto de consumidores trabajando juntos |
| Desplazamiento | Offset | Identificador secuencial de mensaje en partición |
| Deserialización | Deserialization | Convertir JSON a objeto Java |
| Consistencia eventual | Eventual Consistency | Datos consistentes después de un tiempo |
| Saga | Saga | Patrón para transacciones distribuidas |
| Idempotencia | Idempotency | Operación que produce mismo resultado al repetirse |
| Compensación | Compensation | Acción para revertir efecto de transacción fallida |
| Difusión | Broadcast | Enviar mensaje a múltiples consumidores |