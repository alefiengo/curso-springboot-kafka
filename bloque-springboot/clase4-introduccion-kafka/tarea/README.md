# Tarea - Clase 4: Introducción a Apache Kafka

## Objetivo

Desplegar infraestructura de Apache Kafka con Docker Compose, crear topics del dominio e-commerce y experimentar con productores y consumidores desde la línea de comandos.

---

## Contexto

Has completado los laboratorios de la Clase 4 y ahora necesitas preparar la infraestructura de Kafka para las siguientes clases donde integraremos microservicios Spring Boot con mensajería basada en eventos.

---

## Requisitos previos

- Clase 4 completada (Labs 00-02)
- Docker y Docker Compose instalados
- Conocimientos de Kafka CLI

---

## Parte 1: Infraestructura de Kafka (40%)

### Tarea 1.1: Desplegar Kafka con Docker Compose

Crea un directorio `kafka-infrastructure/` en la raíz de tu proyecto y agrega:

**Archivo**: `kafka-infrastructure/docker-compose.yml`

Configuración requerida:

- Zookeeper en puerto `2181`
- Kafka en puerto `9092`
- `KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"` (deshabilitado)
- Volúmenes para persistencia de datos
- `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1`

**Validación**:

1. Iniciar servicios: `docker compose up -d`
2. Verificar estado: `docker compose ps` (ambos deben estar "Up")
3. Captura de pantalla de logs: `docker compose logs kafka` mostrando inicio exitoso

### Tarea 1.2: Verificar conectividad de Kafka

Ejecuta los siguientes comandos y documenta la salida:

```bash
# Entrar al contenedor
docker exec -it kafka bash

# Verificar versión de Kafka
kafka-broker-api-versions --bootstrap-server localhost:9092
```

**Documenta**:

- Captura mostrando que Kafka responde correctamente
- Versión de Kafka utilizada

---

## Parte 2: Creación de topics (40%)

### Tarea 2.1: Crear topics del dominio e-commerce

Usando Kafka CLI, crea los siguientes topics:

| Topic | Particiones | Replication Factor |
|-------|-------------|-------------------|
| `ecommerce.products.created` | 3 | 1 |
| `ecommerce.products.updated` | 3 | 1 |
| `ecommerce.orders.placed` | 3 | 1 |
| `ecommerce.orders.confirmed` | 3 | 1 |
| `ecommerce.orders.cancelled` | 3 | 1 |
| `ecommerce.inventory.updated` | 3 | 1 |

**Comandos**:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.products.created \
  --partitions 3 \
  --replication-factor 1
```

Repite para cada topic.

### Tarea 2.2: Verificar topics creados

Ejecuta y documenta:

```bash
# Listar todos los topics
kafka-topics --bootstrap-server localhost:9092 --list

# Describir un topic específico
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.products.created
```

**Documenta**:

- Captura de salida de `--list` mostrando los 6 topics
- Captura de `--describe` de al menos 2 topics mostrando particiones

### Tarea 2.3: Configurar retención personalizada

Configura el topic `ecommerce.orders.placed` con retención de 24 horas:

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --alter \
  --add-config retention.ms=86400000
```

**Validación**:

```bash
# Ver configuración aplicada
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.orders.placed \
  --describe
```

**Documenta**: Captura mostrando `retention.ms=86400000`

---

## Parte 3: Producción y consumo de mensajes (20%)

### Tarea 3.1: Producir mensajes desde CLI

Publica mensajes al topic `ecommerce.products.created`:

```bash
kafka-console-producer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created
```

Envía al menos 5 mensajes simulando productos en formato JSON:

```json
{"productId":"1","name":"Laptop","price":1500}
{"productId":"2","name":"Mouse","price":25}
{"productId":"3","name":"Teclado","price":80}
{"productId":"4","name":"Monitor","price":300}
{"productId":"5","name":"Webcam","price":60}
```

### Tarea 3.2: Consumir mensajes desde CLI

En otra terminal, consume mensajes desde el inicio:

```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning
```

**Documenta**:

- Captura mostrando los 5 mensajes consumidos correctamente

### Tarea 3.3: Producir mensajes con clave

Publica mensajes al topic `ecommerce.orders.placed` con clave (orderId):

```bash
kafka-console-producer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --property "parse.key=true" \
  --property "key.separator=:"
```

Envía 3 mensajes con formato `key:value`:

```
ORD-001:{"orderId":"ORD-001","productId":"1","quantity":2}
ORD-002:{"orderId":"ORD-002","productId":"3","quantity":1}
ORD-003:{"orderId":"ORD-003","productId":"5","quantity":3}
```

### Tarea 3.4: Consumir mensajes con clave

Consume mostrando la clave:

```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.orders.placed \
  --from-beginning \
  --property print.key=true
```

**Documenta**:

- Captura mostrando mensajes con sus claves (formato `ORD-001 {...}`)

---

## Entrega

### Formato de entrega

