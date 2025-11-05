# Lab 04: Docker Compose para Kafka

## Objetivo

Desplegar Apache Kafka y Zookeeper usando Docker Compose, creando una infraestructura local de mensajería lista para desarrollo y testing de microservicios event-driven.

---

## Comandos a ejecutar

```bash
# 1. Crear directorio para configuración de Kafka
mkdir -p kafka-infrastructure
cd kafka-infrastructure

# 2. Crear archivo docker-compose.yml con toda la configuración
cat > docker-compose.yml << 'EOF'
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
      - zookeeper-logs:/var/lib/zookeeper/log

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
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
EOF

# 3. Descargar imágenes de Docker (primera vez, puede tardar)
docker compose pull

# 4. Iniciar servicios en background
docker compose up -d

# 5. Verificar que los servicios están corriendo
docker compose ps

# 6. Ver logs de Zookeeper
docker compose logs zookeeper

# 7. Ver logs de Kafka
docker compose logs kafka

# 8. Seguir logs en tiempo real (Ctrl+C para salir)
docker compose logs -f kafka

# 9. Verificar que Kafka está listo
docker exec -it kafka kafka-broker-api-versions --bootstrap-server localhost:9092

# 10. Entrar al contenedor de Kafka (para ejecutar comandos CLI)
docker exec -it kafka bash

# 11. Dentro del contenedor, listar topics (vacío por ahora)
kafka-topics --bootstrap-server localhost:9092 --list

# 12. Salir del contenedor
exit

# 13. Detener servicios (mantiene datos)
docker compose stop

# 14. Iniciar servicios nuevamente
docker compose start

# 15. Detener y eliminar servicios (mantiene volúmenes)
docker compose down

# 16. Detener, eliminar servicios y volúmenes (limpieza completa)
docker compose down -v
```

**Salida esperada de `docker compose ps`**:

```
NAME        IMAGE                              STATUS        PORTS
kafka       confluentinc/cp-kafka:latest       Up 2 minutes  0.0.0.0:9092->9092/tcp
zookeeper   confluentinc/cp-zookeeper:latest   Up 2 minutes  0.0.0.0:2181->2181/tcp
```

**Salida esperada de `docker compose logs kafka` (extracto)**:

```
kafka  | [2025-11-05 12:00:45,123] INFO Kafka version: 3.6.0 (org.apache.kafka.common.utils.AppInfoParser)
kafka  | [2025-11-05 12:00:45,234] INFO Kafka startTimeMs: 1699187645234 (org.apache.kafka.common.utils.AppInfoParser)
kafka  | [2025-11-05 12:00:45,456] INFO [KafkaServer id=1] started (kafka.server.KafkaServer)
```

**Salida esperada de `kafka-broker-api-versions`**:

```
localhost:9092 (id: 1 rack: null) -> (
  ApiVersion(api_key=0, min_version=0, max_version=8),
  ApiVersion(api_key=1, min_version=0, max_version=12),
  ...
)
```

---

## Desglose del comando

### Comando: `docker compose up -d`

| Componente | Descripción |
|------------|-------------|
| `docker compose` | Herramienta para definir y ejecutar aplicaciones Docker multi-contenedor |
| `up` | Crea e inicia contenedores definidos en docker-compose.yml |
| `-d` | Detached mode (ejecuta en background, libera la terminal) |

**Diferencia con `docker-compose` (hyphen)**:

- `docker compose` (espacio) → Docker Compose V2 (integrado en Docker Engine)
- `docker-compose` (hyphen) → Docker Compose V1 (binario separado, deprecated)

**IMPORTANTE**: Usar siempre `docker compose` (V2) en este curso.

### Comando: `docker compose down -v`

| Componente | Descripción |
|------------|-------------|
| `down` | Detiene y elimina contenedores, redes creadas por `up` |
| `-v` | Elimina también los volúmenes (datos persistidos) |

**Cuándo usar**:

- Sin `-v` → Mantiene datos (topics, mensajes) entre reinicios
- Con `-v` → Limpieza completa (útil para empezar desde cero)

---

## Explicación detallada

### Paso 1: Crear docker-compose.yml

