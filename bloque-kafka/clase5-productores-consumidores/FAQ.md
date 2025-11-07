# FAQ - Clase 5: Productores y Consumidores con Spring Kafka

## Errores comunes y soluciones

### 1. Error: "Connection to node -1 could not be established"

**Problema**: Spring Boot no puede conectarse a Kafka.

**Solución**:
```bash
# Verificar que Kafka está corriendo
docker compose ps

# Ver logs de Kafka
docker compose logs kafka

# Reiniciar Kafka si es necesario
docker compose restart kafka
```

**Verificar configuración**:
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092  # Verificar puerto correcto
```

---

### 2. Error: "ClassCastException: cannot be cast to java.lang.String"

**Problema**: Deserializador esperaba String pero recibió otro tipo.

**Causa**: Configuración incorrecta de serializadores.

**Solución**:
```yaml
spring:
  kafka:
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

---

### 3. Error: "Topic 'ecommerce.products.created' does not exist"

**Problema**: El topic no fue creado antes de publicar.

**Solución**:
```bash
# Crear el topic
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 \
  --create --topic ecommerce.products.created \
  --partitions 3 --replication-factor 1

# Verificar
kafka-topics --bootstrap-server localhost:9092 --list
```

---

### 4. Error: "Port 8080 is already in use"

**Problema**: product-service y order-service no pueden usar el mismo puerto.

**Solución**:

**order-service application.yml**:
```yaml
server:
  port: 8081  # Puerto diferente
```

O ejecutar con:
```bash
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

---

### 5. Error: "Cannot construct instance of ProductCreatedEvent"

**Problema**: Evento no tiene constructor por defecto.

**Solución**:
```java
public class ProductCreatedEvent {
    // Constructor sin argumentos requerido para deserialización
    public ProductCreatedEvent() {}

    public ProductCreatedEvent(String productId, String name, BigDecimal price) {
        this.productId = productId;
        this.name = name;
        this.price = price;
    }

    // Getters y setters
}
```

---

### 6. Error: "Could not connect to database ecommerce_orders"

**Problema**: Base de datos para order-service no existe.

**Solución**:
```bash
# Entrar a PostgreSQL
docker exec -it postgres psql -U postgres

# Crear base de datos
CREATE DATABASE ecommerce_orders;

# Verificar
\l
```

---

### 7. Eventos no aparecen al consumir desde CLI

**Problema**: Consumidor comienza desde el offset actual.

**Solución**:
```bash
# Consumir desde el inicio
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning  # Importante
```

---

### 8. Error: "Timeout expired while fetching topic metadata"

**Problema**: Kafka no responde en el tiempo esperado.

**Causas posibles**:
1. Kafka no está corriendo
2. Firewall bloqueando puerto 9092
3. Configuración incorrecta de `advertised.listeners`

**Solución**:
```bash
# Verificar estado
docker compose ps

# Ver configuración de Kafka
docker exec -it kafka env | grep KAFKA

# Reiniciar stack completo
docker compose down
docker compose up -d
```

---

### 9. KafkaTemplate no se inyecta (NullPointerException)

**Problema**: Spring no encuentra el bean KafkaTemplate.

**Causa**: Falta dependencia spring-kafka.

**Solución - pom.xml**:
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

Luego:
```bash
mvn clean install
```

---

### 10. Eventos se publican pero no tienen el formato JSON esperado

**Problema**: Serialización incorrecta o falta toString().

**Verificar con CLI**:
```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning \
  --property print.key=true
```

**Solución**:
- Verificar que el evento tiene getters
- Asegurar que `JsonSerializer` está configurado
- Revisar logs de Spring Boot

---

## Preguntas frecuentes

### ¿Cuántas particiones debería tener mi topic?

**Regla general**: Empieza con 3-6 particiones para desarrollo.

**Consideraciones**:
- Más particiones = mayor paralelismo
- Pero más overhead de gestión
- Para producción, depende de throughput esperado

---

### ¿Qué diferencia hay entre clave (key) y valor (value)?

**Key**:
- Identifica el mensaje
- Determina a qué partición va
- Mensajes con misma key van a la misma partición (orden garantizado)

**Value**:
- El contenido del mensaje (tu evento)

**Ejemplo**:
```java
kafkaTemplate.send(topic, productId, event);
//                       ↑ key     ↑ value
```

---

### ¿Debo crear los topics manualmente o automáticamente?

**Desarrollo**: Creación automática está bien (si `auto.create.topics.enable=true`)

**Producción**: Siempre crear manualmente con:
- Número correcto de particiones
- Replication factor adecuado
- Configuración de retención

---

### ¿Puedo tener múltiples productores para el mismo topic?

**Sí**, múltiples microservicios pueden publicar al mismo topic.

**Ejemplo**:
- `product-service` publica `ecommerce.products.created`
- `admin-service` también podría publicar al mismo topic

---

### ¿Cómo sé si mi evento se publicó exitosamente?

**Opción 1 - Logs**:
```java
log.info("Event published successfully");
```

**Opción 2 - Callback**:
```java
kafkaTemplate.send(topic, key, event)
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("Failed to publish", ex);
        } else {
            log.info("Published to partition {}",
                     result.getRecordMetadata().partition());
        }
    });
```

**Opción 3 - Consumer desde CLI**:
```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning
```

---

### ¿Por qué usar eventos en lugar de llamadas REST entre microservicios?

**Ventajas de eventos**:
1. **Desacoplamiento**: Servicios no se conocen directamente
2. **Resiliencia**: Si un servicio está caído, el mensaje espera
3. **Escalabilidad**: Múltiples consumidores pueden procesar en paralelo
4. **Auditoría**: Los eventos quedan registrados en Kafka

**Cuándo usar REST**:
- Necesitas respuesta inmediata
- Operaciones síncronas (ej: login)

**Cuándo usar eventos**:
- Notificaciones
- Actualizaciones asíncronas
- Integración entre bounded contexts

---

### ¿Qué pasa si publico un evento y luego falla la transacción de base de datos?

**Problema**: Evento publicado pero cambio en BD revertido = inconsistencia.

**Solución (avanzado - no en este curso)**:
- Transactional Outbox Pattern
- Kafka Transactions

**Solución simple (este curso)**:
1. Guardar en BD primero
2. Si tiene éxito, publicar evento
3. Si publicar falla, loguear error (el cambio en BD ya está)

---

### ¿Puedo ver los eventos publicados en alguna interfaz gráfica?

**Sí**, opciones:

1. **Kafka UI** (recomendado para este curso):
```yaml
# Agregar a docker-compose.yml
kafka-ui:
  image: provectuslabs/kafka-ui:latest
  ports:
    - "8090:8080"
  environment:
    KAFKA_CLUSTERS_0_NAME: local
    KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

2. **Conduktor** (comercial, tiene free tier)
3. **Kafdrop** (open source)

---

## Troubleshooting avanzado

### Ver offset actual de un topic

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group my-group \
  --describe
```

### Ver configuración de un topic

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name ecommerce.products.created \
  --describe
```

### Limpiar todos los mensajes de un topic (cuidado)

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --delete \
  --topic ecommerce.products.created

# Recrear
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.products.created \
  --partitions 3 \
  --replication-factor 1
```

---

## Recursos adicionales

- [Spring Kafka Documentation](https://docs.spring.io/spring-kafka/reference/html/)
- [Kafka Error Handling](https://www.baeldung.com/spring-kafka-error-handling)
- [Event-Driven Microservices](https://www.confluent.io/blog/event-driven-microservices-with-spring-boot-and-kafka/)