Crear un repositorio Git privado con la siguiente estructura:

```
kafka-tarea-clase4/
├── kafka-infrastructure/
│   └── docker-compose.yml
├── capturas/
│   ├── 01-kafka-running.png
│   ├── 02-topics-list.png
│   ├── 03-topic-describe.png
│   ├── 04-retention-config.png
│   ├── 05-producer-consumer.png
│   └── 06-producer-consumer-keys.png
├── scripts/
│   ├── create-topics.sh
│   └── configure-retention.sh
└── README.md (documentación de la tarea)
```

### Documentación requerida (README.md)

Tu README.md debe incluir:

#### 1. Instrucciones de ejecución

```markdown
## Cómo iniciar Kafka

```bash
cd kafka-infrastructure
docker compose up -d
```

## Verificar estado

```bash
docker compose ps
docker compose logs kafka
```

## Crear topics

```bash
docker exec -it kafka bash
./scripts/create-topics.sh
```
```

#### 2. Topics creados

Lista de todos los topics con:

- Nombre del topic
- Número de particiones
- Replication factor
- Configuración especial (si aplica)

#### 3. Comandos utilizados

Documenta todos los comandos ejecutados con breve explicación.

#### 4. Capturas de pantalla

Incluir capturas de:

- Kafka y Zookeeper corriendo (`docker compose ps`)
- Lista de topics creados
- Descripción de al menos 2 topics
- Configuración de retención modificada
- Productor y consumidor funcionando
- Productor y consumidor con claves

#### 5. Reflexión personal

Responde (mínimo 250 palabras):

1. ¿Cuál es la principal ventaja de usar Kafka en lugar de una base de datos para comunicación entre microservicios?
2. ¿Qué pasaría si no especificas un `replication-factor` en producción con múltiples brokers?
3. Explica con tus palabras qué es una partición y por qué es importante para escalabilidad.
4. ¿Cuándo usarías una clave (key) al enviar mensajes a Kafka?
5. ¿Qué diferencia observaste entre consumir desde el inicio (`--from-beginning`) vs consumir solo mensajes nuevos?

---

## Criterios de evaluación

| Criterio | Puntaje | Descripción |
|----------|---------|-------------|
| **Infraestructura (40%)** | | |
| docker-compose.yml correcto | 15 | Zookeeper, Kafka, volúmenes |
| Kafka inicia correctamente | 10 | Logs muestran inicio exitoso |
| Volúmenes configurados | 5 | Persistencia de datos |
| Documentación | 10 | README con instrucciones claras |
| **Topics (40%)** | | |
| 6 topics creados correctamente | 20 | Nombres, particiones, replication factor |
| Retención personalizada | 5 | Topic orders.placed con 24h |
| Validación con describe | 10 | Capturas de configuración |
| Scripts de creación | 5 | Automatización de comandos |
| **Producción/Consumo (20%)** | | |
| Mensajes publicados correctamente | 5 | 5 mensajes en products.created |
| Consumidor recibe mensajes | 5 | Captura de consumo |
| Mensajes con clave | 5 | Productor con key:value |
| Consumidor muestra claves | 5 | print.key=true |
| **TOTAL** | **100** | |

---

## Bonus (opcional, hasta +10 puntos)

### Bonus 1: Consumer groups (+4 puntos)

Crea dos consumidores en el mismo consumer group:

Terminal 1:

```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --group my-consumer-group \
  --from-beginning
```

Terminal 2:

```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --group my-consumer-group \
  --from-beginning
```

Documenta cómo Kafka distribuye particiones entre ambos consumidores.

### Bonus 2: Kafka UI (+3 puntos)

Agrega [Kafka UI](https://github.com/provectus/kafka-ui) a tu `docker-compose.yml`:

```yaml
kafka-ui:
  image: provectuslabs/kafka-ui:latest
  container_name: kafka-ui
  ports:
    - "8090:8080"
  environment:
    KAFKA_CLUSTERS_0_NAME: local
    KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
```

Proporciona capturas de:

- Topics visualizados en UI
- Mensajes en un topic
- Consumer groups activos

### Bonus 3: Múltiples brokers (+3 puntos)

Modifica `docker-compose.yml` para tener 3 brokers de Kafka. Recrea los topics con `--replication-factor 3` y demuestra que las particiones tienen réplicas en diferentes brokers.

---

## Fecha de entrega

**[Fecha definida por el instructor]**

**Método de entrega**:

1. Subir código a repositorio Git privado
2. Compartir acceso con el instructor
3. Subir README.md con documentación y capturas a Moodle

---

## Recursos de ayuda

- [Labs de la Clase 4](../labs/)
- [Cheatsheet - Clase 4](../cheatsheet.md)
- [FAQ - Clase 4](../FAQ.md)
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kafka CLI Commands](https://kafka.apache.org/documentation/#quickstart)

---

**Última actualización**: Noviembre 2025
**Curso**: Spring Boot & Apache Kafka - i-Quattro