Crear archivo `docker-compose.yml` con el siguiente contenido:

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
      - zookeeper-logs:/var/lib/zookeeper/log

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
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
```

**IMPORTANTE**: NO agregar campo `version:` (deprecated en Docker Compose V2).

### Paso 2: Comprender la configuración de Zookeeper

#### Servicio `zookeeper`

```yaml
zookeeper:
  image: confluentinc/cp-zookeeper:latest
  container_name: zookeeper
  environment:
    ZOOKEEPER_CLIENT_PORT: 2181
    ZOOKEEPER_TICK_TIME: 2000
```

**Explicación de variables**:

| Variable | Valor | Descripción |
|----------|-------|-------------|
| `ZOOKEEPER_CLIENT_PORT` | `2181` | Puerto donde Zookeeper escucha conexiones de clientes (Kafka) |
| `ZOOKEEPER_TICK_TIME` | `2000` | Unidad básica de tiempo en milisegundos para heartbeats |

**Puerto 2181**:

- Puerto estándar de Zookeeper
- Kafka se conecta a `zookeeper:2181`
- Expuesto en host para debugging (opcional)

**Volúmenes**:

```yaml
volumes:
  - zookeeper-data:/var/lib/zookeeper/data
  - zookeeper-logs:/var/lib/zookeeper/log
```

- `zookeeper-data`: Almacena metadata del cluster
- `zookeeper-logs`: Logs de transacciones

**Persistencia**: Los datos sobreviven a reinicios de contenedor.

### Paso 3: Comprender la configuración de Kafka

#### Servicio `kafka`

```yaml
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
```

**Explicación de variables críticas**:

| Variable | Valor | Descripción |
|----------|-------|-------------|
| `KAFKA_BROKER_ID` | `1` | Identificador único del broker en el cluster |
| `KAFKA_ZOOKEEPER_CONNECT` | `zookeeper:2181` | Dirección de Zookeeper (usa nombre del servicio Docker) |
| `KAFKA_ADVERTISED_LISTENERS` | `PLAINTEXT://localhost:9092` | Dirección que Kafka anuncia a productores/consumidores |
| `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR` | `1` | Réplicas del topic interno `__consumer_offsets` |
| `KAFKA_AUTO_CREATE_TOPICS_ENABLE` | `"false"` | Deshabilita creación automática de topics (buena práctica) |

#### Entender KAFKA_ADVERTISED_LISTENERS

Esta es la configuración más crítica y común fuente de errores.

**¿Qué hace?**

Kafka anuncia esta dirección a productores y consumidores para que sepan cómo conectarse.

**Escenarios**:

**Escenario 1: Aplicación Spring Boot en host (NO en Docker)**

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
```

Spring Boot en tu máquina se conecta a `localhost:9092`.

**Escenario 2: Aplicación Spring Boot en Docker**

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
```

Otro contenedor Docker se conecta a `kafka:9092` (nombre del servicio).

