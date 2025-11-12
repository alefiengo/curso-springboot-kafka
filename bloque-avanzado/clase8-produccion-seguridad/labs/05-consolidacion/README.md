# Lab 05: Consolidación Final

---

## 1. Objetivo

Realizar una revisión completa de la arquitectura de microservicios, verificar que todos los componentes estén correctamente configurados y funcionales, y prepararse para el proyecto integrador mediante un test end-to-end del sistema completo.

---

## 2. Comandos a ejecutar

```bash
# Script de verificación completa del sistema
cd ~/workspace

# 1. Verificar infraestructura
docker compose ps
docker exec -it postgres psql -U postgres -c "SELECT version();"
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# 2. Health checks de todos los servicios
curl http://localhost:8080/actuator/health  # product-service
curl http://localhost:8081/actuator/health  # order-service
curl http://localhost:8082/actuator/health  # inventory-service
curl http://localhost:8083/actuator/health  # analytics-service

# 3. Test end-to-end completo

# Login (obtener JWT)
TOKEN=$(curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  -s | jq -r '.token')

echo "Token obtenido: $TOKEN"

# Crear producto (con JWT)
PRODUCT_RESPONSE=$(curl -X POST http://localhost:8080/api/products \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop Dell XPS 15",
    "description": "Laptop profesional de alto rendimiento",
    "price": 1500.00,
    "categoryId": 1
  }' -s)

PRODUCT_ID=$(echo $PRODUCT_RESPONSE | jq -r '.id')
echo "Producto creado: ID=$PRODUCT_ID"

# Crear inventario
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d "{
    \"productId\": $PRODUCT_ID,
    \"productName\": \"Laptop Dell XPS 15\",
    \"initialStock\": 50
  }"

echo "Inventario creado: stock=50"

# Esperar propagación de eventos
sleep 2

# Crear orden
ORDER_RESPONSE=$(curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d "{
    \"productId\": $PRODUCT_ID,
    \"quantity\": 5,
    \"customerName\": \"Juan Pérez\",
    \"customerEmail\": \"juan.perez@example.com\",
    \"totalAmount\": 7500.00
  }" -s)

ORDER_ID=$(echo $ORDER_RESPONSE | jq -r '.id')
echo "Orden creada: ID=$ORDER_ID"

# Esperar procesamiento asíncrono
sleep 3

# Verificar orden procesada
curl http://localhost:8081/api/orders/$ORDER_ID | jq

# Verificar inventario actualizado
curl http://localhost:8082/api/inventory/product/$PRODUCT_ID | jq

# Verificar analytics
curl http://localhost:8083/api/analytics/orders/stats | jq
curl http://localhost:8083/api/analytics/products/top | jq

# Ver logs de eventos Kafka
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning \
  --max-messages 5
```

**Salida esperada**:

```json
{
  "status": "UP",
  "components": {
    "db": {"status": "UP"},
    "kafka": {"status": "UP"},
    "diskSpace": {"status": "UP"}
  }
}
```

---

## 3. Desglose del comando

| Elemento | Descripción |
|----------|-------------|
| `docker compose ps` | Verificar servicios de infraestructura corriendo |
| `curl .../actuator/health` | Health check de cada microservicio |
| `jq -r '.token'` | Extraer token JWT del response JSON |
| `sleep 3` | Esperar procesamiento asíncrono de eventos |
| `kafka-console-consumer` | Verificar eventos publicados en Kafka |
| `--from-beginning` | Leer todos los mensajes del topic desde el inicio |

---

## 4. Explicación detallada

### 4.1 Arquitectura Completa del Sistema

```
┌─────────────────────────────────────────────────────────────────┐
│                        INFRAESTRUCTURA                          │
├─────────────────────────────────────────────────────────────────┤
│  PostgreSQL (5432)  │  Zookeeper (2181)  │  Kafka (9092)       │
└─────────────────────────────────────────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────▼────────┐     ┌────────▼────────┐     ┌────────▼────────┐
│ product-service│     │  order-service  │     │inventory-service│
│   (8080)       │     │    (8081)       │     │    (8082)       │
│ + JWT Security │────▶│  + Kafka Saga   │────▶│ + Event Handler │
└────────┬───────┘     └────────┬────────┘     └────────┬────────┘
         │                      │                       │
         └──────────────────────┼───────────────────────┘
                                │
                      ┌─────────▼─────────┐
                      │ analytics-service │
                      │     (8083)        │
                      │  + Kafka Streams  │
                      │  + CQRS Read Model│
                      └───────────────────┘
```

