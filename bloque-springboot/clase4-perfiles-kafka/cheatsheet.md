# Cheatsheet - Clase 4: Perfiles, Configuración y Apache Kafka

Referencia rápida de comandos, configuraciones y conceptos clave de la Clase 4.

---

## Spring Boot Profiles

### Estructura de archivos

```
src/main/resources/
├── application.yml           # Configuración común
├── application-dev.yml       # Desarrollo
└── application-prod.yml      # Producción
```

### Activar perfiles

```bash
# Maven
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# JAR
java -jar app.jar --spring.profiles.active=prod

# Variable de entorno
export SPRING_PROFILES_ACTIVE=dev
java -jar app.jar
```

### Configuración por perfil

**application-dev.yml**:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    dev.alefiengo: DEBUG
```

**application-prod.yml**:

```yaml
spring:
  jpa:
    show-sql: false

logging:
  level:
    dev.alefiengo: INFO
```

---

## Spring Boot Actuator

### Dependencia

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Configuración

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env
  endpoint:
    health:
      show-details: always
```

### Endpoints principales

```bash
# Health check
curl http://localhost:8080/actuator/health

# Información de la aplicación
curl http://localhost:8080/actuator/info

# Lista de métricas
curl http://localhost:8080/actuator/metrics

# Métrica específica
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Requests HTTP
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/products"
```

### Cambiar nivel de log en tiempo real

```bash
# Ver nivel actual
curl http://localhost:8080/actuator/loggers/dev.alefiengo

# Cambiar a DEBUG
curl -X POST http://localhost:8080/actuator/loggers/dev.alefiengo \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

---

## Variables de Entorno

### Sintaxis en application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}

server:
  port: ${SERVER_PORT:8080}
```

**Formato**: `${VARIABLE:valor_por_defecto}`

### Definir variables

**Linux/Mac**:

```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=ecommerce_dev
export DB_USER=postgres
export DB_PASSWORD=postgres
```

**Windows PowerShell**:

```powershell
$env:DB_HOST="localhost"
$env:DB_PORT="5432"
$env:DB_NAME="ecommerce_dev"
```

### Archivo .env (desarrollo)

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce_dev
DB_USER=postgres
DB_PASSWORD=postgres
SPRING_PROFILES_ACTIVE=dev
```

**IMPORTANTE**: Agregar `.env` a `.gitignore`.

### Docker Compose

```yaml
services:
  product-service:
    image: product-service:latest
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: product_db
      SPRING_PROFILES_ACTIVE: prod
```

---

## Docker Compose - Kafka

### docker-compose.yml

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data

  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  zookeeper-data:
  kafka-data:
```

**IMPORTANTE**: NO agregar `version:` field (deprecated en Docker Compose V2).

### Comandos Docker Compose

```bash
# Iniciar servicios
docker compose up -d

# Ver estado
docker compose ps

# Ver logs
docker compose logs kafka
docker compose logs -f kafka  # Seguir logs

# Detener servicios (mantiene datos)
docker compose stop

# Iniciar servicios detenidos
docker compose start

# Detener y eliminar (mantiene volúmenes)
docker compose down

# Limpieza completa (elimina volúmenes)
docker compose down -v
```

---

## Kafka CLI

### Acceso al contenedor

```bash
docker exec -it kafka bash
```

### Gestión de topics

```bash
# Listar topics
kafka-topics --bootstrap-server localhost:9092 --list

# Crear topic
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.placed \
  --partitions 3 \
  --replication-factor 1

# Describir topic
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.orders.placed

# Aumentar particiones
kafka-topics --bootstrap-server localhost:9092 \
  --alter \
  --topic ecommerce.orders.placed \
  --partitions 5

# Eliminar topic
kafka-topics --bootstrap-server localhost:9092 \
  --delete \
  --topic ecommerce.orders.placed
```

### Configuración de topics

```bash
# Ver configuración
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --describe

# Cambiar retención (24 horas)
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --alter \
  --add-config retention.ms=86400000

# Eliminar configuración (volver a default)
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --alter \
  --delete-config retention.ms
```

### Productor de consola

```bash
# Enviar mensajes manualmente
kafka-console-producer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed

# Con clave (key:value)
kafka-console-producer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --property "parse.key=true" \
  --property "key.separator=:"
```

### Consumidor de consola

```bash
# Consumir desde el final
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed

# Consumir desde el inicio
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning

# Consumir con clave
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning \
  --property print.key=true
```

### Consumer groups

```bash
# Listar consumer groups
kafka-consumer-groups --bootstrap-server localhost:9092 --list

# Describir consumer group
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe

# Reset offset (CUIDADO: reprocesa mensajes)
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group inventory-service \
  --topic ecommerce.orders.placed \
  --reset-offsets \
  --to-earliest \
  --execute
```

