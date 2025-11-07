# Lab 01: Producer en product-service

## Objetivo

Implementar un productor de eventos Kafka en product-service que publique eventos ProductCreatedEvent al topic ecommerce.products.created cada vez que se crea un producto, integrando KafkaTemplate con la capa de servicio.

---

## Comandos a ejecutar

```bash
# 1. Verificar que Kafka está corriendo
docker compose ps

# 2. Crear el topic ecommerce.products.created
docker exec -it kafka bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.products.created \
  --partitions 3 \
  --replication-factor 1

# 3. Verificar topic creado
kafka-topics --bootstrap-server localhost:9092 --list

# 4. Salir del contenedor
exit

# 5. Navegar al proyecto product-service
cd product-service

# 6. Crear estructura de paquetes
mkdir -p src/main/java/dev/alefiengo/productservice/kafka/event
mkdir -p src/main/java/dev/alefiengo/productservice/kafka/producer

# 7. Crear clases Java (ver sección "Desglose del comando")
# - ProductCreatedEvent.java
# - ProductEventProducer.java

# 8. Modificar ProductService.java para publicar eventos

# 9. Ejecutar aplicación
mvn spring-boot:run

# 10. En otra terminal, consumir desde CLI para verificar
docker exec -it kafka bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning \
  --property print.key=true

# 11. Crear un producto vía REST
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop Dell XPS 15",
    "description": "Laptop profesional con procesador Intel i7",
    "price": 1500.00,
    "categoryId": 1
  }'

# Deberías ver el evento en el consumer CLI
```

---

## Desglose del comando

### 1. Crear Topic ecommerce.products.created

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.products.created \
  --partitions 3 \
  --replication-factor 1
```

| Flag | Valor | Descripción |
|------|-------|-------------|
| `--bootstrap-server` | `localhost:9092` | Dirección del broker |
| `--create` | - | Indica creación de topic |
| `--topic` | `ecommerce.products.created` | Nombre del topic (naming convention) |
| `--partitions` | `3` | Número de particiones para paralelismo |
| `--replication-factor` | `1` | Réplicas (1 para desarrollo, 3 para producción) |

**Naming convention**: `app.domain.event-type`

- `ecommerce`: Aplicación
- `products`: Dominio
- `created`: Evento (pasado)

### 2. Estructura del Evento (ProductCreatedEvent.java)

```java
package dev.alefiengo.productservice.kafka.event;

import java.math.BigDecimal;
import java.time.Instant;

public class ProductCreatedEvent {

    private Long productId;
    private String name;
    private String description;
    private BigDecimal price;
    private Long categoryId;
    private Instant timestamp;

    public ProductCreatedEvent() {
        // Constructor sin argumentos requerido para deserialización JSON
    }

    public ProductCreatedEvent(Long productId, String name, String description,
                               BigDecimal price, Long categoryId) {
        this.productId = productId;
        this.name = name;
        this.description = description;
        this.price = price;
        this.categoryId = categoryId;
        this.timestamp = Instant.now();
    }

    // Getters y Setters
    public Long getProductId() { return productId; }
    public void setProductId(Long productId) { this.productId = productId; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }

    public Long getCategoryId() { return categoryId; }
    public void setCategoryId(Long categoryId) { this.categoryId = categoryId; }

    public Instant getTimestamp() { return timestamp; }
    public void setTimestamp(Instant timestamp) { this.timestamp = timestamp; }

    @Override
    public String toString() {
        return "ProductCreatedEvent{" +
                "productId=" + productId +
                ", name='" + name + '\'' +
                ", price=" + price +
                ", timestamp=" + timestamp +
                '}';
    }
}
```

**Características del evento**:

- **POJO**: Objeto Java simple sin anotaciones especiales
- **Constructor vacío**: Requerido para deserialización JSON
- **Inmutabilidad parcial**: timestamp se asigna en construcción
- **toString()**: Útil para logging

### 3. Producer (ProductEventProducer.java)

```java
package dev.alefiengo.productservice.kafka.producer;

