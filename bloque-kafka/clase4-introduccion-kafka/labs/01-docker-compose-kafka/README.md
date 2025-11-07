# Lab 01: Docker Compose para Kafka

## Objetivo

Desplegar Apache Kafka en modo KRaft (sin Zookeeper) usando Docker Compose, creando una infraestructura local de mensajería moderna y simplificada para desarrollo y testing de microservicios event-driven.

---

## Comandos a ejecutar

```bash
# 1. Crear directorio para configuración de Kafka
mkdir -p kafka-infrastructure
cd kafka-infrastructure

# 2. Crear archivo docker-compose.yml con toda la configuración
cat > docker-compose.yml << 'EOF'
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  kafka-data:
EOF

# 3. Descargar imagen de Docker (primera vez, puede tardar)
docker compose pull

# 4. Iniciar servicio en background
docker compose up -d

# 5. Verificar que Kafka está corriendo
docker compose ps

# 6. Ver logs de Kafka
docker compose logs kafka

# 7. Seguir logs en tiempo real (Ctrl+C para salir)
docker compose logs -f kafka

# 8. Verificar que Kafka está listo
docker exec -it kafka kafka-broker-api-versions --bootstrap-server localhost:9092

# 9. Entrar al contenedor de Kafka (para ejecutar comandos CLI)
docker exec -it kafka bash

# 10. Dentro del contenedor, listar topics (vacío por ahora)
kafka-topics --bootstrap-server localhost:9092 --list

# 11. Salir del contenedor
exit

# 12. Detener servicio (mantiene datos)
docker compose stop

# 13. Iniciar servicio nuevamente
docker compose start

# 14. Detener y eliminar servicio (mantiene volumen)
docker compose down

# 15. Detener, eliminar servicio y volumen (limpieza completa)
docker compose down -v
```

**Salida esperada de `docker compose ps`**:

```text
NAME   IMAGE                         STATUS        PORTS
kafka  confluentinc/cp-kafka:latest  Up 2 minutes  0.0.0.0:9092->9092/tcp
```

**Salida esperada de `docker compose logs kafka` (extracto)**:

```text
kafka  | [2025-11-05 12:00:45,123] INFO Kafka version: 3.8.0 (org.apache.kafka.common.utils.AppInfoParser)
kafka  | [2025-11-05 12:00:45,234] INFO [BrokerServer id=1] started (kafka.server.BrokerServer)
```

**Salida esperada de `kafka-broker-api-versions`**:

```text
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

### Paso 1: Comprender KRaft vs Zookeeper

**¿Qué es KRaft?**

KRaft (Kafka Raft) es el nuevo modo de consenso de Apache Kafka que elimina la dependencia de Zookeeper. Desde Kafka 3.3+ es production-ready y desde Kafka 4.0 será el único modo soportado.

**Ventajas de KRaft**:

- Arquitectura más simple (un solo servicio en vez de dos)
- Menos recursos (no necesita Zookeeper)
- Inicio más rápido
- Mejor escalabilidad
- Es el futuro de Kafka

**Arquitectura tradicional (Zookeeper)**:

```
┌────────────┐
│ Zookeeper  │ ← Gestiona metadata del cluster
└────────────┘
      ↑
      │
┌────────────┐
│   Kafka    │ ← Almacena mensajes
└────────────┘
```

**Arquitectura KRaft (moderna)**:

```
┌────────────┐
│   Kafka    │ ← Gestiona metadata Y mensajes
└────────────┘
```

### Paso 2: Crear docker-compose.yml

Crear archivo `docker-compose.yml` con el siguiente contenido:

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  kafka-data:
```

**IMPORTANTE**: NO agregar campo `version:` (deprecated en Docker Compose V2).

### Paso 3: Comprender la configuración KRaft

#### Variables críticas de KRaft

| Variable | Valor | Descripción |
|----------|-------|-------------|
| `KAFKA_NODE_ID` | `1` | Identificador único del nodo en el cluster KRaft |
| `KAFKA_PROCESS_ROLES` | `broker,controller` | Roles del nodo: broker (datos) + controller (metadata) |
| `KAFKA_LISTENERS` | `PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093` | Puertos donde escucha (clientes:9092, control:9093) |
| `KAFKA_ADVERTISED_LISTENERS` | `PLAINTEXT://localhost:9092` | Dirección que anuncia a productores/consumidores |
| `KAFKA_CONTROLLER_LISTENER_NAMES` | `CONTROLLER` | Nombre del listener para consenso KRaft |
| `KAFKA_LISTENER_SECURITY_PROTOCOL_MAP` | `CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT` | Mapeo de protocolos de seguridad |
| `KAFKA_CONTROLLER_QUORUM_VOTERS` | `1@localhost:9093` | Votantes del quorum KRaft (formato: id@host:port) |
| `CLUSTER_ID` | `MkU3OEVBNTcwNTJENDM2Qk` | ID único del cluster (base64, 16 bytes) |