---

## Topics del dominio e-commerce

| Topic | Productor | Consumidores | Propósito |
|-------|-----------|--------------|-----------|
| `ecommerce.products.created` | product-service | analytics-service | Nuevos productos |
| `ecommerce.products.updated` | product-service | analytics-service | Actualización productos |
| `ecommerce.orders.placed` | order-service | inventory-service, analytics | Nueva orden |
| `ecommerce.orders.confirmed` | inventory-service | order-service | Orden confirmada |
| `ecommerce.orders.cancelled` | inventory-service | order-service | Orden cancelada |
| `ecommerce.inventory.updated` | inventory-service | analytics-service | Actualización stock |

### Convención de nombres

**Formato**: `app-name.domain.event-type`

Ejemplos:

- `ecommerce.orders.placed` (pasado)
- `ecommerce.products.created` (pasado)
- `ecommerce.inventory.updated` (pasado)

**NO usar**:

- ❌ `orders` (sin contexto)
- ❌ `PlaceOrder` (comando, no evento)
- ❌ `my-topic` (genérico)

---

## Configuraciones importantes

### Retención de mensajes

```yaml
# Kafka topic config
retention.ms=604800000  # 7 días (default)
retention.ms=86400000   # 24 horas
retention.ms=-1         # Ilimitado
```

### Replication factor

**Desarrollo** (1 broker):

```bash
--replication-factor 1
```

**Producción** (3+ brokers):

```bash
--replication-factor 3
```

### Particiones

**Regla general**: `partitions ≥ número máximo de consumidores`

**Cálculo por throughput**:

```
partitions = throughput_requerido / throughput_por_consumidor
```

**En este curso**: 3 particiones por topic

---

## Troubleshooting común

### Problema: Kafka no inicia

```bash
# Ver logs
docker compose logs kafka

# Verificar Zookeeper
docker compose logs zookeeper

# Reiniciar
docker compose restart kafka
```

### Problema: Connection refused

```bash
# Verificar que Kafka está corriendo
docker compose ps

# Verificar puerto
docker exec -it kafka kafka-broker-api-versions --bootstrap-server localhost:9092
```

### Problema: Topic no se crea

```bash
# Verificar replication-factor
# Con 1 broker, usar replication-factor 1

kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic my-topic \
  --partitions 3 \
  --replication-factor 1
```

### Problema: Profile no se activa

```bash
# Verificar sintaxis
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Para JAR usar:
java -jar app.jar --spring.profiles.active=dev
```

---

## Diferencias clave

### docker compose vs docker-compose

```bash
# ✅ CORRECTO (Docker Compose V2)
docker compose up -d

# ❌ INCORRECTO (Docker Compose V1, deprecated)
docker-compose up -d
```

### Eventos vs Comandos

```java
// ✅ CORRECTO (Eventos - pasado)
public record OrderPlacedEvent(String orderId) {}
public record ProductCreatedEvent(Long productId) {}

// ❌ INCORRECTO (Comandos - imperativo)
public record PlaceOrderCommand(String orderId) {}
public record CreateProductCommand(Long productId) {}
```

### show-sql por perfil

```yaml
# ✅ CORRECTO: Configurar en cada perfil
# application-dev.yml
spring:
  jpa:
    show-sql: true

# application-prod.yml
spring:
  jpa:
    show-sql: false

# ❌ INCORRECTO: Hardcodear en application.yml
# application.yml
spring:
  jpa:
    show-sql: true  # NO hacer esto
```

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Perfil | Profile | Conjunto de configuraciones por entorno |
| Variable de entorno | Environment variable | Configuración externalizada |
| Punto de acceso | Endpoint | URL HTTP para funcionalidad |
| Observabilidad | Observability | Capacidad de monitorear estado de app |
| Broker | Broker | Servidor Kafka que almacena mensajes |
| Tópico | Topic | Canal lógico de mensajes |
| Partición | Partition | División de topic para paralelismo |
| Productor | Producer | Aplicación que publica mensajes |
| Consumidor | Consumer | Aplicación que lee mensajes |
| Grupo de consumidores | Consumer Group | Conjunto de consumidores que procesan topic |
| Desplazamiento | Offset | Identificador único de mensaje |
| Evento | Event | Notificación de algo que ocurrió |
| Arquitectura event-driven | Event-Driven Architecture | Comunicación basada en eventos |
| Consistencia eventual | Eventual Consistency | Datos consistentes después de tiempo |
| Retención | Retention | Tiempo de almacenamiento de mensajes |
| Réplica | Replica | Copia de datos en otro broker |
| Líder | Leader | Broker principal de una partición |

---

**Última actualización**: Noviembre 2025
**Curso**: Spring Boot & Apache Kafka - i-Quattro
