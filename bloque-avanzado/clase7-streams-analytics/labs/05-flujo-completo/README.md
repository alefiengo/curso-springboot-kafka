# Lab 05: Flujo completo end-to-end con analytics

---

## 1. Objetivo

Validar la integración completa de 4 microservicios (product, order, inventory, analytics) con analytics en tiempo real, verificando que todos los Kafka Streams funcionan correctamente y que el patrón CQRS está implementado.

---

## 2. Comandos a ejecutar

```bash
# Script completo de validación
# Guardar como validate-clase7.sh y ejecutar

#!/bin/bash

echo "=== Validación Clase 7: 4 Microservicios + Kafka Streams ==="

# 1. Verificar servicios corriendo
echo "1. Verificando servicios..."
curl -s http://localhost:8080/actuator/health | grep -q "UP" && echo "✓ product-service (8080)" || echo "✗ product-service FAIL"
curl -s http://localhost:8081/actuator/health | grep -q "UP" && echo "✓ order-service (8081)" || echo "✗ order-service FAIL"
curl -s http://localhost:8082/actuator/health | grep -q "UP" && echo "✓ inventory-service (8082)" || echo "✗ inventory-service FAIL"
curl -s http://localhost:8083/actuator/health | grep -q "UP" && echo "✓ analytics-service (8083)" || echo "✗ analytics-service FAIL"

# 2. Crear productos
echo -e "\n2. Creando productos..."
curl -s -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop Dell XPS 15",
    "description": "Laptop profesional",
    "price": 1500.00,
    "categoryId": 1
  }' | jq '.id'

curl -s -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Mouse Logitech MX Master 3",
    "description": "Mouse ergonómico",
    "price": 99.99,
    "categoryId": 1
  }' | jq '.id'

curl -s -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Teclado Mecánico Keychron K2",
    "description": "Teclado mecánico",
    "price": 89.99,
    "categoryId": 1
  }' | jq '.id'

# 3. Crear inventario
echo -e "\n3. Creando inventario..."
curl -s -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "initialStock": 50
  }' > /dev/null

curl -s -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 2,
    "productName": "Mouse Logitech MX Master 3",
    "initialStock": 100
  }' > /dev/null

curl -s -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 3,
    "productName": "Teclado Mecánico Keychron K2",
    "initialStock": 5
  }' > /dev/null

# 4. Crear 10 órdenes (7 confirmadas, 3 canceladas)
echo -e "\n4. Creando órdenes..."

# Órdenes confirmadas (stock suficiente)
for i in {1..7}; do
  curl -s -X POST http://localhost:8081/api/orders \
    -H "Content-Type: application/json" \
    -d "{
      \"productId\": 1,
      \"quantity\": 5,
      \"customerName\": \"Cliente $i\",
      \"customerEmail\": \"cliente$i@example.com\",
      \"totalAmount\": 7500.00
    }" > /dev/null
  echo "  Orden $i creada (product 1, quantity 5)"
  sleep 1
done

# Órdenes canceladas (stock insuficiente)
for i in {8..10}; do
  curl -s -X POST http://localhost:8081/api/orders \
    -H "Content-Type: application/json" \
    -d "{
      \"productId\": 3,
      \"quantity\": 100,
      \"customerName\": \"Cliente $i\",
      \"customerEmail\": \"cliente$i@example.com\",
      \"totalAmount\": 8999.00
    }" > /dev/null
  echo "  Orden $i creada (product 3, quantity 100 - SERÁ CANCELADA)"
  sleep 1
done

# 5. Esperar propagación
echo -e "\n5. Esperando propagación de eventos (3 segundos)..."
sleep 3

# 6. Consultar analytics
echo -e "\n6. Consultando analytics..."

echo -e "\n=== Order Stats ==="
curl -s http://localhost:8083/api/analytics/orders/stats | jq '.'

echo -e "\n=== Top Products ==="
curl -s http://localhost:8083/api/analytics/products/top | jq '.'

echo -e "\n=== Low Stock (threshold=20) ==="
curl -s http://localhost:8083/api/analytics/inventory/low-stock?threshold=20 | jq '.'

echo -e "\n✓ Validación completa"
```

**Nota sobre jq**: El comando `jq '.'` formatea el JSON para mejor lectura. Si no tienes `jq` instalado:
- **Opción 1 (recomendado)**: Instalar jq → `sudo apt install jq` (Linux) o `brew install jq` (Mac)
- **Opción 2**: Remover `| jq '.'` de los comandos curl (la salida será JSON sin formato pero funcional)

**Ejecutar**:

```bash
chmod +x validate-clase7.sh
./validate-clase7.sh
```

**Salida esperada**:

```
=== Validación Clase 7: 4 Microservicios + Kafka Streams ===
1. Verificando servicios...
✓ product-service (8080)
✓ order-service (8081)
✓ inventory-service (8082)
✓ analytics-service (8083)

2. Creando productos...
1
2
3

3. Creando inventario...

4. Creando órdenes...
  Orden 1 creada (product 1, quantity 5)
  Orden 2 creada (product 1, quantity 5)
  ...
  Orden 8 creada (product 3, quantity 100 - SERÁ CANCELADA)
  ...

5. Esperando propagación de eventos (3 segundos)...

6. Consultando analytics...

=== Order Stats ===
{
  "confirmedOrders": 7,
  "cancelledOrders": 3
}

=== Top Products ===
[
  {
    "productId": 1,
    "totalSold": 35
  }
]

=== Low Stock (threshold=20) ===
[
  {
    "productId": 1,
    "availableStock": 15,
    "reservedStock": 35,
    "totalStock": 50
  },
  {
    "productId": 3,
    "availableStock": 5,
    "reservedStock": 0,
    "totalStock": 5
  }
]

✓ Validación completa
```

---

## 3. Desglose del comando

| Paso | Descripción |
|------|-------------|
| 1. Verificar servicios | Health check de 4 microservicios |
| 2. Crear productos | 3 productos en product-service |
| 3. Crear inventario | Inventario inicial para cada producto |
| 4. Crear órdenes | 7 confirmadas (product 1) + 3 canceladas (product 3) |
| 5. Esperar propagación | 3 segundos para eventos Kafka |
| 6. Consultar analytics | Validar contadores, top productos, stock bajo |

---

## 4. Explicación detallada

### 4.1 Arquitectura completa validada

```
┌─────────────────┐
│ product-service │ (8080)
│   PostgreSQL    │
└────────┬────────┘
         │ ecommerce.products.created
         ↓
    ┌────────┐
    │ Kafka  │
    └────┬───┘
         │
         ├─→ ecommerce.orders.placed ──→ inventory-service (8082)
         │                                     │
         ├─→ ecommerce.orders.confirmed ──────┤
         │                                     │
         ├─→ ecommerce.orders.cancelled       │
         │                                     │
         └─→ ecommerce.inventory.updated      │
                                               ↓
                                    ┌──────────────────┐
                                    │ analytics-service│ (8083)
                                    │   Kafka Streams  │
                                    │   State Stores   │
                                    └──────────────────┘
```

### 4.2 Flujo de eventos validado

1. **Cliente crea orden** → POST `/api/orders`
2. **order-service publica** → `ecommerce.orders.placed`
3. **inventory-service consume** → Verifica stock
4. **inventory-service publica**:
   - Stock suficiente → `ecommerce.orders.confirmed` + `ecommerce.inventory.updated`
   - Stock insuficiente → `ecommerce.orders.cancelled`
5. **analytics-service procesa**:
   - KStream → Cuenta confirmed/cancelled
   - KStream → Suma cantidades vendidas por producto
   - KTable → Sincroniza inventario actualizado
6. **Cliente consulta analytics** → GET `/api/analytics/*`

### 4.3 State stores verificados

Al final del lab, deberías tener **4 state stores**:

```bash
# Verificar en logs de analytics-service
Materialized state store order-confirmed-count-store
Materialized state store order-cancelled-count-store
Materialized state store product-sales-store
Materialized state store inventory-read-model-store
```

### 4.4 Validaciones clave

**Validación 1: Contadores de órdenes**

```bash
curl http://localhost:8083/api/analytics/orders/stats
```

- `confirmedOrders: 7` (product 1 tiene stock)
- `cancelledOrders: 3` (product 3 sin stock)

**Validación 2: Top productos**

```bash
curl http://localhost:8083/api/analytics/products/top
```

- `productId: 1, totalSold: 35` (7 órdenes × 5 unidades)

**Validación 3: Stock bajo**

```bash
curl http://localhost:8083/api/analytics/inventory/low-stock?threshold=20
```

- Product 1: 50 - 35 = 15 disponible (< 20)
- Product 3: 5 disponible (< 20, sin cambios porque órdenes canceladas)

### 4.5 Verificar logs de analytics-service

```bash
# Buscar procesamiento de eventos
grep "Processing OrderConfirmedEvent" logs/analytics-service.log
grep "Processing OrderCancelledEvent" logs/analytics-service.log

# Verificar state transition
grep "State transition" logs/analytics-service.log
```

---

## 5. Conceptos aprendidos

- **Arquitectura event-driven completa** con 4 microservices + Kafka
- **Analytics NO consulta bases de datos** - Solo Kafka + state stores
- **CQRS**: Write models (order, inventory) vs Read model (analytics)
- **Eventual consistency**: Segundos entre write y read
- **State stores son fault-tolerant** - Respaldados en Kafka
- **KStream vs KTable** aplicados en producción:
  - KStream: Contadores, agregaciones (todos los eventos)
  - KTable: Estado actual (último valor por key)
