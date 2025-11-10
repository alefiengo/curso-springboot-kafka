# Lab 02: Crear order-service

## Objetivo

Crear un segundo microservicio (order-service) desde cero usando Spring Initializr, configurar base de datos PostgreSQL independiente, implementar entidad Order con CRUD básico y preparar la estructura para integrar Kafka en el siguiente lab.

---

## Comandos a ejecutar

```bash
# 1. Navegar al directorio de trabajo
cd ~/workspace

# 2. Generar proyecto con Spring Initializr (desde línea de comandos)
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,validation \
  -d groupId=dev.alefiengo \
  -d artifactId=order-service \
  -d name=OrderService \
  -d description="Microservicio de gestión de órdenes para e-commerce" \
  -d packageName=dev.alefiengo.orderservice \
  -d javaVersion=17 \
  -d type=maven-project \
  -o order-service.zip

# 3. Descomprimir y entrar al directorio
unzip order-service.zip
cd order-service

# 4. Crear base de datos en PostgreSQL
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_orders;"

# 5. Verificar base de datos creada
docker exec -it postgres psql -U postgres -c "\l"

# 6. Configurar application.yml (ver sección "Desglose del comando")

# 7. Crear estructura de paquetes
mkdir -p src/main/java/dev/alefiengo/orderservice/model/entity
mkdir -p src/main/java/dev/alefiengo/orderservice/model/dto
mkdir -p src/main/java/dev/alefiengo/orderservice/repository
mkdir -p src/main/java/dev/alefiengo/orderservice/service
mkdir -p src/main/java/dev/alefiengo/orderservice/controller

# 8. Crear clases Java (ver sección "Desglose del comando")

# 9. Compilar y ejecutar en puerto 8081
mvn clean install
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081

# 10. Verificar que funciona
curl http://localhost:8081/api/orders

# 11. Crear una orden de prueba
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "quantity": 2,
    "customerName": "Juan Perez",
    "customerEmail": "juan@example.com"
  }'
```

---

## Desglose del comando

### 1. Spring Initializr - Generación del proyecto

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,validation \
  -d groupId=dev.alefiengo \
  -d artifactId=order-service \
  -d javaVersion=17
```

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `dependencies` | `web,data-jpa,postgresql,validation` | Spring Web, JPA, PostgreSQL Driver, Validation |
| `groupId` | `dev.alefiengo` | Identificador de organización |
| `artifactId` | `order-service` | Nombre del proyecto |
| `packageName` | `dev.alefiengo.orderservice` | Paquete base Java |
| `javaVersion` | `17` | Versión de Java LTS |
| `type` | `maven-project` | Build tool Maven |

**Alternativa**: Usar [start.spring.io](https://start.spring.io) en navegador.

### 2. Crear base de datos independiente

```bash
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_orders;"
```

**¿Por qué base de datos independiente?**

Arquitectura de microservicios: Cada microservicio tiene su propia base de datos.

```
product-service → ecommerce (PostgreSQL)
order-service   → ecommerce_orders (PostgreSQL)
```

**Ventajas**:

- Autonomía: order-service no depende de product-service
- Escalabilidad: Cada BD puede escalarse independientemente
- Resiliencia: Falla de una BD no afecta otros servicios

### 3. Configuración application.yml

Crear `src/main/resources/application.yml`:

```yaml
spring:
  application:
    name: order-service

  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce_orders}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

server:
  port: ${SERVER_PORT:8081}

logging:
  level:
    dev.alefiengo.orderservice: DEBUG
    org.springframework.web: DEBUG
```

**Diferencias con product-service**:

| Propiedad | product-service | order-service |
|-----------|----------------|---------------|
| `spring.application.name` | `product-service` | `order-service` |
| `datasource.url` | `ecommerce` | `ecommerce_orders` |
| `server.port` | `8080` | `8081` |

**¿Por qué puerto diferente?**

Ambos servicios corren en la misma máquina (localhost) durante desarrollo. Usar puertos diferentes evita conflictos.

### 4. Entidad Order

Crear `src/main/java/dev/alefiengo/orderservice/model/entity/Order.java`:

```java
package dev.alefiengo.orderservice.model.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "customer_orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long productId;

    @Column(nullable = false)
    private Integer quantity;

    @Column(nullable = false)
    private String customerName;

    @Column(nullable = false)
    private String customerEmail;

    @Column(nullable = false)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
        if (this.status == null) {
            this.status = OrderStatus.PENDING;
        }
    }

    // Getters y Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public Long getProductId() { return productId; }
    public void setProductId(Long productId) { this.productId = productId; }

    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }

    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }

    public String getCustomerEmail() { return customerEmail; }
    public void setCustomerEmail(String customerEmail) { this.customerEmail = customerEmail; }

    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }

    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
}
```

**Enum OrderStatus** (crear archivo separado):

```java
package dev.alefiengo.orderservice.model.entity;