#### KAFKA_PROCESS_ROLES: Roles combinados o separados

**Modo combinado (desarrollo)**:

```yaml
KAFKA_PROCESS_ROLES: broker,controller
```

Un solo nodo cumple ambos roles. Ideal para desarrollo y testing.

**Modo separado (producción)**:

```yaml
# Nodos dedicados a control (3 nodos)
KAFKA_PROCESS_ROLES: controller

# Nodos dedicados a datos (3+ nodos)
KAFKA_PROCESS_ROLES: broker
```

Separación de responsabilidades para alta disponibilidad.

#### KAFKA_LISTENERS: Entender los puertos

```yaml
KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
```

- `PLAINTEXT://0.0.0.0:9092` → Puerto para productores/consumidores (tráfico de datos)
- `CONTROLLER://0.0.0.0:9093` → Puerto para consenso KRaft (tráfico de control interno)

**Importante**: Puerto 9093 es interno, NO exponerlo fuera del contenedor.

#### KAFKA_CONTROLLER_QUORUM_VOTERS: El quorum de control

```yaml
KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
```

Define los nodos que participan en el consenso KRaft.

**Formato**: `id@host:port,id@host:port,...`

**Ejemplo con 3 nodos**:

```yaml
KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
```

#### CLUSTER_ID: Identificador único del cluster

```yaml
CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
```

**¿Qué es?**

Un identificador único base64 de 16 bytes que identifica al cluster KRaft.

**¿Cómo generarlo?**

```bash
# Generar nuevo CLUSTER_ID
docker run --rm confluentinc/cp-kafka:latest kafka-storage random-uuid
```

**Importante**: Una vez creado el cluster, NO cambiar el CLUSTER_ID o se perderán los datos.

#### KAFKA_ADVERTISED_LISTENERS: Conectividad

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
```

Esta es la dirección que Kafka anuncia a clientes.

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

**Para este curso**: Usaremos `localhost:9092` (aplicaciones en host).

#### Replication Factor: Configuración para desarrollo

```yaml
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
```

**¿Por qué 1?**

En desarrollo con 1 solo broker, el replication factor debe ser 1.

**En producción**: Usar 3 brokers con replication factor 3 para alta disponibilidad.

### Paso 4: Iniciar la infraestructura

```bash
docker compose up -d
```

**Proceso interno KRaft**:

1. Docker crea red `kafka-infrastructure_default`
2. Inicia contenedor `kafka`
3. Kafka formatea el directorio de datos con CLUSTER_ID
4. Kafka inicia el controlador KRaft
5. Kafka inicia el broker
6. Kafka queda listo en `localhost:9092`

**Tiempo estimado**: 20-40 segundos en primera ejecución (descarga de imagen).

**Ventaja sobre Zookeeper**: Inicio más rápido (no espera dependencias externas).

### Paso 5: Verificar servicio corriendo

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

**Logs importantes de Kafka KRaft**:

```text
[BrokerServer id=1] started (kafka.server.BrokerServer)
```

Indica que el broker inició correctamente.

```text
[QuorumController id=1] Becoming active at epoch 1
```

Indica que el controlador KRaft está activo.

**Diferencia con Zookeeper**: NO verás logs de conexión a Zookeeper (porque no existe).

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

**Detener servicio (mantiene contenedor)**:

```bash
docker compose stop
```

**Iniciar servicio detenido**:

```bash
docker compose start
```

**Reiniciar servicio**:

```bash
docker compose restart
```

**Detener y eliminar contenedor (mantiene volumen)**:

```bash
docker compose down
```

**Limpieza completa (elimina TODO)**:

```bash
docker compose down -v
```

**ADVERTENCIA**: `-v` elimina topics y mensajes. Solo usar para reset completo.

### Paso 9: Persistencia de datos

El volumen Docker garantiza que:

- Topics creados sobreviven a reinicios
- Mensajes publicados persisten
- Offsets de consumidores se mantienen
- Metadata del controlador KRaft persiste

**Ver volumen creado**:

```bash
docker volume ls | grep kafka
```

**Salida**:

```text
local     kafka-infrastructure_kafka-data
```

**Eliminar volumen manualmente** (si `down -v` no funciona):

```bash
docker volume rm kafka-infrastructure_kafka-data
```

**Ventaja de KRaft**: Solo 1 volumen necesario (arquitectura simplificada).

---

## Conceptos aprendidos

- **Docker Compose V2**: Orquestación de contenedores con sintaxis YAML (sin campo `version`)
- **KRaft**: Modo moderno de Kafka sin dependencia de Zookeeper
- **KAFKA_PROCESS_ROLES**: Roles de nodo (broker, controller, o ambos)
- **CLUSTER_ID**: Identificador único del cluster KRaft
- **KAFKA_CONTROLLER_QUORUM_VOTERS**: Nodos del quorum de consenso
- **Listeners separados**: Puerto 9092 (clientes) y 9093 (control interno)
- **ADVERTISED_LISTENERS**: Configuración crítica para conectividad
- **Replication Factor**: Número de copias de datos (1 en desarrollo, 3 en producción)
- **Volúmenes Docker**: Persistencia de datos entre reinicios

---

## Troubleshooting

### Problema 1: Kafka no inicia (restarting loop)

**Síntoma**:

```bash
docker compose ps
# kafka   Restarting
```

**Causa**: Error de configuración KRaft.

**Solución**:

```bash
# Ver logs de Kafka
docker compose logs kafka