**Escenario 3: Ambos (host y Docker)**

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,INTERNAL://kafka:9092
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,INTERNAL:PLAINTEXT
KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,INTERNAL://0.0.0.0:9093
KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
```

**Para este curso**: Usaremos configuración simple con `localhost:9092` (aplicaciones en host).

#### Replication Factor

```yaml
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
```

**¿Por qué 1?**

En desarrollo con 1 solo broker, el replication factor debe ser 1.

**En producción**: Usar 3 brokers con replication factor 3 para alta disponibilidad.

#### Deshabilitar auto-creación de topics

```yaml
KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
```

**Ventajas**:

- Evita topics accidentales por typos
- Control explícito sobre particiones y replication factor
- Buena práctica en producción

**Desventaja**:

- Debes crear topics manualmente con Kafka CLI (Lab 05)

### Paso 4: Iniciar la infraestructura

```bash
docker compose up -d
```

**Proceso interno**:

1. Docker crea red `kafka-infrastructure_default`
2. Inicia contenedor `zookeeper`
3. Espera que Zookeeper esté listo
4. Inicia contenedor `kafka`
5. Kafka se conecta a Zookeeper en `zookeeper:2181`
6. Kafka registra el broker con ID 1
7. Kafka queda listo en `localhost:9092`

**Tiempo estimado**: 30-60 segundos en primera ejecución (descarga de imágenes).

### Paso 5: Verificar servicios corriendo

```bash
docker compose ps
```

**Estados esperados**:

- `Up X minutes` → Servicio corriendo correctamente
- `Restarting` → Servicio reiniciándose (posible error de configuración)
- `Exit 1` → Servicio falló (revisar logs)

### Paso 6: Revisar logs

**Ver logs completos**:

```bash
docker compose logs zookeeper
docker compose logs kafka
```

**Seguir logs en tiempo real**:

```bash
docker compose logs -f kafka
```

**Buscar errores**:

```bash
docker compose logs kafka | grep -i error
docker compose logs kafka | grep -i exception
```

**Logs importantes de Kafka**:

```
[KafkaServer id=1] started (kafka.server.KafkaServer)
```

Indica que Kafka inició correctamente.

```
INFO Registered broker 1 at path /brokers/ids/1 (kafka.zk.KafkaZkClient)
```

Indica que Kafka se registró en Zookeeper.

### Paso 7: Verificar conectividad

**Desde el host**:

```bash
docker exec -it kafka kafka-broker-api-versions --bootstrap-server localhost:9092
```

Si muestra lista de API versions, Kafka está accesible.

**Desde dentro del contenedor**:

```bash
docker exec -it kafka bash

# Ahora estás dentro del contenedor
kafka-broker-api-versions --bootstrap-server localhost:9092
```

### Paso 8: Gestión del ciclo de vida

**Detener servicios (mantiene contenedores)**:

```bash
docker compose stop
```

**Iniciar servicios detenidos**:

```bash
docker compose start
```

**Reiniciar servicios**:

```bash
docker compose restart
```

**Detener y eliminar contenedores (mantiene volúmenes)**:

```bash
docker compose down
```

**Limpieza completa (elimina TODO)**:

```bash
docker compose down -v
```

**ADVERTENCIA**: `-v` elimina topics y mensajes. Solo usar para reset completo.

### Paso 9: Persistencia de datos

Los volúmenes Docker garantizan que:

- Topics creados sobreviven a reinicios
- Mensajes publicados persisten
- Offsets de consumidores se mantienen

**Ver volúmenes creados**:

```bash
docker volume ls | grep kafka
```

**Salida**:

```
local     kafka-infrastructure_kafka-data
local     kafka-infrastructure_zookeeper-data
local     kafka-infrastructure_zookeeper-logs
```

**Eliminar volúmenes manualmente** (si `down -v` no funciona):

```bash
docker volume rm kafka-infrastructure_kafka-data
docker volume rm kafka-infrastructure_zookeeper-data
docker volume rm kafka-infrastructure_zookeeper-logs
```

---

## Conceptos aprendidos

- **Docker Compose V2**: Orquestación de contenedores con sintaxis YAML (sin campo `version`)
- **Zookeeper**: Coordinador del cluster de Kafka
- **Kafka Broker**: Servidor que almacena y distribuye mensajes
- **ADVERTISED_LISTENERS**: Configuración crítica para conectividad entre host y contenedores
- **Replication Factor**: Número de copias de datos (1 en desarrollo, 3 en producción)
- **Volúmenes Docker**: Persistencia de datos entre reinicios
- **depends_on**: Orden de inicio de servicios
- **Auto-create topics**: Buena práctica deshabilitarlo en producción

---

## Troubleshooting

### Problema 1: Kafka no inicia (restarting loop)

**Síntoma**:

```bash
docker compose ps
# kafka   Restarting
```

**Causa**: Error de configuración o Zookeeper no disponible.

**Solución**:

```bash
# Ver logs de Kafka
docker compose logs kafka

# Buscar errores
docker compose logs kafka | grep -i "error\|exception"

# Error común: Connection refused to Zookeeper
# Solución: Verificar que Zookeeper esté corriendo
docker compose ps zookeeper
```

### Problema 2: Connection refused al intentar conectarse

**Síntoma**:

```bash
docker exec -it kafka kafka-broker-api-versions --bootstrap-server localhost:9092
# Error: Connection refused
```

**Causa**: Kafka aún no terminó de iniciar.

**Solución**:

```bash
# Esperar 30-60 segundos más
# Verificar logs
docker compose logs -f kafka

