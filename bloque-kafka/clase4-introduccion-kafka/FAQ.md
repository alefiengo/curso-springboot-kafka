# FAQ - Clase 4: Introducción a Apache Kafka

Respuestas a preguntas frecuentes sobre Apache Kafka, su arquitectura y uso con Docker.

---

## Docker Compose

### ¿Debo agregar version: '3.8' en docker-compose.yml?

**NO**. El campo `version` está deprecated en Docker Compose V2.

```yaml
# ✅ CORRECTO
services:
  kafka:
    image: confluentinc/cp-kafka:latest

# ❌ INCORRECTO
version: '3.8'  # NO agregar esto
services:
  kafka:
    image: confluentinc/cp-kafka:latest
```

### ¿Cuál es la diferencia entre docker compose y docker-compose?

| Comando | Versión | Estado |
|---------|---------|--------|
| `docker compose` (espacio) | V2 | Actual, integrado en Docker Engine |
| `docker-compose` (hyphen) | V1 | Deprecated, binario separado |

**Usar siempre**: `docker compose` (V2)

### ¿Por qué Kafka tarda en iniciar en modo KRaft?

**Síntoma**:

```
Waiting for controller quorum to be initialized
```

**Causa**:

Kafka en modo KRaft necesita formatear el directorio de logs en el primer inicio.

**Solución**: Esperar 30-60 segundos. Verificar logs:

```bash
docker compose logs -f kafka
# Buscar: "[KafkaRaftServer] Kafka Server started"
```

Si persiste el problema, formatear manualmente:

```bash
docker compose down -v
docker compose up -d
```

### ¿Cómo persisten los datos de Kafka entre reinicios?

Los volúmenes Docker garantizan persistencia:

```yaml
volumes:
  kafka-data:/var/lib/kafka/data
```

**Comandos**:

- `docker compose down` → Detiene contenedores, MANTIENE volúmenes
- `docker compose down -v` → Detiene y ELIMINA volúmenes (limpieza completa)

### ¿Por qué mi aplicación Spring Boot no se conecta a Kafka?

**Problema común**: `KAFKA_ADVERTISED_LISTENERS` incorrecto.

**Escenario 1: Spring Boot en host (NO en Docker)**:

```yaml
# docker-compose.yml
environment:
  KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092

# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
```

**Escenario 2: Spring Boot en Docker**:

```yaml
# docker-compose.yml
environment:
  KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

# application.yml
spring:
  kafka:
    bootstrap-servers: kafka:9092
```

---

## Apache Kafka

### ¿Por qué debo crear topics manualmente? ¿No se crean automáticamente?

Deshabilitamos auto-creación con:

```yaml
KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
```

**Ventajas de creación manual**:

- Control de particiones y replication factor
- Evita topics accidentales por typos
- Configuración explícita (retención, compresión)
- Buena práctica en producción

**Si habilitas auto-creación**:

- Topics se crean con 1 partición y replication factor 1 (subóptimo)
- No tienes control sobre configuración

### ¿Cuántas particiones debo crear por topic?

**Regla general**:

```
partitions ≥ número máximo de consumidores paralelos
```

**Cálculo por throughput**:

```
partitions = throughput_requerido / throughput_por_consumidor
```

**Ejemplo**:

- Throughput requerido: 10,000 msg/s
- Consumidor procesa: 500 msg/s
- Particiones: 10,000 / 500 = 20

**En este curso**: 3 particiones (balance para desarrollo)

### ¿Puedo reducir el número de particiones de un topic?

**NO**. Kafka NO permite reducir particiones.

**Opciones**:

1. Mantener particiones actuales (recomendado)
2. Eliminar y recrear topic (pierdes todos los mensajes)

```bash
kafka-topics --bootstrap-server localhost:9092 --delete --topic my-topic
kafka-topics --bootstrap-server localhost:9092 --create --topic my-topic --partitions 3 --replication-factor 1
```

### ¿Qué significa replication-factor y por qué es importante?

**Replication factor**: Número de copias de cada partición en diferentes brokers.

**Ejemplo con replication-factor 3**:

```
Partition 0: [Broker 1 (Leader), Broker 2 (Replica), Broker 3 (Replica)]
```

**Ventajas**:

- **Alta disponibilidad**: Si un broker cae, otro asume como líder
- **Tolerancia a fallos**: Datos NO se pierden
- **Durabilidad**: Garantiza que mensajes persisten

**Desventajas**:

- Mayor uso de almacenamiento (3x)
- Mayor latencia de escritura (debe replicarse)

**Configuración**:

- **Desarrollo** (1 broker): `--replication-factor 1`
- **Producción** (3+ brokers): `--replication-factor 3`

### ¿Qué es KRaft y por qué no usamos Zookeeper?

**KRaft** (Kafka Raft Metadata): Modo de operación de Kafka sin dependencia de Zookeeper.

| Característica | Descripción |
|----------------|-------------|
| **Dependencia** | Integrado en Kafka (sin procesos externos) |
| **Complejidad** | Menor (un solo sistema) |
| **Estabilidad** | Estable desde Kafka 3.3, producción-ready desde 3.6 |
| **Metadata** | Almacenada en quorum de Kafka |
| **Performance** | Mejor latencia para operaciones de metadata |

**En este curso**: Usamos KRaft por ser el estándar actual de Kafka.

**Nota**: Zookeeper fue deprecated en Kafka 3.5 y será removido en Kafka 4.0.

### ¿Cómo funciona el consumer group rebalancing?

**Rebalancing**: Redistribución de particiones entre consumidores del grupo.

**Cuándo ocurre**:

- Nuevo consumidor se une al grupo
- Consumidor existente sale del grupo (falla, se apaga)
- Nuevo topic se agrega al grupo
- Particiones se agregan a topic

**Proceso**:

1. Kafka detecta cambio en el grupo
2. Pausa todos los consumidores del grupo
3. Redistribuye particiones
4. Consumidores reanudan procesamiento

**Ejemplo**:

```
Antes: 3 particiones, 2 consumidores
- Consumidor A: Partitions 0, 1
- Consumidor B: Partition 2

Nuevo consumidor C se une
↓ Rebalancing ↓

Después: 3 particiones, 3 consumidores
- Consumidor A: Partition 0
- Consumidor B: Partition 1
- Consumidor C: Partition 2
```

### ¿Qué pasa si tengo más consumidores que particiones?

**Ejemplo**: 3 particiones, 5 consumidores

```
- Consumidor A: Partition 0
- Consumidor B: Partition 1
- Consumidor C: Partition 2
- Consumidor D: Inactivo (idle)
- Consumidor E: Inactivo (idle)
```

Consumidores D y E NO procesan mensajes, pero siguen activos por si A, B o C fallan.

### ¿Cómo garantizo el orden de mensajes?

**Orden garantizado**:

- **Dentro de una partición**: Sí, siempre
- **Entre particiones**: No, no hay garantía

**Estrategia**: Usar partition key

```java
// Mensajes con el mismo orderId van a la misma partición
kafkaTemplate.send("ecommerce.orders.placed", orderId, orderEvent);
//                  topic                      key (orderId)
```

Todos los eventos de un pedido van a la misma partición → orden garantizado.

### ¿Kafka almacena mensajes en memoria o disco?

**Disco**. Kafka persiste todos los mensajes en disco (logs).

**Ventajas**:

- Durabilidad (no se pierden si broker reinicia)
- Replay (puedes releer mensajes antiguos)
- Escalabilidad (no limitado por RAM)

**Rendimiento**: Kafka usa OS page cache → acceso rápido a mensajes recientes.

### ¿Cuánto tiempo se retienen los mensajes en Kafka?

Por defecto: **7 días** (`retention.ms=604800000`)

**Configurar retención**:

```bash
# 24 horas
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config retention.ms=86400000

# Ilimitado
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config retention.ms=-1
```

**Por tamaño**:

```bash
# Máximo 1 GB por partición
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config retention.bytes=1073741824
```

---

## Conceptos event-driven

### ¿Cuál es la diferencia entre un evento y un comando?

| Tipo | Tiempo verbal | Ejemplo | Semántica |
|------|---------------|---------|-----------|
| **Evento** | Pasado | `OrderPlacedEvent` | "Algo ocurrió" (notificación) |
| **Comando** | Imperativo | `PlaceOrderCommand` | "Haz esto" (solicitud) |

**En arquitectura event-driven**: Usamos eventos (past tense).

### ¿Qué es eventual consistency?

**Consistencia eventual**: Los datos NO son consistentes inmediatamente, pero lo serán después de un tiempo.

**Ejemplo**:

```
t=0ms : Cliente crea orden → order-service guarda (status=PENDING)
t=10ms: order-service publica OrderPlacedEvent
t=20ms: inventory-service recibe evento → valida stock
t=30ms: inventory-service publica OrderConfirmedEvent
t=40ms: order-service actualiza status=CONFIRMED
```

Durante 40ms, el cliente ve `PENDING` aunque la orden será confirmada.

**Implicaciones**:

- UI debe mostrar estados intermedios
- NO esperar respuesta síncrona
- Diseñar para tolerancia a inconsistencias temporales

### ¿Cuándo usar Kafka vs REST API directa?

**Usar Kafka cuando**:

- Comunicación asíncrona aceptable
- Necesitas desacoplamiento
- Alta disponibilidad crítica
- Auditoría completa de eventos
- Múltiples consumidores del mismo evento

**Usar REST cuando**:

- Necesitas respuesta inmediata
- Operación síncrona requerida
- Cliente externo sin Kafka
- Query simple (no modificación de estado)

**Ejemplo**:

- ✅ Crear orden → Kafka (puede ser asíncrono)
- ✅ Consultar productos → REST (necesita respuesta inmediata)

---

## Troubleshooting

### Error: "Replication factor: 3 larger than available brokers: 1"

**Causa**: Solo hay 1 broker, pero intentas crear topic con `--replication-factor 3`.

**Solución**:

```bash
# Con 1 broker, usar replication-factor 1
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic my-topic \
  --partitions 3 --replication-factor 1
```

### Error: "Topic already exists"

**Causa**: Topic ya fue creado previamente.

**Solución**:

Usar flag `--if-not-exists`:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic my-topic \
  --partitions 3 --replication-factor 1 \
  --if-not-exists
```

### Error: "Connection refused" al conectar a Kafka

**Causas comunes**:

1. **Kafka no está corriendo**:

```bash
docker compose ps kafka
docker compose up -d kafka
```

2. **Kafka aún no terminó de iniciar**:

Esperar 30 segundos y verificar logs:

```bash
docker compose logs -f kafka
# Buscar: "[KafkaServer id=1] started"
```

3. **ADVERTISED_LISTENERS incorrecto**:

Ver sección "¿Por qué mi aplicación Spring Boot no se conecta a Kafka?"

### Kafka usa mucho espacio en disco

**Causa**: Retención muy alta o muchos mensajes.

**Solución**:

```bash
# Reducir retención a 24 horas
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config retention.ms=86400000

# Limitar tamaño por partición (1 GB)
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config retention.bytes=1073741824
```

---

**Última actualización**: Noviembre 2025
**Curso**: Spring Boot & Apache Kafka - i-Quattro