import dev.alefiengo.productservice.kafka.event.ProductCreatedEvent;
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
        String key = event.getProductId().toString();

        log.info("Publishing ProductCreatedEvent: productId={}, name={}",
                 event.getProductId(), event.getName());

        kafkaTemplate.send(TOPIC, key, event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish ProductCreatedEvent: productId={}",
                              event.getProductId(), ex);
                } else {
                    log.info("ProductCreatedEvent published successfully: productId={}, partition={}",
                             event.getProductId(),
                             result.getRecordMetadata().partition());
                }
            });
    }
}
```

**Elementos clave**:

| Elemento | Descripción |
|----------|-------------|
| `@Component` | Registra la clase como bean de Spring |
| `KafkaTemplate<String, ProductCreatedEvent>` | String = tipo de key, ProductCreatedEvent = tipo de value |
| `TOPIC` | Constante con nombre del topic |
| `key` | productId convertido a String, determina partición |
| `whenComplete()` | Callback asíncrono para manejar resultado o error |

### 4. Integración en ProductService

Modificar `ProductService.java`:

```java
package dev.alefiengo.productservice.service;

import dev.alefiengo.productservice.kafka.event.ProductCreatedEvent;
import dev.alefiengo.productservice.kafka.producer.ProductEventProducer;
import dev.alefiengo.productservice.model.dto.ProductRequest;
import dev.alefiengo.productservice.model.dto.ProductResponse;
import dev.alefiengo.productservice.model.entity.Category;
import dev.alefiengo.productservice.model.entity.Product;
import dev.alefiengo.productservice.repository.CategoryRepository;
import dev.alefiengo.productservice.repository.ProductRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductEventProducer productEventProducer; // Nuevo

    public ProductService(ProductRepository productRepository,
                          CategoryRepository categoryRepository,
                          ProductEventProducer productEventProducer) { // Nuevo
        this.productRepository = productRepository;
        this.categoryRepository = categoryRepository;
        this.productEventProducer = productEventProducer; // Nuevo
    }

    @Transactional
    public ProductResponse create(ProductRequest request) {
        // 1. Validar que existe la categoría
        Category category = categoryRepository.findById(request.getCategoryId())
            .orElseThrow(() -> new RuntimeException("Category not found"));

        // 2. Crear y guardar producto
        Product product = new Product();
        product.setName(request.getName());
        product.setDescription(request.getDescription());
        product.setPrice(request.getPrice());
        product.setCategory(category);

        Product savedProduct = productRepository.save(product);

        // 3. Publicar evento a Kafka
        ProductCreatedEvent event = new ProductCreatedEvent(
            savedProduct.getId(),
            savedProduct.getName(),
            savedProduct.getDescription(),
            savedProduct.getPrice(),
            savedProduct.getCategory().getId()
        );
        productEventProducer.publishProductCreated(event);

        // 4. Retornar respuesta
        return mapToResponse(savedProduct);
    }

    // Resto de métodos...
}
```

---

## Explicación detallada

### Paso 1: ¿Por qué crear un topic antes de publicar?

Aunque Kafka puede crear topics automáticamente, es mejor crearlos manualmente porque:

1. **Control de particiones**: Decidir cuántas particiones según carga esperada
2. **Control de replicación**: Configurar replication factor para durabilidad
3. **Configuración explícita**: Retención, compresión, etc.
4. **Evitar errores**: Si `auto.create.topics.enable=false`, publicar a topic inexistente falla

**En producción**: SIEMPRE crear topics manualmente con configuración adecuada.

### Paso 2: Diseño del Evento

**ProductCreatedEvent** representa algo que **ya ocurrió**:

```java
public class ProductCreatedEvent {
    private Long productId;      // ID del producto creado
    private String name;         // Nombre del producto
    private BigDecimal price;    // Precio
    private Instant timestamp;   // Cuándo ocurrió
}
```

**Características de un buen evento**:

- **Pasado**: `ProductCreated` (no `CreateProduct` - eso sería comando)
- **Inmutable**: Una vez creado, no cambia (timestamp fijo)
- **Completo**: Contiene toda la información relevante
- **Serializable**: JSON-friendly (getters/setters, constructor vacío)

**¿Por qué no enviar la entidad Product directamente?**

```java
// ❌ MAL - No hacer esto
kafkaTemplate.send(TOPIC, product); // Product es entidad JPA

// ✅ BIEN - Usar evento dedicado
kafkaTemplate.send(TOPIC, event);  // ProductCreatedEvent es POJO
```

**Razones**:

1. **Separación de concerns**: Entidad es modelo de BD, evento es modelo de mensajería
2. **Evitar acoplamiento**: Cambios en BD no afectan estructura de eventos
3. **Control de información**: Evento solo incluye campos necesarios (no toda la entidad)
4. **Evitar lazy loading**: Entidades JPA pueden tener relaciones lazy que fallan al serializar

### Paso 3: KafkaTemplate - Anatomía del envío

```java
kafkaTemplate.send(TOPIC, key, event)
//             send(topic, key, value)
```

**Parámetros**:

1. **topic**: `"ecommerce.products.created"` - A dónde va el mensaje
2. **key**: `productId.toString()` - Determina partición
3. **value**: `event` - El contenido (ProductCreatedEvent)

**¿Cómo funciona el particionamiento?**

```
Kafka calcula: partition = hash(key) % num_partitions