# Buscar mensaje de inicio exitoso
# [KafkaServer id=1] started
```

### Problema 3: Aplicación Spring Boot no se conecta

**Síntoma**:

```
org.apache.kafka.common.errors.TimeoutException: Failed to update metadata
```

**Causa**: `KAFKA_ADVERTISED_LISTENERS` incorrecto.

**Solución**:

Si tu aplicación Spring Boot corre en el host (NO en Docker):

```yaml
# docker-compose.yml
environment:
  KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
```

Si tu aplicación Spring Boot corre en Docker:

```yaml
# docker-compose.yml
environment:
  KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

# application.yml (Spring Boot)
spring:
  kafka:
    bootstrap-servers: kafka:9092
```

### Problema 4: Puerto 9092 ya en uso

**Síntoma**:

```
Error starting userland proxy: listen tcp4 0.0.0.0:9092: bind: address already in use
```

**Causa**: Otra instancia de Kafka o aplicación usa el puerto 9092.

**Solución**:

```bash
# Linux/Mac: Ver qué proceso usa el puerto
lsof -i :9092

# Windows: Ver qué proceso usa el puerto
netstat -ano | findstr :9092

# Matar proceso (reemplaza PID)
kill -9 <PID>

# O cambiar puerto en docker-compose.yml
ports:
  - "9093:9092"  # Host:9093 → Container:9092
```

### Problema 5: Datos no persisten después de reinicio

**Síntoma**: Topics creados desaparecen después de `docker compose down`.

**Causa**: Usaste `-v` que elimina volúmenes.

**Solución**:

```bash
# Detener sin eliminar volúmenes
docker compose down

# Iniciar nuevamente
docker compose up -d

# Topics y datos persisten
```

### Problema 6: Volúmenes ocupan mucho espacio

**Síntoma**: Disco lleno por logs de Kafka.

**Solución**:

```bash
# Ver tamaño de volúmenes
docker system df -v

# Eliminar volúmenes no usados
docker volume prune

# Eliminar volúmenes de Kafka específicamente
docker compose down -v
```

Configurar retención de logs en `docker-compose.yml`:

```yaml
environment:
  KAFKA_LOG_RETENTION_HOURS: 24  # Retener solo 24 horas
  KAFKA_LOG_RETENTION_BYTES: 1073741824  # Máximo 1 GB por partición
```

---

## Desafío adicional

### Desafío 1: Agregar Kafka UI

Kafka UI es una herramienta web para visualizar topics, mensajes y consumidores.

Agregar al `docker-compose.yml`:

```yaml
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
```

Acceder a [http://localhost:8090](http://localhost:8090) para ver UI gráfica.

### Desafío 2: Configurar múltiples brokers

Modifica `docker-compose.yml` para tener 3 brokers:

```yaml
services:
  zookeeper:
    # ... (sin cambios)

  kafka1:
    image: confluentinc/cp-kafka:latest
    container_name: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092

  kafka2:
    image: confluentinc/cp-kafka:latest
    container_name: kafka2
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9093

  kafka3:
    image: confluentinc/cp-kafka:latest
    container_name: kafka3
    ports:
      - "9094:9094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9094
```

Ahora puedes crear topics con `--replication-factor 3`.

### Desafío 3: Migrar a KRaft (sin Zookeeper)

Investiga cómo configurar Kafka en modo KRaft (sin Zookeeper):

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
```

**Recurso**: [Confluent KRaft Guide](https://docs.confluent.io/platform/current/kafka/kraft.html)

---

## Recursos adicionales

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Confluent Platform Docker Images](https://docs.confluent.io/platform/current/installation/docker/image-reference.html)
- [Kafka Docker Quick Start](https://developer.confluent.io/quickstart/kafka-docker/)
- [Kafka Configuration Reference](https://kafka.apache.org/documentation/#configuration)
- [provectuslabs/kafka-ui GitHub](https://github.com/provectus/kafka-ui)
- [Understanding Kafka Listeners](https://www.confluent.io/blog/kafka-listeners-explained/)
