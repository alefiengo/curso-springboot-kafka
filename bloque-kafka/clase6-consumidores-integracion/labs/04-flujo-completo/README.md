# Lab 04: Flujo Completo End-to-End

Ejecutar los 3 microservicios simultáneamente (product-service, order-service, inventory-service) y demostrar el flujo completo de comunicación asíncrona mediante eventos Kafka, validando el patrón Saga implementado desde la creación de una orden hasta su confirmación o cancelación automática.

---

## Objetivo

Integrar y probar la arquitectura completa de microservicios con Kafka, ejecutar escenarios realistas de negocio, monitorear eventos en tiempo real con Kafka CLI, verificar consistencia eventual entre servicios y consolidar todos los conceptos aprendidos en la clase 6.

---

## Arquitectura que ejecutaremos

```
┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
│  product-service    │      │   order-service     │      │ inventory-service   │
│  (puerto 8080)      │      │   (puerto 8081)     │      │  (puerto 8082)      │
├─────────────────────┤      ├─────────────────────┤      ├─────────────────────┤
│ PostgreSQL          │      │ PostgreSQL          │      │ PostgreSQL          │
│ ecommerce           │      │ ecommerce_orders    │      │ ecommerce_inventory │
├─────────────────────┤      ├─────────────────────┤      ├─────────────────────┤
│ PRODUCER:           │      │ PRODUCER:           │      │ PRODUCER:           │
│ products.created    │      │ orders.placed       │      │ orders.confirmed    │
│                     │      │                     │      │ orders.cancelled    │
│                     │      │ CONSUMER:           │      │                     │
│                     │      │ orders.confirmed    │      │ CONSUMER:           │
│                     │      │ orders.cancelled    │      │ orders.placed       │
└─────────────────────┘      └─────────────────────┘      └─────────────────────┘
                                       ▲                            │
                                       │                            │
                                       └────────── Kafka ───────────┘
                                         (4 topics activos)
```

---

## Comandos a ejecutar

### Paso 1: Verificar infraestructura Kafka

```bash
cd ~/workspace/kafka-infrastructure

# Verificar que Kafka y PostgreSQL están corriendo
docker compose ps
```

**Output esperado**:
```
NAME       IMAGE                          STATUS
kafka      confluentinc/cp-kafka:latest   Up
postgres   postgres:15                    Up
```

**Si no están corriendo**:
```bash
docker compose up -d
```

### Paso 2: Verificar bases de datos

```bash
# Verificar que las 3 bases de datos existen
docker exec -it postgres psql -U postgres -c "\l" | grep ecommerce
```

**Output esperado**:
```
 ecommerce           | postgres | UTF8     | C       | C     |
 ecommerce_inventory | postgres | UTF8     | C       | C     |
 ecommerce_orders    | postgres | UTF8     | C       | C     |
```

**Si falta alguna**:
```bash
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce;"
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_orders;"
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_inventory;"
```

### Paso 3: Verificar topics Kafka

```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list | grep ecommerce
```

**Output esperado**:
```
ecommerce.orders.cancelled
ecommerce.orders.confirmed
ecommerce.orders.placed
ecommerce.products.created
```

**Si falta algún topic**, ejecutar:
```bash
docker exec -it kafka bash

# Topics que deben existir:
kafka-topics --bootstrap-server localhost:9092 --create --topic ecommerce.products.created --partitions 3 --replication-factor 1
kafka-topics --bootstrap-server localhost:9092 --create --topic ecommerce.orders.placed --partitions 3 --replication-factor 1
kafka-topics --bootstrap-server localhost:9092 --create --topic ecommerce.orders.confirmed --partitions 3 --replication-factor 1
kafka-topics --bootstrap-server localhost:9092 --create --topic ecommerce.orders.cancelled --partitions 3 --replication-factor 1

exit
```

### Paso 4: Limpiar datos de pruebas anteriores (opcional)