public enum OrderStatus {
    PENDING,      // Orden creada, esperando validación
    CONFIRMED,    // Orden confirmada por inventory-service
    CANCELLED     // Orden cancelada (sin stock)
}
```

**Campos importantes**:

- `productId`: Referencia al producto (NO es FK, es solo ID)
- `status`: Estado de la orden (PENDING por defecto)
- `createdAt`: Timestamp de creación automático

### 5. DTOs (Request y Response)

**OrderRequest.java**:

```java
package dev.alefiengo.orderservice.model.dto;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public class OrderRequest {

    @NotNull(message = "El ID del producto es requerido")
    @Positive(message = "El ID del producto debe ser positivo")
    private Long productId;

    @NotNull(message = "La cantidad es requerida")
    @Positive(message = "La cantidad debe ser positiva")
    @Max(value = 100, message = "La cantidad no puede exceder 100")
    private Integer quantity;

    @NotBlank(message = "El nombre del cliente es requerido")
    @Size(min = 3, max = 100, message = "El nombre debe tener entre 3 y 100 caracteres")
    private String customerName;

    @NotBlank(message = "El email del cliente es requerido")
    @Email(message = "El email debe ser válido")
    private String customerEmail;

    @NotNull(message = "El monto total es requerido")
    @Positive(message = "El monto total debe ser positivo")
    private BigDecimal totalAmount;

    // Getters y Setters
    public Long getProductId() { return productId; }
    public void setProductId(Long productId) { this.productId = productId; }

    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }

    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }

    public String getCustomerEmail() { return customerEmail; }
    public void setCustomerEmail(String customerEmail) { this.customerEmail = customerEmail; }

    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }
}
```

**OrderResponse.java**:

```java
package dev.alefiengo.orderservice.model.dto;

import dev.alefiengo.orderservice.model.entity.OrderStatus;
import java.math.BigDecimal;
import java.time.LocalDateTime;

public class OrderResponse {

    private Long id;
    private Long productId;
    private Integer quantity;
    private String customerName;
    private String customerEmail;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private LocalDateTime createdAt;

    // Getters y Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public Long getProductId() { return productId; }
    public void setProductId(Long productId) { this.productId = productId; }

    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }

    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }

    public String getCustomerEmail() { return customerEmail; }
    public void setCustomerEmail(String customerEmail) { this.customerEmail = customerEmail; }

    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }

    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
}
```

### 6. Repository

```java
package dev.alefiengo.orderservice.repository;

import dev.alefiengo.orderservice.model.entity.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

### 7. Service

```java
package dev.alefiengo.orderservice.service;

import dev.alefiengo.orderservice.model.dto.OrderRequest;
import dev.alefiengo.orderservice.model.dto.OrderResponse;
import dev.alefiengo.orderservice.model.entity.Order;
import dev.alefiengo.orderservice.repository.OrderRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public OrderResponse create(OrderRequest request) {
        Order order = new Order();
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setCustomerName(request.getCustomerName());
        order.setCustomerEmail(request.getCustomerEmail());
        order.setTotalAmount(request.getTotalAmount());

        Order saved = orderRepository.save(order);
        return mapToResponse(saved);
    }

    @Transactional(readOnly = true)
    public List<OrderResponse> findAll() {
        return orderRepository.findAll().stream()
                .map(this::mapToResponse)
                .collect(Collectors.toList());
    }

    @Transactional(readOnly = true)
    public OrderResponse findById(Long id) {
        Order order = orderRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Order not found with id: " + id));
        // Nota: En producción, crear OrderNotFoundException extends RuntimeException
        // Ver Clase 3 para manejo de excepciones con @ControllerAdvice
        return mapToResponse(order);
    }

    private OrderResponse mapToResponse(Order order) {
        OrderResponse response = new OrderResponse();
        response.setId(order.getId());
        response.setProductId(order.getProductId());
        response.setQuantity(order.getQuantity());
        response.setCustomerName(order.getCustomerName());
        response.setCustomerEmail(order.getCustomerEmail());
        response.setTotalAmount(order.getTotalAmount());
        response.setStatus(order.getStatus());
        response.setCreatedAt(order.getCreatedAt());
        return response;
    }
}
```