# Buscar errores
docker compose logs kafka | grep -i "error\|exception"

# Error común: CLUSTER_ID inválido
# Solución: Generar nuevo CLUSTER_ID
docker run --rm confluentinc/cp-kafka:latest kafka-storage random-uuid
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
# Esperar 20-40 segundos más
# Verificar logs
docker compose logs -f kafka

# Buscar mensaje de inicio exitoso
# [BrokerServer id=1] started
```

### Problema 3: Aplicación Spring Boot no se conecta

**Síntoma**:

```text
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

```text
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

### Problema 6: Error "Cluster ID doesn't match stored"

**Síntoma**:

```text
FATAL The Cluster ID doesn't match stored clusterId
```

**Causa**: Cambiaste CLUSTER_ID con datos existentes.

**Solución**:

```bash
# Eliminar volumen y empezar desde cero
docker compose down -v
docker compose up -d
```

### Problema 7: Volumen ocupa mucho espacio

**Síntoma**: Disco lleno por logs de Kafka.

**Solución**:

```bash
# Ver tamaño de volúmenes
docker system df -v

# Eliminar volúmenes no usados
docker volume prune

# Eliminar volumen de Kafka específicamente
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
services:
  kafka:
    # ... (sin cambios)

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
```

Acceder a [http://localhost:8090](http://localhost:8090) para ver UI gráfica.

**Nota**: NO configurar Zookeeper (Kafka UI soporta KRaft nativamente).

### Desafío 2: Configurar múltiples brokers KRaft

Modifica `docker-compose.yml` para tener 3 brokers en modo KRaft:

```yaml
services:
  kafka1:
    image: confluentinc/cp-kafka:latest
    container_name: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk

  kafka2:
    image: confluentinc/cp-kafka:latest
    container_name: kafka2
    ports:
      - "9093:9092"
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk

  kafka3:
    image: confluentinc/cp-kafka:latest
    container_name: kafka3
    ports:
      - "9094:9092"
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9094
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
```

Ahora puedes crear topics con `--replication-factor 3`.

**Importante**: Los 3 nodos deben usar el mismo CLUSTER_ID.

### Desafío 3: Separar roles (controller vs broker)

Configura un cluster con 3 controllers dedicados y 3 brokers dedicados:

```yaml
services:
  # Controllers dedicados
  controller1:
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: controller
      # ...

  controller2:
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: controller
      # ...

  controller3:
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: controller
      # ...

  # Brokers dedicados
  broker1:
    environment:
      KAFKA_NODE_ID: 4
      KAFKA_PROCESS_ROLES: broker
      # ...

  broker2:
    environment:
      KAFKA_NODE_ID: 5
      KAFKA_PROCESS_ROLES: broker
      # ...

  broker3:
    environment:
      KAFKA_NODE_ID: 6
      KAFKA_PROCESS_ROLES: broker
      # ...
```

**Recurso**: [Confluent KRaft Guide](https://docs.confluent.io/platform/current/kafka/kraft.html)

---

## Recursos adicionales

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Confluent Platform Docker Images](https://docs.confluent.io/platform/current/installation/docker/image-reference.html)
- [Kafka KRaft Quick Start](https://kafka.apache.org/documentation/#quickstart_startserver)
- [KRaft Overview (Apache Kafka)](https://kafka.apache.org/documentation/#kraft)
- [Confluent KRaft Guide](https://docs.confluent.io/platform/current/kafka/kraft.html)
- [provectuslabs/kafka-ui GitHub](https://github.com/provectus/kafka-ui)
- [Understanding Kafka Listeners](https://www.confluent.io/blog/kafka-listeners-explained/)