```bash
# Truncar tablas para empezar desde cero
docker exec -it postgres psql -U postgres -d ecommerce -c "TRUNCATE TABLE products, categories RESTART IDENTITY CASCADE;"
docker exec -it postgres psql -U postgres -d ecommerce_orders -c "TRUNCATE TABLE customer_orders RESTART IDENTITY CASCADE;"
docker exec -it postgres psql -U postgres -d ecommerce_inventory -c "TRUNCATE TABLE inventory_items RESTART IDENTITY CASCADE;"
```

**Output esperado**:
```
TRUNCATE TABLE
TRUNCATE TABLE
TRUNCATE TABLE
```

### Paso 5: Abrir 8 terminales

Vamos a necesitar 8 terminales organizadas así:

**Terminales de microservicios** (1-3):
- Terminal 1: product-service
- Terminal 2: order-service
- Terminal 3: inventory-service

**Terminales de Kafka consumers CLI** (4-7):
- Terminal 4: Consumer de products.created
- Terminal 5: Consumer de orders.placed
- Terminal 6: Consumer de orders.confirmed
- Terminal 7: Consumer de orders.cancelled

**Terminal de comandos** (8):
- Terminal 8: Ejecutar curls y consultas

### Paso 6: Iniciar product-service (Terminal 1)

```bash
cd ~/workspace/product-service
mvn spring-boot:run
```

**Esperar mensaje**:
```
Started ProductServiceApplication in X.XXX seconds
```

### Paso 7: Iniciar order-service (Terminal 2)

```bash
cd ~/workspace/order-service
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

**Esperar mensaje**:
```
Started OrderServiceApplication in X.XXX seconds
```

### Paso 8: Iniciar inventory-service (Terminal 3)

```bash
cd ~/workspace/inventory-service
mvn spring-boot:run
```

**Esperar mensaje**:
```
Started InventoryServiceApplication in X.XXX seconds
```

### Paso 9: Iniciar consumers Kafka CLI

**Terminal 4** (products.created):
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"
```

**Terminal 5** (orders.placed):
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"
```

**Terminal 6** (orders.confirmed):
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"
```

**Terminal 7** (orders.cancelled):
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.cancelled \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"
```

### Paso 10: Crear categoría (Terminal 8)

```bash
curl -X POST http://localhost:8080/api/categories \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Electrónica",
    "description": "Productos electrónicos y tecnología"
  }'
```

**Output esperado**:
```json
{
  "id": 1,
  "name": "Electrónica",
  "description": "Productos electrónicos y tecnología",
  "createdAt": "2025-11-08T13:00:00.123Z"
}
```

### Paso 11: Crear productos (Terminal 8)

```bash
# Producto 1: Laptop
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop Dell XPS 15",
    "description": "Laptop profesional con Intel i7",
    "price": 1500.00,
    "categoryId": 1
  }'

# Producto 2: Mouse
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Mouse Logitech MX Master",
    "description": "Mouse ergonómico inalámbrico",
    "price": 99.99,
    "categoryId": 1
  }'

# Producto 3: Teclado
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Teclado Mecánico Keychron K2",
    "description": "Teclado mecánico inalámbrico",
    "price": 89.99,
    "categoryId": 1
  }'
```

**Observar Terminal 4**: Deberías ver 3 eventos ProductCreatedEvent.

**Output esperado en Terminal 4**:
```
1:{"productId":1,"name":"Laptop Dell XPS 15","price":1500.00,...}
2:{"productId":2,"name":"Mouse Logitech MX Master","price":99.99,...}
3:{"productId":3,"name":"Teclado Mecánico Keychron K2","price":89.99,...}
```

### Paso 12: Crear items de inventario (Terminal 8)

```bash
# Inventario para producto 1 (Laptop) - 50 unidades
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "initialStock": 50
  }'

# Inventario para producto 2 (Mouse) - 200 unidades
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 2,
    "productName": "Mouse Logitech MX Master",
    "initialStock": 200
  }'

# Inventario para producto 3 (Teclado) - 5 unidades (stock bajo a propósito)
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 3,
    "productName": "Teclado Mecánico Keychron K2",
    "initialStock": 5
  }'
```

### Paso 13: Verificar estado inicial (Terminal 8)

```bash
# Listar productos
curl http://localhost:8080/api/products

# Listar inventario
curl http://localhost:8082/api/inventory