### 8. Controller

```java
package dev.alefiengo.orderservice.controller;

import dev.alefiengo.orderservice.model.dto.OrderRequest;
import dev.alefiengo.orderservice.model.dto.OrderResponse;
import dev.alefiengo.orderservice.service.OrderService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderRequest request) {
        OrderResponse response = orderService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping
    public ResponseEntity<List<OrderResponse>> findAll() {
        List<OrderResponse> orders = orderService.findAll();
        return ResponseEntity.ok(orders);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> findById(@PathVariable Long id) {
        OrderResponse order = orderService.findById(id);
        return ResponseEntity.ok(order);
    }
}
```

---

## Explicación detallada

### Paso 1: ¿Por qué crear un segundo microservicio?

**Arquitectura de microservicios**:

```
┌──────────────────┐      ┌──────────────────┐
│ product-service  │      │  order-service   │
│   (puerto 8080)  │      │   (puerto 8081)  │
├──────────────────┤      ├──────────────────┤
│ PostgreSQL       │      │ PostgreSQL       │
│ ecommerce        │      │ ecommerce_orders │
└──────────────────┘      └──────────────────┘
```

**Ventajas**:

1. **Autonomía**: Cada servicio se despliega independientemente
2. **Escalabilidad**: Escalar order-service sin afectar product-service
3. **Resiliencia**: Falla de productos no afecta órdenes
4. **Dominio separado**: Productos vs Órdenes son bounded contexts diferentes

### Paso 2: Diferencias entre Product y Order

| Aspecto | Product | Order |
|---------|---------|-------|
| **Propósito** | Catálogo | Transacciones |
| **Relaciones** | Category (1:N) | Ninguna (solo productId) |
| **Mutabilidad** | Alta (precio cambia) | Baja (orden es snapshot) |
| **Ciclo de vida** | Permanente | Temporal (archivada) |

**¿Por qué Order NO tiene FK a Product?**

```java
// ❌ NO hacer esto en microservicios
@ManyToOne
@JoinColumn(name = "product_id")
private Product product; // FK a otra BD

// ✅ CORRECTO
@Column(nullable = false)
private Long productId; // Solo ID, sin FK
```

**Razón**: Microservicios NO comparten base de datos. `productId` es solo un identificador, no una relación de BD.

### Paso 3: OrderStatus - Máquina de estados

```java
public enum OrderStatus {
    PENDING,      // Estado inicial
    CONFIRMED,    // Transición: inventory-service valida stock
    CANCELLED     // Transición: inventory-service rechaza (sin stock)
}
```

**Flujo de estados** (implementado en Clase 6):

```
PENDING ──(stock OK)──> CONFIRMED
    └───(no stock)───> CANCELLED
```

**En este lab**: Todas las órdenes quedan en PENDING.

**En Clase 6**: inventory-service cambiará el estado mediante eventos.

### Paso 4: Validaciones con Bean Validation

```java
@NotNull(message = "Product ID is required")
@Positive(message = "Product ID must be positive")
private Long productId;

@Max(value = 100, message = "Quantity cannot exceed 100")
private Integer quantity;
```

**Anotaciones comunes**:

| Anotación | Validación | Ejemplo |
|-----------|------------|---------|
| `@NotNull` | Campo no puede ser null | `productId` |
| `@NotBlank` | String no puede estar vacío | `customerName` |
| `@Positive` | Número debe ser positivo | `quantity` |
| `@Email` | Formato de email válido | `customerEmail` |
| `@Max` | Valor máximo | `quantity <= 100` |

**¿Cuándo se ejecutan?**

Al usar `@Valid` en el controller:

```java
@PostMapping
public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderRequest request) {
    // Si validación falla, Spring retorna 400 Bad Request automáticamente
}
```

### Paso 5: Arquitectura en capas (igual que product-service)

