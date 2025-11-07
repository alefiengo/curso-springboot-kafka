# Conceptos - Clase 5: Productores y Consumidores con Spring Kafka

## Introducción

En esta clase integramos Apache Kafka con Spring Boot usando **spring-kafka**, el proyecto oficial que simplifica la interacción con Kafka desde aplicaciones Spring. Aprenderemos a publicar eventos desde microservicios usando productores.

---

## Spring Kafka

### ¿Qué es spring-kafka?

**Spring Kafka** es el proyecto oficial de Spring para trabajar con Apache Kafka. Proporciona abstracciones de alto nivel que simplifican:

- Configuración de productores y consumidores
- Serialización y deserialización automática
- Manejo de errores y reintentos
- Integración con el contenedor de Spring

### Ventajas de usar spring-kafka

1. **Configuración simplificada**: Usa application.yml en lugar de Properties
2. **Inyección de dependencias**: KafkaTemplate como bean de Spring
3. **Anotaciones declarativas**: @KafkaListener para consumidores
4. **Serialización JSON automática**: Convierte objetos Java a JSON
5. **Manejo de transacciones**: Soporte para transacciones Kafka
6. **Testing**: Herramientas para testing con Kafka embebido

---

## Productores de Kafka

### KafkaTemplate

`KafkaTemplate` es la clase principal de spring-kafka para enviar mensajes. Es análogo a JdbcTemplate o RestTemplate en otros componentes de Spring.

**Características**:
- Thread-safe (puede usarse en múltiples hilos)
- Soporta envío síncrono y asíncrono
- Permite especificar topic, key y value
- Retorna CompletableFuture con resultado del envío

### Anatomía de un Mensaje Kafka

Un mensaje en Kafka tiene tres componentes principales:

1. **Key (Clave)**: Identificador del mensaje, determina la partición
2. **Value (Valor)**: El contenido del mensaje (nuestro evento)
3. **Headers (Opcional)**: Metadatos adicionales

### Serialización

**Serialización** es el proceso de convertir un objeto Java en bytes para enviar por la red.

**Opciones comunes**:
- **StringSerializer**: Para strings simples
- **JsonSerializer**: Para objetos Java → JSON (el que usaremos)
- **AvroSerializer**: Para esquemas Avro (más eficiente, no lo veremos)

En este curso usamos **JsonSerializer** porque es simple y legible.

---

## Eventos de Dominio

### ¿Qué es un Evento de Dominio?

Un **evento de dominio** representa algo que ya ocurrió en el sistema. Es un hecho inmutable del pasado.

**Características**:
- Nombre en pasado: `OrderPlaced`, `ProductCreated`, `PaymentProcessed`
- Inmutable: Una vez creado, no se modifica
- Contiene toda la información relevante del evento
- Timestamp del momento en que ocurrió

### Diseño de Eventos

**Buenas prácticas**:

1. **Incluir identificadores**: `orderId`, `productId`, `customerId`
2. **Timestamp**: Fecha y hora del evento
3. **Información completa**: Todo lo necesario para procesar el evento
4. **Versión del evento**: Para evolución futura (opcional)

**Ejemplo de evento bien diseñado**:

```java
public class OrderPlacedEvent {
    private String orderId;
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private LocalDateTime placedAt;
}
```

---

## Arquitectura Producer en Spring Boot

### Flujo de Publicación de Eventos

```
[Controller] → [Service] → [EventProducer] → [KafkaTemplate] → [Kafka Broker]
```

1. **Controller**: Recibe la petición HTTP
2. **Service**: Procesa la lógica de negocio (ej: guardar orden en BD)
3. **EventProducer**: Crea el evento y lo publica
4. **KafkaTemplate**: Serializa y envía a Kafka
5. **Kafka Broker**: Almacena el mensaje en el topic

### Patrón de Diseño

Usamos el patrón **Event Publisher** para separar responsabilidades:

```java
@Service
public class OrderService {
    private final OrderRepository repository;
    private final OrderEventProducer eventProducer;

    public Order createOrder(OrderRequest request) {
        // 1. Guardar en base de datos
        Order order = repository.save(toEntity(request));

        // 2. Publicar evento
        eventProducer.publishOrderPlaced(order);

        return order;
    }
}
```