# Listar órdenes (debe estar vacío)
curl http://localhost:8081/api/orders
```

---

## Escenario 1: Orden exitosa (CON stock suficiente)

### Paso 14: Crear orden para Laptop (5 unidades)

```bash
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 5,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com",
    "totalAmount": 7500.00
  }'
```

**Output esperado**:
```json
{
  "id": 1,
  "productId": 1,
  "quantity": 5,
  "customerName": "Juan Perez",
  "customerEmail": "juan@example.com",
  "totalAmount": 7500.00,
  "status": "PENDING",
  "createdAt": "2025-11-08T13:05:00.123Z"
}
```

### Paso 15: Observar flujo de eventos

**Terminal 2** (order-service) - Logs esperados:
```
INFO ... Creating order: productId=1, quantity=5
INFO ... OrderPlacedEvent published successfully: orderId=1, partition=0, offset=0
[... 1-2 segundos después ...]
INFO ... Received OrderConfirmedEvent: orderId=1
INFO ... Confirming order: orderId=1
INFO ... Order confirmed: orderId=1, newStatus=CONFIRMED
```

**Terminal 3** (inventory-service) - Logs esperados:
```
INFO ... Received OrderPlacedEvent: orderId=1, productId=1, quantity=5
INFO ... Processing order: orderId=1, productId=1, quantity=5
INFO ... Stock reserved successfully: orderId=1, newAvailableStock=45, newReservedStock=5
INFO ... OrderConfirmedEvent published successfully: orderId=1, partition=0, offset=0
```

**Terminal 5** (orders.placed) - Evento observado:
```
1:{"orderId":1,"productId":1,"quantity":5,"customerName":"Juan Perez",...}
```

**Terminal 6** (orders.confirmed) - Evento observado:
```
1:{"orderId":1,"productId":1,"quantity":5,"availableStockAfterReservation":45,"reservedStockAfterReservation":5,...}
```

### Paso 16: Verificar estado actualizado

```bash
# Consultar orden - Debe estar CONFIRMED
curl http://localhost:8081/api/orders/1

# Consultar inventario - Stock debe estar reservado
curl http://localhost:8082/api/inventory/product/1
```

**Output esperado (orden)**:
```json
{
  "id": 1,
  "productId": 1,
  "quantity": 5,
  "customerName": "Juan Perez",
  "customerEmail": "juan@example.com",
  "totalAmount": 7500.00,
  "status": "CONFIRMED",
  "createdAt": "2025-11-08T13:05:00.123Z"
}
```

**Output esperado (inventario)**:
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

## Escenario 2: Orden rechazada (SIN stock suficiente)

### Paso 17: Crear orden para Teclado (10 unidades - más del disponible)

```bash
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 3,
    "quantity": 10,
    "customerName": "Maria Lopez",
    "customerEmail": "maria@example.com",
    "totalAmount": 899.90
  }'
```

**Output esperado**:
```json
{
  "id": 2,
  "productId": 3,
  "quantity": 10,
  "customerName": "Maria Lopez",
  "customerEmail": "maria@example.com",
  "totalAmount": 899.90,
  "status": "PENDING",
  "createdAt": "2025-11-08T13:10:00.123Z"
}
```

### Paso 18: Observar flujo de cancelación

**Terminal 2** (order-service) - Logs esperados:
```
INFO ... Creating order: productId=3, quantity=10
INFO ... OrderPlacedEvent published successfully: orderId=2
[... 1-2 segundos después ...]
INFO ... Received OrderCancelledEvent: orderId=2, reason=Insufficient stock
INFO ... Cancelling order: orderId=2, reason=Insufficient stock
INFO ... Order cancelled: orderId=2, newStatus=CANCELLED
```

**Terminal 3** (inventory-service) - Logs esperados:
```
INFO ... Received OrderPlacedEvent: orderId=2, productId=3, quantity=10
INFO ... Processing order: orderId=2, productId=3, quantity=10
WARN ... Insufficient stock for order: orderId=2, requested=10, available=5
INFO ... OrderCancelledEvent published successfully: orderId=2
```

**Terminal 5** (orders.placed) - Evento observado:
```
2:{"orderId":2,"productId":3,"quantity":10,"customerName":"Maria Lopez",...}
```

**Terminal 7** (orders.cancelled) - Evento observado:
```
2:{"orderId":2,"productId":3,"requestedQuantity":10,"availableStock":5,"reason":"Insufficient stock",...}
```

### Paso 19: Verificar orden cancelada

```bash
# Consultar orden - Debe estar CANCELLED
curl http://localhost:8081/api/orders/2