### 4.2 Topics de Kafka Activos

| Topic | Producer | Consumer | Propósito |
|-------|----------|----------|-----------|
| `ecommerce.products.created` | product-service | analytics-service | Notificar creación de productos |
| `ecommerce.orders.placed` | order-service | inventory-service, analytics-service | Procesar nuevas órdenes |
| `ecommerce.inventory.updated` | inventory-service | order-service | Confirmar actualización de stock |
| `ecommerce.orders.confirmed` | order-service | analytics-service | Órdenes confirmadas |
| `ecommerce.orders.rejected` | order-service | analytics-service | Órdenes rechazadas (stock insuficiente) |

### 4.3 Checklist de Verificación del Sistema

#### Infraestructura

- [ ] PostgreSQL corriendo y accesible
- [ ] Kafka corriendo con Zookeeper
- [ ] 5 topics creados en Kafka
- [ ] Bases de datos creadas (ecommerce, ecommerce_orders, ecommerce_inventory)

**Comandos de verificación**:

```bash
# PostgreSQL
docker exec -it postgres psql -U postgres -l

# Kafka topics
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# Bases de datos
docker exec -it postgres psql -U postgres -c "\l" | grep ecommerce
```

#### Microservicios

- [ ] product-service (8080) - Status UP, DB UP, JWT configurado
- [ ] order-service (8081) - Status UP, DB UP, Kafka UP
- [ ] inventory-service (8082) - Status UP, DB UP, Kafka UP
- [ ] analytics-service (8083) - Status UP, Kafka Streams corriendo

**Comandos de verificación**:

```bash
# Health checks
for port in 8080 8081 8082 8083; do
  echo "=== Service on port $port ==="
  curl -s http://localhost:$port/actuator/health | jq '.status'
done
```

#### Configuración de Producción

- [ ] Profiles configurados (dev/prod) en todos los servicios
- [ ] Actuator habilitado con health, info, metrics
- [ ] Variables de entorno con fallbacks
- [ ] JWT implementado en product-service
- [ ] Logging estructurado con Logback

**Comandos de verificación**:

```bash
# Verificar perfil activo
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.name | contains("applicationConfig"))'

# Verificar endpoints de Actuator
curl http://localhost:8080/actuator | jq '.["_links"] | keys'
```

### 4.4 Flujo End-to-End Explicado

#### Paso 1: Autenticación

```bash
POST /api/auth/login
{
  "username": "admin",
  "password": "admin123"
}

Response:
{
  "token": "eyJhbGciOiJIUzUxMiJ9..."
}
```

**Flujo interno**:
1. AuthController recibe credenciales
2. Verifica username/password con BCrypt
3. JwtUtil genera token firmado
4. Token tiene validez de 24 horas

#### Paso 2: Crear Producto (con JWT)

```bash
POST /api/products
Authorization: Bearer <token>
{
  "name": "Laptop Dell XPS 15",
  "price": 1500.00
}
```

**Flujo interno**:
1. JwtAuthenticationFilter valida token
2. ProductController recibe request
3. ProductService guarda en BD
4. ProductEventProducer publica `ecommerce.products.created`
5. analytics-service consume evento (actualiza contadores)

#### Paso 3: Crear Inventario

```bash
POST /api/inventory
{
  "productId": 1,
  "initialStock": 50
}
```

**Flujo interno**:
1. InventoryController recibe request
2. InventoryService crea registro en BD
3. Stock inicial: 50 unidades

#### Paso 4: Crear Orden

```bash
POST /api/orders
{
  "productId": 1,
  "quantity": 5,
  "totalAmount": 7500.00
}
```

**Flujo interno (Saga Pattern)**:

1. **order-service**:
   - Crea orden con status PENDING
   - Publica `ecommerce.orders.placed`

2. **inventory-service** (asíncrono):
   - Consume `ecommerce.orders.placed`
   - Verifica stock disponible (50 >= 5) ✓
   - Reduce stock: 50 - 5 = 45
   - Publica `ecommerce.inventory.updated`

3. **order-service** (asíncrono):
   - Consume `ecommerce.inventory.updated`
   - Actualiza orden a CONFIRMED
   - Publica `ecommerce.orders.confirmed`

4. **analytics-service** (asíncrono):
   - Consume todos los eventos
   - Actualiza contadores en Kafka Streams
   - Incrementa totalOrders, totalRevenue

#### Paso 5: Verificar Analytics