- **Interactive Queries**: REST API consulta state stores directamente
- **Real-time analytics**: Actualización automática sin polling

---

## 6. Troubleshooting

### Problema 1: Analytics no actualiza

**Síntoma**: Contadores en 0 o valores incorrectos.

**Verificar**:

1. **Streams en RUNNING**:
```bash
# Ver logs de analytics-service
grep "State transition to RUNNING" logs/analytics-service.log
```

2. **Topics existen**:
```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list
```

3. **Eventos se publican**:
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.confirmed \
  --from-beginning
```

### Problema 2: Contadores incorrectos

**Síntoma**: Números no coinciden con órdenes creadas.

**Causas posibles**:

1. **Eventos duplicados**: Reiniciaste consumer sin commit
2. **Eventos perdidos**: Kafka no estaba corriendo
3. **State store corrupto**: Borrar `/tmp/kafka-streams` y reiniciar

**Solución**:

```bash
# Limpiar state stores
rm -rf /tmp/kafka-streams/*

# Reiniciar analytics-service
# State stores se reconstruyen desde Kafka (toma tiempo)
```

### Problema 3: State store vacío después de reiniciar

**Síntoma**: Contadores en 0 después de reiniciar analytics-service.

**Causa**: State stores se reconstruyen desde Kafka changelog topic.

**Solución**: Esperar reconstrucción (puede tomar varios segundos):

```bash
# Ver progreso en logs
grep "Restoring state" logs/analytics-service.log
```

### Problema 4: Un microservicio no responde

**Verificar**:

```bash
# Health check
curl http://localhost:8080/actuator/health  # product
curl http://localhost:8081/actuator/health  # order
curl http://localhost:8082/actuator/health  # inventory
curl http://localhost:8083/actuator/health  # analytics

# Ver logs
docker compose logs -f product-service
docker compose logs -f order-service
docker compose logs -f inventory-service
# Ver logs locales de analytics-service
```

---

## 7. Desafío adicional

### Desafío 1: Dashboard de analytics

Crea un script Python/Node.js que consulte los endpoints de analytics cada 5 segundos y muestre un dashboard en terminal:

```
=== Dashboard Analytics (actualizado cada 5s) ===
Órdenes confirmadas: 15
Órdenes canceladas: 3
Top 3 productos más vendidos:
  1. Laptop Dell XPS 15 - 75 unidades
  2. Mouse Logitech - 30 unidades
  3. Teclado Mecánico - 10 unidades
Productos con stock bajo (<20):
  - Laptop Dell XPS 15: 5 disponibles
```

### Desafío 2: Stress test

Crea 100 órdenes concurrentes y verifica que analytics se actualiza correctamente:

```bash
for i in {1..100}; do
  curl -X POST http://localhost:8081/api/orders \
    -H "Content-Type: application/json" \
    -d "{...}" &
done
wait
sleep 5
curl http://localhost:8083/api/analytics/orders/stats
```

### Desafío 3: Verificación de eventual consistency

Mide el tiempo entre crear orden y aparecer en analytics:

1. Crear orden → Capturar timestamp
2. Consultar analytics cada 100ms
3. Cuando aparece → Calcular latencia
4. Promedio de 10 órdenes

---

## 8. Recursos adicionales

### Checklist completo Clase 7

**analytics-service**:
- [ ] Servicio corre en puerto 8083
- [ ] Streams en estado RUNNING
- [ ] 4 state stores creados:
  - order-confirmed-count-store
  - order-cancelled-count-store
  - product-sales-store
  - inventory-read-model-store

**Kafka Topics**:
- [ ] 5 topics activos:
  - ecommerce.products.created
  - ecommerce.orders.placed
  - ecommerce.orders.confirmed
  - ecommerce.orders.cancelled
  - ecommerce.inventory.updated

**Endpoints Analytics**:
- [ ] GET `/api/analytics/orders/stats` - Contadores
- [ ] GET `/api/analytics/products/top` - Top productos
- [ ] GET `/api/analytics/inventory/low-stock?threshold=10` - Stock bajo

**Conceptos implementados**:
- [ ] KStream (órdenes confirmadas/canceladas, ventas por producto)
- [ ] KTable (inventario)
- [ ] State stores (KeyValueStore)
- [ ] Interactive Queries (REST API)
- [ ] CQRS (read model separado)
- [ ] Operaciones stateful (count, reduce)
- [ ] Eventual consistency

### Documentación

- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [Spring Kafka Streams](https://docs.spring.io/spring-kafka/reference/html/#kafka-streams)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Event-Driven Architectures](https://martinfowler.com/articles/201701-event-driven.html)

---

**¡Felicidades!** Has completado la implementación de un sistema de analytics en tiempo real con Kafka Streams y el patrón CQRS.

**Próximo**: Clase 8 - Production readiness (Profiles, Actuator, JWT Security)