# Consultar inventario - Stock NO debe haber cambiado
curl http://localhost:8082/api/inventory/product/3
```

**Output esperado (orden)**:
```json
{
  "id": 2,
  "productId": 3,
  "quantity": 10,
  "customerName": "Maria Lopez",
  "customerEmail": "maria@example.com",
  "totalAmount": 899.90,
  "status": "CANCELLED",
  "createdAt": "2025-11-08T13:10:00.123Z"
}
```

**Output esperado (inventario)**:
```json
{
  "id": 3,
  "productId": 3,
  "availableStock": 5,
  "reservedStock": 0,
  "totalStock": 5
}
```

**Observar**: Stock NO cambió (orden fue rechazada).

---

## Escenario 3: Múltiples órdenes simultáneas

### Paso 20: Crear 5 órdenes en rápida sucesión

```bash
# Orden 3: Mouse (10 unidades)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":2,"quantity":10,"customerName":"Carlos Ruiz","customerEmail":"carlos@example.com","totalAmount":999.90}'

# Orden 4: Laptop (3 unidades)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":3,"customerName":"Ana Martinez","customerEmail":"ana@example.com","totalAmount":4500.00}'

# Orden 5: Mouse (50 unidades)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":2,"quantity":50,"customerName":"Pedro Gomez","customerEmail":"pedro@example.com","totalAmount":4999.50}'

# Orden 6: Teclado (2 unidades - debe confirmar)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":3,"quantity":2,"customerName":"Laura Diaz","customerEmail":"laura@example.com","totalAmount":179.98}'

# Orden 7: Laptop (100 unidades - debe cancelar)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":100,"customerName":"Roberto Silva","customerEmail":"roberto@example.com","totalAmount":150000.00}'
```

### Paso 21: Observar procesamiento en paralelo

**Observar Terminales 2 y 3**: Verás logs intercalados procesando múltiples órdenes simultáneamente.

**Observar Terminales 5, 6, 7**: Verás eventos apareciendo en tiempo real.

### Paso 22: Verificar resultado de todas las órdenes

```bash
curl http://localhost:8081/api/orders
```

**Resultado esperado**:
- Orden 1: CONFIRMED (Laptop, 5 unidades)
- Orden 2: CANCELLED (Teclado, 10 unidades - sin stock)
- Orden 3: CONFIRMED (Mouse, 10 unidades)
- Orden 4: CONFIRMED (Laptop, 3 unidades)
- Orden 5: CONFIRMED (Mouse, 50 unidades)
- Orden 6: CONFIRMED (Teclado, 2 unidades)
- Orden 7: CANCELLED (Laptop, 100 unidades - sin stock)

### Paso 23: Verificar inventario final

```bash
curl http://localhost:8082/api/inventory
```

**Estado esperado**:

**Producto 1 (Laptop)**:
- Stock inicial: 50
- Reservado: 5 (orden 1) + 3 (orden 4) = 8
- Disponible: 50 - 8 = 42

**Producto 2 (Mouse)**:
- Stock inicial: 200
- Reservado: 10 (orden 3) + 50 (orden 5) = 60
- Disponible: 200 - 60 = 140

**Producto 3 (Teclado)**:
- Stock inicial: 5
- Reservado: 2 (orden 6) = 2
- Disponible: 5 - 2 = 3

---

## Verificación en PostgreSQL

### Paso 24: Consultar tablas directamente

**Ver órdenes**:
```bash
docker exec -it postgres psql -U postgres -d ecommerce_orders \
  -c "SELECT id, product_id, quantity, customer_name, status FROM customer_orders ORDER BY id;"