```bash
GET /api/analytics/orders/stats

Response:
{
  "totalOrders": 1,
  "totalRevenue": 7500.00,
  "averageOrderValue": 7500.00
}
```

### 4.5 Escenarios de Prueba

#### Escenario 1: Orden exitosa (stock suficiente)

```bash
# Stock inicial: 50
# Cantidad pedida: 5
# Resultado esperado: CONFIRMED, stock final = 45

ORDER_ID=$(curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":5,"customerName":"Juan","customerEmail":"juan@example.com","totalAmount":7500.00}' \
  -s | jq -r '.id')

sleep 3

curl http://localhost:8081/api/orders/$ORDER_ID | jq '.status'
# Output: "CONFIRMED"

curl http://localhost:8082/api/inventory/product/1 | jq '.stock'
# Output: 45
```

#### Escenario 2: Orden rechazada (stock insuficiente)

```bash
# Stock actual: 45
# Cantidad pedida: 100
# Resultado esperado: REJECTED, stock sin cambios

ORDER_ID=$(curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":100,"customerName":"Maria","customerEmail":"maria@example.com","totalAmount":150000.00}' \
  -s | jq -r '.id')

sleep 3

curl http://localhost:8081/api/orders/$ORDER_ID | jq '.status'
# Output: "REJECTED"

curl http://localhost:8082/api/inventory/product/1 | jq '.stock'
# Output: 45 (sin cambios)
```

#### Escenario 3: Producto sin autenticación (debe fallar)

```bash
curl http://localhost:8080/api/products
# Output: 403 Forbidden

curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","price":100}'
# Output: 403 Forbidden
```

### 4.6 Monitoreo de Logs

#### Ver logs en tiempo real (terminal separadas)

**Terminal 1 - product-service**:
```bash
tail -f ~/workspace/product-service/logs/application.log
```

**Terminal 2 - order-service**:
```bash
tail -f ~/workspace/order-service/logs/application.log
```

**Terminal 3 - inventory-service**:
```bash
tail -f ~/workspace/inventory-service/logs/application.log
```

**Terminal 4 - Kafka events**:
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning \
  --property print.key=true
```

### 4.7 Script de Inicio Completo

Crear `start-all-services.sh`:

```bash
#!/bin/bash

echo "===== Iniciando Sistema E-commerce ====="

# 1. Infraestructura
echo "[1/5] Iniciando infraestructura (PostgreSQL, Kafka)..."
docker compose up -d
sleep 10

# 2. Verificar infraestructura
echo "[2/5] Verificando infraestructura..."
docker exec -it postgres psql -U postgres -c "SELECT 1" > /dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "✓ PostgreSQL OK"
else
  echo "✗ PostgreSQL ERROR"
  exit 1
fi

# 3. Crear topics de Kafka
echo "[3/5] Creando topics de Kafka..."
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create --if-not-exists --topic ecommerce.products.created --partitions 3 --replication-factor 1

docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create --if-not-exists --topic ecommerce.orders.placed --partitions 3 --replication-factor 1

docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create --if-not-exists --topic ecommerce.inventory.updated --partitions 3 --replication-factor 1

docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create --if-not-exists --topic ecommerce.orders.confirmed --partitions 3 --replication-factor 1

docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create --if-not-exists --topic ecommerce.orders.rejected --partitions 3 --replication-factor 1

# 4. Iniciar microservicios
echo "[4/5] Iniciando microservicios..."

cd ~/workspace/product-service
mvn spring-boot:run -Dspring-boot.run.profiles=dev > /dev/null 2>&1 &
echo "✓ product-service iniciado (8080)"

cd ~/workspace/order-service
mvn spring-boot:run -Dspring-boot.run.profiles=dev > /dev/null 2>&1 &
echo "✓ order-service iniciado (8081)"

cd ~/workspace/inventory-service
mvn spring-boot:run -Dspring-boot.run.profiles=dev > /dev/null 2>&1 &
echo "✓ inventory-service iniciado (8082)"

cd ~/workspace/analytics-service
mvn spring-boot:run -Dspring-boot.run.profiles=dev > /dev/null 2>&1 &
echo "✓ analytics-service iniciado (8083)"

# 5. Esperar inicialización
echo "[5/5] Esperando inicialización completa..."
sleep 30

