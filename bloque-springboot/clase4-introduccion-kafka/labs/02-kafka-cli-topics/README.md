# Lab 02: Kafka CLI - Crear Topics

## Objetivo

Utilizar las herramientas de línea de comandos de Kafka (Kafka CLI) para crear, listar, describir y eliminar topics, preparando la infraestructura de mensajería para el dominio e-commerce del curso.

---

## Comandos a ejecutar

```bash
# 1. Verificar que Kafka está corriendo
docker compose ps

# 2. Entrar al contenedor de Kafka
docker exec -it kafka bash

# ==========================================
# DENTRO DEL CONTENEDOR DE KAFKA
# ==========================================

# 3. Listar topics existentes (vacío inicialmente)
kafka-topics --bootstrap-server localhost:9092 --list

# 4. Crear topic para eventos de productos
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.products.created \
  --partitions 3 \
  --replication-factor 1

# 5. Crear topic para eventos de órdenes (placed)
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.placed \
  --partitions 3 \
  --replication-factor 1

# 6. Crear topic para eventos de órdenes (confirmed)
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.confirmed \
  --partitions 3 \
  --replication-factor 1

# 7. Crear topic para eventos de órdenes (cancelled)
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.cancelled \
  --partitions 3 \
  --replication-factor 1

# 8. Crear topic para eventos de inventario
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.inventory.updated \
  --partitions 3 \
  --replication-factor 1

# 9. Listar todos los topics creados
kafka-topics --bootstrap-server localhost:9092 --list

# 10. Describir un topic específico
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.orders.placed

# 11. Ver configuración de un topic
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --describe

# 12. Modificar número de particiones de un topic
kafka-topics --bootstrap-server localhost:9092 \
  --alter \
  --topic ecommerce.products.created \
  --partitions 5

# 13. Eliminar un topic (CUIDADO: operación destructiva)
kafka-topics --bootstrap-server localhost:9092 \
  --delete \
  --topic ecommerce.products.created

# 14. Salir del contenedor
exit
```

**Salida esperada de `--list`**:

```
ecommerce.inventory.updated
ecommerce.orders.cancelled
ecommerce.orders.confirmed
ecommerce.orders.placed
ecommerce.products.created
```

**Salida esperada de `--describe`**:

```
Topic: ecommerce.orders.placed  TopicId: Xy7J9kL3RQWaB8dF2Hg4Zw  PartitionCount: 3  ReplicationFactor: 1  Configs:
  Topic: ecommerce.orders.placed  Partition: 0  Leader: 1  Replicas: 1  Isr: 1
  Topic: ecommerce.orders.placed  Partition: 1  Leader: 1  Replicas: 1  Isr: 1
  Topic: ecommerce.orders.placed  Partition: 2  Leader: 1  Replicas: 1  Isr: 1
```

---

## Desglose del comando

### Comando: `kafka-topics --bootstrap-server localhost:9092 --create --topic ecommerce.orders.placed --partitions 3 --replication-factor 1`

| Componente | Descripción |
|------------|-------------|
| `kafka-topics` | Herramienta CLI para gestionar topics de Kafka |
| `--bootstrap-server localhost:9092` | Dirección del broker de Kafka |
| `--create` | Operación a realizar (crear topic) |
| `--topic ecommerce.orders.placed` | Nombre del topic |
| `--partitions 3` | Número de particiones del topic |
| `--replication-factor 1` | Número de réplicas por partición |

**Operaciones disponibles**:

| Operación | Descripción | Ejemplo |
|-----------|-------------|---------|
| `--create` | Crear nuevo topic | `--create --topic my-topic` |
| `--list` | Listar todos los topics | `--list` |
| `--describe` | Mostrar detalles de un topic | `--describe --topic my-topic` |
| `--alter` | Modificar configuración | `--alter --topic my-topic --partitions 5` |
| `--delete` | Eliminar topic | `--delete --topic my-topic` |

---

## Explicación detallada

### Paso 1: Acceder al contenedor de Kafka

```bash
docker exec -it kafka bash
```

**¿Por qué desde el contenedor?**

Las herramientas Kafka CLI (`kafka-topics`, `kafka-console-producer`, etc.) están instaladas dentro del contenedor de Kafka.

**Alternativa** (instalar Kafka CLI en el host):