```

**Output esperado**:
```
 id | product_id | quantity | customer_name  |  status
----+------------+----------+----------------+-----------
  1 |          1 |        5 | Juan Perez     | CONFIRMED
  2 |          3 |       10 | Maria Lopez    | CANCELLED
  3 |          2 |       10 | Carlos Ruiz    | CONFIRMED
  4 |          1 |        3 | Ana Martinez   | CONFIRMED
  5 |          2 |       50 | Pedro Gomez    | CONFIRMED
  6 |          3 |        2 | Laura Diaz     | CONFIRMED
  7 |          1 |      100 | Roberto Silva  | CANCELLED
(7 rows)
```

**Ver inventario**:
```bash
docker exec -it postgres psql -U postgres -d ecommerce_inventory \
  -c "SELECT product_id, product_name, available_stock, reserved_stock FROM inventory_items ORDER BY product_id;"
```

**Output esperado**:
```
 product_id |        product_name         | available_stock | reserved_stock
------------+-----------------------------+-----------------+----------------
          1 | Laptop Dell XPS 15          |              42 |              8
          2 | Mouse Logitech MX Master    |             140 |             60
          3 | Teclado Mecánico Keychron K2|               3 |              2
(3 rows)
```

---

## Monitoreo de Consumer Groups

### Paso 25: Ver estado de consumer groups

```bash
# Consumer group de order-service
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-service \
  --describe

# Consumer group de inventory-service
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe
```

**Output esperado (order-service)**:
```
GROUP           TOPIC                      PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
order-service   ecommerce.orders.confirmed 0          5               5               0
order-service   ecommerce.orders.confirmed 1          0               0               0
order-service   ecommerce.orders.confirmed 2          0               0               0
order-service   ecommerce.orders.cancelled 0          1               1               0
order-service   ecommerce.orders.cancelled 1          1               1               0
order-service   ecommerce.orders.cancelled 2          0               0               0
```

**Interpretación**:
- `LAG = 0`: Todos los mensajes fueron procesados
- `CURRENT-OFFSET = LOG-END-OFFSET`: Al día

---

## Desglose del comando

### Flujo completo del Saga pattern

```
T0: Cliente → POST /api/orders
    order-service: Crea Order(id=1, status=PENDING)
    ↓

T1: order-service → Kafka
    Publica: OrderPlacedEvent(orderId=1, productId=1, quantity=5)
    Topic: ecommerce.orders.placed
    ↓

T2: Kafka → inventory-service
    Consumer recibe: OrderPlacedEvent
    ↓

T3: inventory-service → PostgreSQL
    Busca InventoryItem(productId=1)
    Valida: availableStock(50) >= quantity(5) ✓
    Reserva: availableStock=45, reservedStock=5
    Guarda cambios en BD
    ↓

T4: inventory-service → Kafka
    Publica: OrderConfirmedEvent(orderId=1, availableStock=45)
    Topic: ecommerce.orders.confirmed
    ↓

T5: Kafka → order-service
    Consumer recibe: OrderConfirmedEvent
    ↓

T6: order-service → PostgreSQL
    Busca Order(id=1)
    Actualiza: status=CONFIRMED
    Guarda cambios en BD
    ↓

T7: Cliente consulta GET /api/orders/1
    Respuesta: Order(id=1, status=CONFIRMED)
```

**Duración total**: T0 a T7 = 1-2 segundos

### Arquitectura de topics

```
ecommerce.products.created (3 particiones)
├─ Partition 0: [msg1, msg4, msg7]
├─ Partition 1: [msg2, msg5, msg8]
└─ Partition 2: [msg3, msg6, msg9]

ecommerce.orders.placed (3 particiones)
├─ Producer: order-service
└─ Consumer: inventory-service (group: inventory-service)

ecommerce.orders.confirmed (3 particiones)
├─ Producer: inventory-service
└─ Consumer: order-service (group: order-service)