# Health checks
echo ""
echo "===== Health Checks ====="
for port in 8080 8081 8082 8083; do
  status=$(curl -s http://localhost:$port/actuator/health | jq -r '.status')
  echo "Port $port: $status"
done

echo ""
echo "===== Sistema Iniciado ====="
echo "product-service:    http://localhost:8080"
echo "order-service:      http://localhost:8081"
echo "inventory-service:  http://localhost:8082"
echo "analytics-service:  http://localhost:8083"
```

Ejecutar:
```bash
chmod +x start-all-services.sh
./start-all-services.sh
```

---

## 5. Conceptos aprendidos

- **Arquitectura de microservicios completa**: 4 servicios comunicándose vía Kafka
- **Event-driven architecture**: Comunicación asíncrona con eventos
- **Saga pattern**: Transacciones distribuidas con compensación
- **CQRS**: Read model separado (analytics-service)
- **JWT Security**: Protección de endpoints REST
- **Spring Boot Profiles**: Configuración por entorno
- **Health checks**: Monitoreo con Actuator
- **Logging estructurado**: Trazabilidad y debugging
- **Variables de entorno**: Externalización de configuración
- **End-to-end testing**: Validación completa del flujo

---

## 6. Troubleshooting

### Problema 1: Algún servicio no inicia

**Síntoma**: Health check devuelve error de conexión.

**Solución**: Verificar logs:

```bash
# Ver logs del servicio
tail -f ~/workspace/product-service/logs/application.log

# Buscar errores
grep ERROR ~/workspace/product-service/logs/application.log
```

### Problema 2: Orden no se confirma

**Síntoma**: Status permanece en PENDING después de varios minutos.

**Causa**: inventory-service no está consumiendo eventos.

**Solución**:

```bash
# Verificar que inventory-service está UP
curl http://localhost:8082/actuator/health

# Verificar logs de Kafka consumer
tail -f ~/workspace/inventory-service/logs/application.log | grep "order placed"

# Verificar consumer group
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe
```

### Problema 3: Analytics no refleja datos

**Síntoma**: `/api/analytics/orders/stats` devuelve ceros.

**Causa**: analytics-service no está procesando Kafka Streams.

**Solución**:

```bash
# Verificar logs de Kafka Streams
tail -f ~/workspace/analytics-service/logs/application.log | grep "stream"

# Reiniciar analytics-service
cd ~/workspace/analytics-service
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

### Problema 4: JWT token expiró

**Error**: `401 Unauthorized` al usar token.

**Solución**: Generar nuevo token:

```bash
TOKEN=$(curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  -s | jq -r '.token')
```

---

## 7. Desafío adicional

1. **Crea un dashboard de monitoreo** con todos los health checks:

```bash
#!/bin/bash
# monitor-dashboard.sh

while true; do
  clear
  echo "===== E-commerce System Dashboard ====="
  echo "$(date)"
  echo ""

  for port in 8080 8081 8082 8083; do
    status=$(curl -s http://localhost:$port/actuator/health | jq -r '.status')
    if [ "$status" == "UP" ]; then
      echo "✓ Port $port: UP"
    else
      echo "✗ Port $port: DOWN"
    fi
  done

  echo ""
  echo "===== Analytics ====="
  curl -s http://localhost:8083/api/analytics/orders/stats | jq

  sleep 5
done
```

2. **Implementa script de pruebas de carga**:

```bash
#!/bin/bash
# load-test.sh

TOKEN=$(curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  -s | jq -r '.token')

for i in {1..100}; do
  curl -X POST http://localhost:8081/api/orders \
    -H "Content-Type: application/json" \
    -d "{
      \"productId\": 1,
      \"quantity\": 1,
      \"customerName\": \"Customer $i\",
      \"customerEmail\": \"customer$i@example.com\",
      \"totalAmount\": 1500.00
    }" &
done

wait
echo "100 órdenes creadas"
```

3. **Documenta la arquitectura completa** en README.md del proyecto:

```markdown
# E-commerce Microservices Platform

## Arquitectura

- product-service: Gestión de productos con JWT
- order-service: Gestión de órdenes con Saga pattern
- inventory-service: Control de inventario con eventos
- analytics-service: Kafka Streams + CQRS

## Tecnologías

- Spring Boot 3.x
- Apache Kafka
- PostgreSQL
- JWT Security
- Docker Compose

## Iniciar Sistema

```bash
./start-all-services.sh
```

## Endpoints Principales

...
```

---

## 8. Recursos adicionales

- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/index.html)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)

---

**Siguiente**: Preparar propuesta de [Proyecto Integrador](../../../../PROYECTO_INTEGRADOR.md)
