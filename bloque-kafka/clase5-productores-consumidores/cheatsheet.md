# Cheatsheet - Clase 5: Productores y Consumidores con Spring Kafka

## Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

---

## Configuración Producer (application.yml)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

---

## Crear un Evento de Dominio

```java
package dev.alefiengo.productservice.model.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class ProductCreatedEvent {
    private String productId;
    private String name;
    private BigDecimal price;
    private LocalDateTime createdAt;

    // Constructor, getters, setters
}
```

---

## Producer con KafkaTemplate

```java
package dev.alefiengo.productservice.kafka.producer;

import dev.alefiengo.productservice.model.dto.ProductCreatedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class ProductEventProducer {

    private static final Logger log = LoggerFactory.getLogger(ProductEventProducer.class);
    private static final String TOPIC = "ecommerce.products.created";

    private final KafkaTemplate<String, ProductCreatedEvent> kafkaTemplate;

    public ProductEventProducer(KafkaTemplate<String, ProductCreatedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publishProductCreated(ProductCreatedEvent event) {
        log.info("Publishing ProductCreatedEvent: productId={}", event.getProductId());
        kafkaTemplate.send(TOPIC, event.getProductId(), event);
        log.info("ProductCreatedEvent published successfully");
    }
}
```

---

## Integrar Producer en Service

```java
@Service
public class ProductService {

    private final ProductRepository repository;
    private final ProductEventProducer eventProducer;

    public ProductService(ProductRepository repository,
                          ProductEventProducer eventProducer) {
        this.repository = repository;
        this.eventProducer = eventProducer;
    }

    public ProductResponse createProduct(ProductRequest request) {
        // 1. Guardar en base de datos
        Product product = repository.save(toEntity(request));

        // 2. Publicar evento
        ProductCreatedEvent event = new ProductCreatedEvent(
            product.getId().toString(),
            product.getName(),
            product.getPrice(),
            LocalDateTime.now()
        );
        eventProducer.publishProductCreated(event);

        // 3. Retornar respuesta
        return toResponse(product);
    }
}
```

---

## Crear Topic desde CLI

```bash
# Entrar al contenedor de Kafka
docker exec -it kafka bash

# Crear topic
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.products.created \
  --partitions 3 \
  --replication-factor 1

# Verificar topic creado
kafka-topics --bootstrap-server localhost:9092 --list

# Describir topic
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic ecommerce.products.created
```

---

## Consumir mensajes desde CLI (Testing)

```bash
# Consumir desde el inicio
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning

# Consumir con key y timestamp
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning \
  --property print.key=true \
  --property print.timestamp=true
```

---

## Crear nuevo proyecto Spring Boot

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,kafka \
  -d groupId=dev.alefiengo \
  -d artifactId=order-service \
  -d name=OrderService \
  -d packageName=dev.alefiengo.orderservice \
  -d javaVersion=17 \
  -o order-service.zip

unzip order-service.zip
cd order-service
```

---

## application.yml para order-service

```yaml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce_orders
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

server:
  port: 8081
```

---

## Comandos útiles

```bash
# Ejecutar order-service
cd order-service
mvn spring-boot:run

# Ejecutar en puerto específico
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081

# Ver logs de Kafka
docker compose logs -f kafka

# Reiniciar Kafka
docker compose restart kafka
```

---

## Testing con curl

```bash
# Crear producto (product-service en puerto 8080)
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop Dell XPS",
    "description": "High-end laptop",
    "price": 1299.99,
    "categoryId": 1
  }'

# Crear orden (order-service en puerto 8081)
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "CUST-123",
    "items": [
      {"productId": "1", "quantity": 2}
    ]
  }'
```

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Productor | Producer | Publica mensajes a Kafka |
| Plantilla Kafka | KafkaTemplate | Bean de Spring para enviar mensajes |
| Serializador | Serializer | Convierte objeto a bytes |
| Evento de dominio | Domain Event | Hecho inmutable del sistema |
| Clave del mensaje | Message Key | Identifica el mensaje, determina partición |