ecommerce.orders.cancelled (3 particiones)
├─ Producer: inventory-service
└─ Consumer: order-service (group: order-service)
```

---

## Explicación detallada

### Eventual consistency observable

**Timeline real observada**:

```
13:05:00.123 - Cliente crea orden
13:05:00.150 - order-service guarda en BD (PENDING)
13:05:00.200 - Cliente recibe HTTP 201 (status=PENDING)
13:05:00.250 - OrderPlacedEvent publicado
13:05:00.300 - inventory-service recibe evento
13:05:00.380 - inventory-service reserva stock
13:05:00.450 - OrderConfirmedEvent publicado
13:05:00.500 - order-service recibe evento
13:05:00.550 - order-service actualiza orden (CONFIRMED)

Cliente consulta:
13:05:00.600 - GET /api/orders/1 → CONFIRMED
```

**Ventana de inconsistencia**: 500ms (orden PENDING en order-service pero stock ya reservado en inventory-service).

### Idempotencia en acción

**Experimento**: Reiniciar order-service mientras hay mensajes pendientes.

```bash
# Terminal 2: Ctrl+C para detener order-service

# Crear orden mientras está detenido
curl -X POST http://localhost:8081/api/orders/...
# Error: Connection refused (esperado)

# inventory-service SIGUE procesando órdenes viejas
# Publica OrderConfirmedEvent → Kafka

# Reiniciar order-service
mvn spring-boot:run

# order-service consume mensajes pendientes
# Si algún mensaje se repitió:
# → Verifica order.status != PENDING
# → Return sin hacer nada (idempotente)
```

### Load balancing con múltiples instancias

**Experimento avanzado**: Ejecutar 2 instancias de order-service.

```bash
# Terminal 2a: Instancia 1
cd ~/workspace/order-service
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081

# Terminal 2b: Instancia 2
cd ~/workspace/order-service
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8091
```

**Comportamiento**:
- Ambas instancias pertenecen al grupo "order-service"
- Kafka distribuye particiones entre ellas
- Ejemplo: Instancia 1 procesa partitions 0 y 1, Instancia 2 procesa partition 2
- Cada mensaje solo lo recibe UNA instancia

### Particionamiento y orden garantizado

**Observar en Terminal 6** (orders.confirmed):

```
1:{"orderId":1,...}  ← Partition 0
4:{"orderId":4,...}  ← Partition 0
...
```

**Orden garantizado**: Todas las órdenes con la misma key (orderId) van a la misma partición.

**Ejemplo**:
- Order 1 → Partition 0
- Order 2 → Partition 1
- Order 3 → Partition 0 (mismo hash que Order 1)

**Dentro de cada partición**: Orden FIFO garantizado.
**Entre particiones**: NO hay garantía de orden.

---

## Conceptos aprendidos

- **Saga pattern end-to-end**: Flujo completo desde creación de orden hasta confirmación/cancelación
- **Eventual consistency visual**: Ver estado PENDING → CONFIRMED en tiempo real
- **Microservicios desacoplados**: Cada servicio funciona independientemente
- **Communication via events**: NO hay llamadas REST entre servicios
- **Idempotencia necesaria**: Previene problemas con mensajes duplicados
- **Consumer groups**: Load balancing y offset management
- **Particionamiento**: Distribuir carga y garantizar orden por key
- **At-least-once delivery**: Mensajes NO se pierden aunque consumer falle
- **Resiliencia**: inventory-service caído → mensajes esperan en Kafka
- **Monitoreo de topics**: Observar flujo de eventos en tiempo real
- **Verificación de consistencia**: PostgreSQL confirma estado correcto

---

## Troubleshooting

### Orden se queda en PENDING indefinidamente

**Verificaciones**:

1. **inventory-service corriendo**:
```bash
curl http://localhost:8082/api/inventory
# Debe responder
```

2. **Item de inventario existe**:
```bash
curl http://localhost:8082/api/inventory/product/1
# Debe retornar item
```

3. **Eventos en Kafka**:
```bash
# Ver si OrderPlacedEvent fue publicado
docker exec -it kafka kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning
```

4. **Logs de inventory-service**:
```bash
# Buscar en Terminal 3:
# "Received OrderPlacedEvent"
# Si NO aparece → Consumer no está funcionando
```

5. **Consumer group lag**:
```bash
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe

# Si LAG > 0 → Hay mensajes pendientes
```

### Todos los microservicios corren pero no hay eventos

**Causa**: Topics no existen.

**Solución**:
```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# Si faltan topics, crearlos:
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create --topic ecommerce.orders.placed --partitions 3 --replication-factor 1
# (repetir para otros topics)
```

### Error: "Inventory item not found for productId: X"

**Causa**: Creaste orden para producto sin inventario.

**Solución**:
```bash
# Crear item de inventario primero
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Test Product",
    "initialStock": 100
  }'
```

### Stock negativo en inventario

**Síntoma**: `availableStock = -10`.

**Causa**: Múltiples instancias de inventory-service sin sincronización.

**Solución**: Para este curso, ejecuta UNA SOLA instancia de inventory-service.

**En producción**: Usar transacciones distribuidas o locks optimistas.

### Consumer CLI no muestra eventos

**Causa**: Consumer inició DESPUÉS de publicar eventos.

**Solución**: Agregar `--from-beginning`:
```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning
```

### Eventos aparecen en Terminal 6 pero orden no se confirma

**Verificar logs de order-service** (Terminal 2):
```bash
# Buscar:
ERROR ... Error confirming order: orderId=X
```

**Posibles causas**:
- Order no existe en BD (fue eliminada)
- Exception en método confirmOrder()
- Configuración de consumer incorrecta

---

## Desafío adicional

### Desafío 1: Implementar endpoint de estadísticas

Crear endpoint `GET /api/orders/stats` que retorne:
```json
{
  "totalOrders": 7,
  "confirmed": 5,
  "cancelled": 2,
  "pending": 0,
  "totalRevenueFromConfirmed": 5499.95
}
```

**Pistas**:
- Agregar método en OrderRepository: `countByStatus(OrderStatus status)`
- Calcular revenue multiplicando quantity por precio del producto

### Desafío 2: Agregar notificaciones por email (simuladas)

Crear consumer en order-service que escuche orders.confirmed y orders.cancelled, y loguee "Email sent to {customerEmail}".

**Pistas**:
- Crear `EmailService` con método `sendOrderConfirmedEmail()`
- Inyectar en OrderEventConsumer
- Llamar después de actualizar orden

### Desafío 3: Implementar health check endpoint

Crear endpoint `GET /actuator/health/kafka` que verifique:
- Kafka está accesible
- Todos los topics existen
- Consumer groups están activos

**Pistas**:
- Usar Spring Boot Actuator
- Crear custom HealthIndicator

### Desafío 4: Dashboard en tiempo real (avanzado)

Crear página HTML simple que consulte cada 2 segundos:
- Total de órdenes
- Órdenes confirmadas vs canceladas
- Stock disponible de cada producto

**Pistas**:
- Crear endpoint REST que agregue datos
- Usar JavaScript fetch() con setInterval()

---

## Recursos adicionales

- [Saga Pattern Implementation](https://microservices.io/patterns/data/saga.html)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Kafka Consumer Groups Deep Dive](https://kafka.apache.org/documentation/#consumergroups)
- [Spring Boot Kafka Integration](https://docs.spring.io/spring-kafka/reference/)
- [Eventual Consistency](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- [Idempotent Consumer Pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html)

---

## Resumen de comandos útiles

```bash
# Ver todos los topics
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# Ver consumer groups
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 --list

# Ver lag de un grupo
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-service --describe

# Resetear offsets (CUIDADO)
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-service \
  --reset-offsets --to-earliest --all-topics --execute

# Ver mensajes de un topic
docker exec -it kafka kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed --from-beginning

# Limpiar bases de datos
docker exec -it postgres psql -U postgres -d ecommerce_orders \
  -c "TRUNCATE TABLE customer_orders RESTART IDENTITY CASCADE;"

# Verificar estado de servicios
curl http://localhost:8080/api/products
curl http://localhost:8081/api/orders
curl http://localhost:8082/api/inventory
```

---

**Felicitaciones!** Has completado la implementación y prueba de una arquitectura completa de microservicios con comunicación asíncrona mediante Apache Kafka, implementando el patrón Saga para gestionar transacciones distribuidas.