Descargar Kafka desde [https://kafka.apache.org/downloads](https://kafka.apache.org/downloads) y agregar `bin/` al PATH.

### Paso 2: Comprender la convención de nombres de topics

**Formato estándar del curso**: `app-name.domain.event-type`

#### Topics del dominio e-commerce

| Topic | Productor | Consumidores | Propósito |
|-------|-----------|--------------|-----------|
| `ecommerce.products.created` | product-service | analytics-service | Notificar nuevos productos |
| `ecommerce.products.updated` | product-service | analytics-service | Notificar actualización de productos |
| `ecommerce.orders.placed` | order-service | inventory-service, analytics-service | Nueva orden creada |
| `ecommerce.orders.confirmed` | inventory-service | order-service | Orden confirmada (stock disponible) |
| `ecommerce.orders.cancelled` | inventory-service | order-service | Orden cancelada (stock insuficiente) |
| `ecommerce.inventory.updated` | inventory-service | analytics-service | Actualización de stock |

**Criterios de diseño**:

- **app-name** (`ecommerce`): Identifica el sistema
- **domain** (`products`, `orders`, `inventory`): Contexto de negocio
- **event-type** (`created`, `placed`, `confirmed`): Acción que ocurrió (past tense)

### Paso 3: Crear topics con configuración adecuada

#### Comando básico de creación

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.placed \
  --partitions 3 \
  --replication-factor 1
```

**Salida**:

```
Created topic ecommerce.orders.placed.
```

#### ¿Cuántas particiones elegir?

**Regla general**: `partitions ≥ número máximo de consumidores esperados`

**Para este curso**:

- Usaremos **3 particiones** para todos los topics
- Permite hasta 3 consumidores en paralelo
- Balance entre paralelismo y overhead

**En producción**:

Calcular según throughput:

```
partitions = throughput_requerido / throughput_por_consumidor
```

Ejemplo:

- Throughput requerido: 10,000 mensajes/segundo
- Throughput por consumidor: 500 mensajes/segundo
- Particiones necesarias: 10,000 / 500 = 20

#### ¿Qué replication-factor usar?

**Desarrollo** (1 broker):

```
--replication-factor 1
```

**Producción** (3+ brokers):

```
--replication-factor 3
```

**¿Por qué 3?**

- Tolera fallo de hasta 2 brokers
- Balance entre redundancia y overhead de red
- Estándar de la industria

**IMPORTANTE**: `replication-factor` NO puede ser mayor que el número de brokers disponibles.

### Paso 4: Listar topics

```bash
kafka-topics --bootstrap-server localhost:9092 --list
```

**Salida**:

```
ecommerce.inventory.updated
ecommerce.orders.cancelled
ecommerce.orders.confirmed
ecommerce.orders.placed
ecommerce.products.created
```

**Topics internos de Kafka** (ocultos por defecto):

```bash
# Listar TODOS los topics (incluye internos)
kafka-topics --bootstrap-server localhost:9092 --list --exclude-internal=false
```

Topics internos:

- `__consumer_offsets`: Almacena offsets de consumer groups
- `__transaction_state`: Estado de transacciones (Kafka Transactions)

### Paso 5: Describir un topic

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.orders.placed
```

**Salida detallada**:

```
Topic: ecommerce.orders.placed  TopicId: Xy7J9kL3RQWaB8dF2Hg4Zw  PartitionCount: 3  ReplicationFactor: 1  Configs:
  Topic: ecommerce.orders.placed  Partition: 0  Leader: 1  Replicas: 1  Isr: 1
  Topic: ecommerce.orders.placed  Partition: 1  Leader: 1  Replicas: 1  Isr: 1
  Topic: ecommerce.orders.placed  Partition: 2  Leader: 1  Replicas: 1  Isr: 1
```

**Interpretación**:

- `PartitionCount: 3`: Topic tiene 3 particiones
- `ReplicationFactor: 1`: Cada partición tiene 1 réplica (no hay redundancia)
- `Leader: 1`: Broker 1 es el líder de todas las particiones
- `Replicas: 1`: Réplicas están en broker 1
- `Isr: 1`: In-Sync Replicas (réplicas sincronizadas) en broker 1

**Con múltiples brokers** (producción):

```
Topic: orders  Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
Topic: orders  Partition: 1  Leader: 2  Replicas: 2,3,1  Isr: 2,3,1
Topic: orders  Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3,1,2
```

Cada partición tiene réplicas en 3 brokers diferentes.

### Paso 6: Ver configuración avanzada de un topic

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --describe
```

**Salida**:

```
Dynamic configs for topic ecommerce.orders.placed are:
  retention.ms=604800000 sensitive=false synonyms={DEFAULT_CONFIG:log.retention.ms=604800000}
```

**Configuraciones importantes**:

| Config | Valor por defecto | Descripción |
|--------|-------------------|-------------|
| `retention.ms` | `604800000` (7 días) | Tiempo de retención de mensajes |
| `retention.bytes` | `-1` (ilimitado) | Tamaño máximo de mensajes retenidos |
| `segment.ms` | `604800000` (7 días) | Tiempo antes de crear nuevo segment |
| `cleanup.policy` | `delete` | Estrategia de limpieza (delete o compact) |

### Paso 7: Modificar configuración de retención

**Cambiar retención a 24 horas**:

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --alter \
  --add-config retention.ms=86400000
```

**Cambiar retención a ilimitada**:

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --alter \
  --add-config retention.ms=-1
```

**Eliminar configuración personalizada** (volver a default):

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --alter \
  --delete-config retention.ms
```

### Paso 8: Aumentar número de particiones

**IMPORTANTE**: Puedes AUMENTAR particiones, pero NO puedes reducirlas.

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --alter \
  --topic ecommerce.products.created \
  --partitions 5
```

**Salida**:

```
Updated partition count for topic ecommerce.products.created.
```

**Verificar cambio**:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.products.created
```

**Consecuencias**:

- Mensajes existentes NO se redistribuyen
- Nuevos mensajes se distribuyen entre 5 particiones
- Consumidores existentes se rebalancea

### Paso 9: Eliminar un topic

**ADVERTENCIA**: Operación destructiva, elimina todos los mensajes.

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --delete \
  --topic ecommerce.products.created
```

**Salida**:

```
Topic ecommerce.products.created is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.
```

**Verificar eliminación**:

```bash
kafka-topics --bootstrap-server localhost:9092 --list
# Topic ya no aparece
```

**Habilitar eliminación de topics** (si no está habilitado):

Agregar a `docker-compose.yml`:

```yaml
environment:
  KAFKA_DELETE_TOPIC_ENABLE: "true"
```

### Paso 10: Crear todos los topics del curso

**Script para crear topics en batch**:

```bash
#!/bin/bash

BOOTSTRAP_SERVER="localhost:9092"
PARTITIONS=3
REPLICATION_FACTOR=1

# Array de topics del dominio e-commerce
TOPICS=(
  "ecommerce.products.created"
  "ecommerce.products.updated"
  "ecommerce.orders.placed"
  "ecommerce.orders.confirmed"
  "ecommerce.orders.cancelled"
  "ecommerce.inventory.updated"
  "ecommerce.inventory.insufficient"
)

for TOPIC in "${TOPICS[@]}"; do
  echo "Creando topic: $TOPIC"
  kafka-topics --bootstrap-server $BOOTSTRAP_SERVER \
    --create \
    --topic $TOPIC \
    --partitions $PARTITIONS \
    --replication-factor $REPLICATION_FACTOR \
    --if-not-exists
done

echo "Topics creados exitosamente"
kafka-topics --bootstrap-server $BOOTSTRAP_SERVER --list
```

Guardar como `create-topics.sh`, dar permisos de ejecución y ejecutar:

```bash
chmod +x create-topics.sh
docker cp create-topics.sh kafka:/tmp/
docker exec -it kafka bash -c "/tmp/create-topics.sh"
```

---

## Conceptos aprendidos

- **Kafka CLI**: Herramientas de línea de comandos para administrar Kafka
- **kafka-topics**: Comando principal para gestión de topics
- **Particiones**: División de topics para paralelismo (pueden aumentarse, NO reducirse)
- **Replication factor**: Número de copias de datos (1 en dev, 3 en prod)
- **Retención**: Tiempo o tamaño máximo de almacenamiento de mensajes
- **Leader y replicas**: Distribución de particiones entre brokers
- **ISR (In-Sync Replicas)**: Réplicas sincronizadas con el líder
- **Topic configs**: Configuración avanzada (retention, cleanup, compression)
- **Convención de nombres**: app-name.domain.event-type
- **--if-not-exists**: Flag para evitar error si topic ya existe

---

## Troubleshooting

### Problema 1: Topic no se crea (error de replication-factor)

**Síntoma**:

```
Error while executing topic command : Replication factor: 3 larger than available brokers: 1.
```

**Causa**: Solo hay 1 broker, pero se especificó `--replication-factor 3`.

**Solución**:

```bash
# Con 1 broker, usar replication-factor 1
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic my-topic \
  --partitions 3 \
  --replication-factor 1
```

### Problema 2: No se puede aumentar particiones

**Síntoma**:

```
Error while executing topic command : Topic currently has 5 partitions, which is higher than the requested 3.
```

**Causa**: Intentaste REDUCIR particiones (no permitido).

**Solución**:

NO puedes reducir particiones. Opciones:

1. **Mantener 5 particiones** (recomendado)
2. **Eliminar y recrear topic** (pierdes todos los datos)

```bash
kafka-topics --bootstrap-server localhost:9092 --delete --topic my-topic
kafka-topics --bootstrap-server localhost:9092 --create --topic my-topic --partitions 3 --replication-factor 1
```

### Problema 3: Topic no se elimina

**Síntoma**:

```
Topic my-topic is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.
```

**Causa**: Eliminación de topics no habilitada en broker.

**Solución**:

Agregar a `docker-compose.yml`:

```yaml
kafka:
  environment:
    KAFKA_DELETE_TOPIC_ENABLE: "true"
```

Reiniciar servicios:

```bash
docker compose down
docker compose up -d
```

### Problema 4: Connection refused al ejecutar kafka-topics

**Síntoma**:

```
Error connecting to node localhost:9092 (id: -1 rack: null)
```

**Causa**: Kafka no está corriendo o no accesible en `localhost:9092`.

**Solución**:

```bash
# Verificar que Kafka está corriendo
docker compose ps

# Si no está corriendo, iniciarlo
docker compose up -d kafka

# Esperar 30 segundos y verificar logs
docker compose logs -f kafka
```

### Problema 5: Kafka CLI no encontrado

**Síntoma**:

```bash
kafka-topics
# bash: kafka-topics: command not found
```

**Causa**: Estás ejecutando desde el host, no desde el contenedor de Kafka.

**Solución**:

```bash
# Entrar al contenedor primero
docker exec -it kafka bash

# Ahora ejecutar kafka-topics
kafka-topics --bootstrap-server localhost:9092 --list
```

### Problema 6: Topic con nombre incorrecto

**Síntoma**: Creaste `ecommerce-orders-placed` en lugar de `ecommerce.orders.placed`.

**Solución**:

```bash
# Eliminar topic incorrecto
kafka-topics --bootstrap-server localhost:9092 \
  --delete \
  --topic ecommerce-orders-placed

# Crear topic con nombre correcto
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.orders.placed \
  --partitions 3 \
  --replication-factor 1
```

---

## Desafío adicional

### Desafío 1: Crear topic con compactación

Algunos topics como `ecommerce.inventory.current` deben usar **log compaction** para mantener solo el último estado:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.inventory.current \
  --partitions 3 \
  --replication-factor 1 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.01 \
  --config segment.ms=100
```

**Cleanup policies**:

- `delete`: Eliminar mensajes antiguos (por defecto)
- `compact`: Mantener solo último mensaje por clave
- `compact,delete`: Ambas estrategias

### Desafío 2: Crear topic con compresión

Habilitar compresión para reducir uso de red y almacenamiento:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.analytics.events \
  --partitions 3 \
  --replication-factor 1 \
  --config compression.type=snappy
```

**Tipos de compresión**:

- `gzip`: Máxima compresión, más CPU
- `snappy`: Balance (recomendado)
- `lz4`: Más rápido, menos compresión
- `zstd`: Mejor ratio, requiere Kafka 2.1+

### Desafío 3: Automatizar creación de topics al iniciar Kafka

Crear script que se ejecute al iniciar el contenedor:

**create-topics.sh**:

```bash
#!/bin/bash
# Esperar que Kafka esté listo
sleep 30

TOPICS=(
  "ecommerce.products.created"
  "ecommerce.orders.placed"
  "ecommerce.inventory.updated"
)

for TOPIC in "${TOPICS[@]}"; do
  kafka-topics --bootstrap-server localhost:9092 \
    --create \
    --topic $TOPIC \
    --partitions 3 \
    --replication-factor 1 \
    --if-not-exists
done
```

**docker-compose.yml**:

```yaml
kafka:
  image: confluentinc/cp-kafka:latest
  volumes:
    - ./create-topics.sh:/usr/local/bin/create-topics.sh
  command: >
    bash -c "
      /etc/confluent/docker/run &
      /usr/local/bin/create-topics.sh &&
      tail -f /dev/null
    "
```

---

## Recursos adicionales

- [Kafka Topics Documentation](https://kafka.apache.org/documentation/#topicconfigs)
- [Confluent CLI Tools](https://docs.confluent.io/platform/current/installation/cli-reference.html)
- [Kafka CLI Cheat Sheet](https://gist.github.com/ursuad/e5b8542024a15e4db601f34906b30bb5)
- [Log Compaction](https://kafka.apache.org/documentation/#compaction)
- [Topic Configuration Reference](https://kafka.apache.org/documentation/#topicconfigs)
- [Partitioning Best Practices](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/)