Ejemplo con 3 particiones:
- productId=1 → hash(1) % 3 = partition 1
- productId=2 → hash(2) % 3 = partition 2
- productId=3 → hash(3) % 3 = partition 0
- productId=4 → hash(4) % 3 = partition 1
```

**Ventaja de usar productId como key**:

- Todos los eventos del mismo producto van a la misma partición
- Orden garantizado de eventos por producto
- Útil si después agregas `ProductUpdatedEvent`, `ProductDeletedEvent`

### Paso 4: Manejo asíncrono con whenComplete()

```java
kafkaTemplate.send(TOPIC, key, event)
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("Failed to publish", ex);
        } else {
            log.info("Published to partition {}",
                     result.getRecordMetadata().partition());
        }
    });
```

**¿Por qué asíncrono?**

`kafkaTemplate.send()` retorna `CompletableFuture<SendResult>`:

- **No bloquea**: El método ProductService.create() retorna inmediatamente
- **Performance**: No espera confirmación de Kafka para responder al cliente
- **Callback**: `whenComplete()` se ejecuta cuando Kafka confirma (o falla)

**Alternativa síncrona (NO recomendado en producción)**:

```java
// Bloquea hasta que Kafka confirma
SendResult result = kafkaTemplate.send(TOPIC, key, event).get();
```

**Trade-off**:

- **Asíncrono**: Más rápido, pero si Kafka falla el cliente no se entera
- **Síncrono**: Más lento, pero garantiza que mensaje llegó antes de responder

**En este curso usamos asíncrono** para simular arquitectura event-driven real.

### Paso 5: Integración en ProductService

**Flujo completo**:

```
1. Cliente → POST /api/products
2. ProductController recibe request
3. ProductService.create():
   a. Valida categoría (BD)
   b. Crea producto (BD)
   c. Guarda en BD (commit transaction)
   d. Publica evento (Kafka) ← NUEVO
4. ProductController retorna response
5. Asíncronamente: Kafka confirma recepción
```

**Orden importante**:

1. Primero guardar en BD
2. Luego publicar evento

**¿Por qué este orden?**

Si publicamos evento antes de guardar:

```java
// ❌ MAL
productEventProducer.publishProductCreated(event); // Evento publicado
Product saved = productRepository.save(product);    // Si falla, evento ya publicado!
```

Problema: Evento dice "producto creado" pero no existe en BD.

```java
// ✅ BIEN
Product saved = productRepository.save(product);    // Primero guardar
productEventProducer.publishProductCreated(event); // Luego publicar
```

Si falla publicar evento: Producto existe en BD, pero nadie fue notificado (menos grave).

**Nota avanzada**: Para garantía total usar **Transactional Outbox Pattern** (fuera del alcance de este curso).

### Paso 6: Verificación con kafka-console-consumer

```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning \
  --property print.key=true
```

**Salida esperada**:

```
1	{"productId":1,"name":"Laptop Dell XPS 15","description":"Laptop profesional...","price":1500.00,"categoryId":1,"timestamp":"2025-11-07T15:30:45.123Z"}
```

**Formato**: `key<TAB>value`

- **key**: `1` (productId)
- **value**: JSON del evento

**Si no ves el evento**:

1. Verificar que product-service está corriendo
2. Verificar logs: `Publishing ProductCreatedEvent: productId=1`
3. Verificar topic existe: `kafka-topics --list`
4. Verificar Kafka está accesible: `docker compose ps`

---

## Conceptos aprendidos

- **Eventos de dominio** representan hechos que ya ocurrieron, usando tiempo pasado (ProductCreated)
- **Separación de modelos**: Entidades JPA vs Eventos de Kafka son diferentes
- **KafkaTemplate** envía mensajes con formato `send(topic, key, value)`
- **Key de particionamiento** determina a qué partición va el mensaje (hash(key) % partitions)
- **Serialización JSON automática** convierte eventos Java a JSON sin código adicional
- **Envío asíncrono** con `whenComplete()` no bloquea el flujo principal
- **Orden de operaciones**: Primero persistir en BD, luego publicar evento
- **Logging de eventos** facilita debugging y auditoría
- Topics deben crearse antes de publicar (buena práctica)
- **kafka-console-consumer** es útil para verificar eventos publicados desde CLI

---

## Troubleshooting

### Problema 1: Error "Topic does not exist"

**Síntoma**:

```
ERROR o.a.k.clients.producer.internals.Sender : Topic ecommerce.products.created not present in metadata
```

**Causa**: Topic no fue creado antes de publicar.

**Solución**:

```bash
docker exec -it kafka bash
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic ecommerce.products.created \
  --partitions 3 --replication-factor 1