```
Controller → Service → Repository → Database

OrderController
    ├─> OrderService
         ├─> OrderRepository
              ├─> PostgreSQL (ecommerce_orders)
```

**Responsabilidades**:

- **Controller**: HTTP, validación de input, mapeo a DTOs
- **Service**: Lógica de negocio, transacciones
- **Repository**: Acceso a datos, queries

### Paso 6: Ejecutar en puerto diferente

```bash
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

**Alternativa**: Configurar en application.yml:

```yaml
server:
  port: 8081
```

**Verificar ambos servicios corriendo**:

```bash
curl http://localhost:8080/api/products  # product-service
curl http://localhost:8081/api/orders    # order-service
```

---

## Conceptos aprendidos

- Cada **microservicio tiene su propia base de datos** (database per service pattern)
- **Spring Initializr** permite crear proyectos Spring Boot desde CLI o web
- **Puertos diferentes** evitan conflictos cuando múltiples servicios corren en localhost
- **OrderStatus enum** representa máquina de estados del ciclo de vida de una orden
- **Bean Validation** (@NotNull, @Positive, @Email) valida input automáticamente
- **productId como Long** (no FK) porque microservicios no comparten BD
- **Arquitectura en capas** se mantiene consistente entre microservicios
- **@PrePersist** ejecuta lógica antes de guardar entidad (timestamp, valores por defecto)
- **DTOs separados** (OrderRequest, OrderResponse) separan API de modelo de datos
- Orden queda en estado **PENDING** hasta que inventory-service la valide (Clase 6)

---

## Troubleshooting

### Problema 1: Base de datos ecommerce_orders no existe

**Síntoma**:

```
PSQLException: database "ecommerce_orders" does not exist
```

**Solución**:

```bash
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_orders;"
```

### Problema 2: Puerto 8081 ocupado

**Síntoma**:

```
Port 8081 was already in use
```

**Solución**:

```bash
# Verificar qué usa el puerto
lsof -ti:8081

# Matar proceso
lsof -ti:8081 | xargs kill -9

# O usar otro puerto
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8082
```

### Problema 3: Error de validación pero no se muestra mensaje

**Síntoma**: Retorna 400 pero sin detalles del error.

**Solución**: Agregar exception handler:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );
        return ResponseEntity.badRequest().body(errors);
    }
}
```

### Problema 4: Orden se crea sin createdAt

**Síntoma**: Campo `createdAt` es null.

**Solución**: Verificar que `@PrePersist` esté presente:

```java
@PrePersist
protected void onCreate() {
    this.createdAt = LocalDateTime.now();
    if (this.status == null) {
        this.status = OrderStatus.PENDING;
    }
}
```

### Problema 5: Cannot resolve symbol OrderStatus

**Síntoma**: IDE no encuentra OrderStatus enum.

**Solución**: Crear archivo `OrderStatus.java` en package `model.entity`:

```java
package dev.alefiengo.orderservice.model.entity;

public enum OrderStatus {
    PENDING,
    CONFIRMED,
    CANCELLED
}
```

---

## Desafío adicional

### Desafío 1: Agregar endpoint DELETE

Implementa eliminación de órdenes:

```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    orderService.delete(id);
    return ResponseEntity.noContent().build();
}
```

### Desafío 2: Filtrar órdenes por estado

Agrega query method en OrderRepository:

```java
List<Order> findByStatus(OrderStatus status);
```

Y endpoint en Controller:

```java
@GetMapping("/status/{status}")
public ResponseEntity<List<OrderResponse>> findByStatus(@PathVariable OrderStatus status) {
    // ...
}
```

### Desafío 3: Calcular totalAmount automáticamente

En lugar de recibir totalAmount en OrderRequest, calcularlo:

1. Agregar ProductClient (RestTemplate o WebClient)
2. Consultar precio de producto
3. Calcular: `price * quantity`
4. Asignar a order.totalAmount

**Nota**: Este es un anti-patrón en microservicios (llamada síncrona). En Clase 6 lo haremos event-driven.

---

## Recursos adicionales

- [Spring Initializr](https://start.spring.io/)
- [Database per Service Pattern](https://microservices.io/patterns/data/database-per-service.html)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Bean Validation Constraints](https://jakarta.ee/specifications/bean-validation/3.0/apidocs/jakarta/validation/constraints/package-summary.html)
- [Microservices Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html)