### Configuración de Producer

En `application.yml` configuramos:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

**Explicación**:
- `bootstrap-servers`: Dirección del broker de Kafka
- `key-serializer`: Cómo serializar la clave (String)
- `value-serializer`: Cómo serializar el valor (JSON)

---

## Topics de E-commerce

En esta clase crearemos dos topics para nuestro dominio de e-commerce:

### ecommerce.products.created

**Propósito**: Notificar cuando se crea un nuevo producto en el catálogo

**Evento**: `ProductCreatedEvent`
```java
{
  "productId": "123",
  "name": "Laptop Dell XPS",
  "price": 1299.99,
  "createdAt": "2025-11-07T10:30:00"
}
```

**Productores**: product-service
**Consumidores futuros**: analytics-service, search-service

### ecommerce.orders.placed

**Propósito**: Notificar cuando un cliente realiza una orden

**Evento**: `OrderPlacedEvent`
```java
{
  "orderId": "ORD-001",
  "customerId": "CUST-123",
  "items": [
    {"productId": "123", "quantity": 2}
  ],
  "totalAmount": 2599.98,
  "placedAt": "2025-11-07T10:35:00"
}
```

**Productores**: order-service
**Consumidores futuros**: inventory-service, notification-service

---

## Microservicios en esta Clase

### product-service (Existente)

**Modificaciones**:
- Agregar dependencia spring-kafka
- Crear ProductCreatedEvent
- Crear ProductEventProducer
- Modificar ProductService para publicar eventos

### order-service (Nuevo)

**Creación desde cero**:
- Proyecto Spring Boot con web, data-jpa, kafka, postgresql
- Entidad Order y OrderItem
- OrderRepository
- OrderService con lógica de negocio
- OrderController para REST API
- OrderEventProducer para publicar eventos

---

## Comunicación Asíncrona

### Síncrona vs Asíncrona

**Comunicación Síncrona** (REST):
- Cliente espera respuesta inmediata
- Acoplamiento fuerte entre servicios
- Si el servicio destino está caído, falla la operación

**Comunicación Asíncrona** (Kafka):
- Cliente no espera respuesta
- Desacoplamiento entre servicios
- Si un consumidor está caído, el mensaje espera
- Mayor resiliencia del sistema

### Eventual Consistency

Al usar Kafka, aceptamos **consistencia eventual**:

1. Orden se guarda en order-service (consistente)
2. Se publica evento a Kafka (puede tardar milisegundos)
3. inventory-service lee el evento (puede tardar segundos)
4. Eventualmente, todos los servicios reflejan el cambio

**Trade-off**: Sacrificamos consistencia inmediata por disponibilidad y escalabilidad.

---

## Logging y Observabilidad

### Logging de Eventos

Es crítico loguear cuando se publican eventos:

```java
log.info("Publishing ProductCreatedEvent: productId={}", event.getProductId());
kafkaTemplate.send(topic, key, event);
log.info("ProductCreatedEvent published successfully");
```

**Razones**:
- Debugging: Rastrear flujo de eventos
- Auditoría: Registro de qué se publicó y cuándo
- Troubleshooting: Identificar eventos perdidos

### Manejo de Errores

KafkaTemplate retorna un `CompletableFuture`. Podemos manejar errores:

```java
kafkaTemplate.send(topic, key, event)
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("Failed to publish event", ex);
        } else {
            log.info("Event published to partition {}",
                     result.getRecordMetadata().partition());
        }
    });
```

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Productor | Producer | Aplicación que publica mensajes a Kafka |
| Consumidor | Consumer | Aplicación que lee mensajes de Kafka |
| Evento de dominio | Domain Event | Hecho inmutable que ocurrió en el sistema |
| Serialización | Serialization | Convertir objeto Java a bytes |
| Deserialización | Deserialization | Convertir bytes a objeto Java |
| Consistencia eventual | Eventual Consistency | Sistema alcanza consistencia después de un tiempo |
| Asíncrono | Asynchronous | Operación que no bloquea esperando respuesta |