```

### Problema 2: Evento no se serializa (JSON vacío)

**Síntoma**: Consumer muestra `{}` en lugar del evento completo.

**Causa**: Falta constructor vacío o getters en ProductCreatedEvent.

**Solución**:

```java
public class ProductCreatedEvent {
    // ✅ DEBE tener constructor vacío
    public ProductCreatedEvent() {}

    // ✅ DEBE tener getters
    public Long getProductId() { return productId; }
    // ...
}
```

### Problema 3: NullPointerException en ProductEventProducer

**Síntoma**:

```
NullPointerException at ProductEventProducer.publishProductCreated
```

**Causa**: ProductEventProducer no está siendo inyectado como bean.

**Solución**: Verificar anotación `@Component`:

```java
@Component // ← IMPORTANTE
public class ProductEventProducer {
    // ...
}
```

### Problema 4: Eventos se publican múltiples veces

**Síntoma**: Un solo producto crea múltiples eventos en Kafka.

**Causa**: Reintentos automáticos del producer por configuración.

**Solución temporal** (para desarrollo):

```yaml
spring:
  kafka:
    producer:
      retries: 0  # Deshabilitar reintentos en desarrollo
```

**Solución producción**: Diseñar consumidores para ser idempotentes.

### Problema 5: Error de transacción al publicar evento

**Síntoma**:

```
Transaction rolled back because it has been marked as rollback-only
```

**Causa**: Si publicar falla, Spring hace rollback de toda la transacción incluyendo el save().

**Solución**: Mover publicación fuera de transacción o usar `@TransactionalEventListener`:

```java
@Transactional
public ProductResponse create(ProductRequest request) {
    Product saved = productRepository.save(product);
    // Publicar evento DESPUÉS del commit
    return mapToResponse(saved);
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleProductCreated(ProductCreatedEvent event) {
    productEventProducer.publishProductCreated(event);
}
```

**Nota**: Para este lab, mantener el código simple (publicar dentro del método).

---

## Desafío adicional

### Desafío 1: Agregar metadata al evento

Agrega información adicional al evento:

```java
public class ProductCreatedEvent {
    // Campos existentes...
    private String createdBy;      // Usuario que creó el producto
    private String source;         // "WEB", "API", "ADMIN"

    // Getters y setters
}
```

Modifica ProductService para incluir esta información.

### Desafío 2: Implementar ProductUpdatedEvent

Crea un segundo evento `ProductUpdatedEvent` y publícalo cuando se actualiza un producto:

1. Crear `ProductUpdatedEvent.java` en package `kafka.event`
2. Agregar método `publishProductUpdated()` en `ProductEventProducer`
3. Crear topic `ecommerce.products.updated`
4. Modificar `ProductService.update()` para publicar evento

Verifica que ambos eventos se publican correctamente.

### Desafío 3: Agregar headers personalizados

Agrega headers al mensaje Kafka:

```java
public void publishProductCreated(ProductCreatedEvent event) {
    ProducerRecord<String, ProductCreatedEvent> record =
        new ProducerRecord<>(TOPIC, key, event);

    record.headers().add("event-type", "ProductCreated".getBytes());
    record.headers().add("timestamp", String.valueOf(System.currentTimeMillis()).getBytes());

    kafkaTemplate.send(record);
}
```

Verifica headers desde consumer CLI:

```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning \
  --property print.headers=true
```

---

## Recursos adicionales

- [KafkaTemplate Documentation](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/core/KafkaTemplate.html)
- [Event-Driven Microservices](https://www.confluent.io/blog/event-driven-microservices-with-spring-boot-and-kafka/)
- [Domain Events Pattern](https://martinfowler.com/eaaDev/DomainEvent.html)
- [Kafka Partitioning Strategy](https://www.confluent.io/blog/apache-kafka-producer-improvements-sticky-partitioner/)
- [Spring Kafka JsonSerializer](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/support/serializer/JsonSerializer.html)
