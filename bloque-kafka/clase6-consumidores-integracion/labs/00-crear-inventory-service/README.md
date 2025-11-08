# Lab 00: Crear inventory-service

Crear el tercer microservicio de la arquitectura (inventory-service) que gestionará el inventario de productos y validará la disponibilidad de stock para las órdenes mediante eventos Kafka.

---

## Objetivo

Implementar inventory-service como un microservicio independiente con base de datos PostgreSQL propia, endpoints REST para gestión de inventario, y métodos de negocio para reservar y liberar stock.

---

## Comandos a ejecutar

### Paso 1: Generar proyecto con Spring Initializr

```bash
cd ~/workspace

curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,kafka,validation \
  -d groupId=dev.alefiengo \
  -d artifactId=inventory-service \
  -d name=InventoryServiceApplication \
  -d description="Inventory management microservice" \
  -d packageName=dev.alefiengo.inventoryservice \
  -d javaVersion=17 \
  -d type=maven-project \
  -o inventory-service.zip

unzip inventory-service.zip -d inventory-service
cd inventory-service
```

**Output esperado**:
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 56234  100 56234    0     0  45234      0  0:00:01  0:00:01 --:--:-- 45234
Archive:  inventory-service.zip
   creating: inventory-service/
  inflating: inventory-service/mvnw
  inflating: inventory-service/mvnw.cmd
...
```

### Paso 2: Crear base de datos PostgreSQL

```bash
# Verificar que PostgreSQL está corriendo
docker compose -f ~/workspace/kafka-infrastructure/docker-compose.yml ps | grep postgres

# Crear base de datos ecommerce_inventory
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_inventory;"

# Verificar creación
docker exec -it postgres psql -U postgres -c "\l" | grep ecommerce_inventory
```

**Output esperado**:
```
CREATE DATABASE
 ecommerce_inventory | postgres | UTF8     | C       | C     |
```

### Paso 3: Configurar application.yml

```bash
cd ~/workspace/inventory-service/src/main/resources
rm application.properties
cat > application.yml << 'EOF'
spring:
  application:
    name: inventory-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce_inventory}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

server:
  port: 8082

logging:
  level:
    dev.alefiengo.inventoryservice: DEBUG
    org.springframework.web: INFO
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 4: Crear estructura de directorios

```bash
cd ~/workspace/inventory-service/src/main/java/dev/alefiengo/inventoryservice

mkdir -p controller
mkdir -p service
mkdir -p repository
mkdir -p model/entity
mkdir -p model/dto
```

**Output esperado**: Sin salida, directorios creados.

### Paso 5: Crear entidad InventoryItem

```bash
cat > model/entity/InventoryItem.java << 'EOF'
package dev.alefiengo.inventoryservice.model.entity;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "inventory_items")
public class InventoryItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private Long productId;  // FK lógico a product-service

    @Column(nullable = false)
    private String productName;

    @Column(nullable = false)
    private Integer availableStock;

    @Column(nullable = false)
    private Integer reservedStock;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at")
    private Instant updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = Instant.now();
        updatedAt = Instant.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = Instant.now();
    }

    // Constructor vacío (requerido por JPA)
    public InventoryItem() {
    }

    // Constructor completo
    public InventoryItem(Long productId, String productName, Integer initialStock) {
        this.productId = productId;
        this.productName = productName;
        this.availableStock = initialStock;
        this.reservedStock = 0;
    }

    // Métodos de negocio

    /**
     * Verifica si hay stock disponible suficiente
     */
    public boolean hasAvailableStock(Integer quantity) {
        return this.availableStock >= quantity;
    }

    /**
     * Reserva stock (mueve de disponible a reservado)
     * NO persiste automáticamente, debe llamar repository.save()
     */
    public void reserveStock(Integer quantity) {
        if (!hasAvailableStock(quantity)) {
            throw new IllegalStateException(
                "Insufficient available stock. Available: " + availableStock + ", Requested: " + quantity
            );
        }
        this.availableStock -= quantity;
        this.reservedStock += quantity;
    }

    /**
     * Libera stock (mueve de reservado a disponible)
     * Usado cuando una orden se cancela
     */
    public void releaseStock(Integer quantity) {
        if (this.reservedStock < quantity) {
            throw new IllegalStateException(
                "Cannot release more than reserved. Reserved: " + reservedStock + ", Requested: " + quantity
            );
        }
        this.reservedStock -= quantity;
        this.availableStock += quantity;
    }

    /**
     * Calcula el stock total (disponible + reservado)
     */
    public Integer getTotalStock() {
        return this.availableStock + this.reservedStock;
    }

    // Getters y Setters

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }

    public Integer getAvailableStock() {
        return availableStock;
    }

    public void setAvailableStock(Integer availableStock) {
        this.availableStock = availableStock;
    }

    public Integer getReservedStock() {
        return reservedStock;
    }

    public void setReservedStock(Integer reservedStock) {
        this.reservedStock = reservedStock;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Instant createdAt) {
        this.createdAt = createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(Instant updatedAt) {
        this.updatedAt = updatedAt;
    }

    @Override
    public String toString() {
        return "InventoryItem{" +
                "id=" + id +
                ", productId=" + productId +
                ", productName='" + productName + '\'' +
                ", availableStock=" + availableStock +
                ", reservedStock=" + reservedStock +
                ", totalStock=" + getTotalStock() +
                '}';
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 6: Crear DTOs

```bash
cat > model/dto/InventoryItemRequest.java << 'EOF'
package dev.alefiengo.inventoryservice.model.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public class InventoryItemRequest {

    @NotNull(message = "Product ID is required")
    private Long productId;

    @NotBlank(message = "Product name is required")
    private String productName;

    @NotNull(message = "Initial stock is required")
    @Min(value = 0, message = "Initial stock must be non-negative")
    private Integer initialStock;

    // Constructor vacío
    public InventoryItemRequest() {
    }

    // Constructor completo
    public InventoryItemRequest(Long productId, String productName, Integer initialStock) {
        this.productId = productId;
        this.productName = productName;
        this.initialStock = initialStock;
    }

    // Getters y Setters

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }

    public Integer getInitialStock() {
        return initialStock;
    }

    public void setInitialStock(Integer initialStock) {
        this.initialStock = initialStock;
    }

    @Override
    public String toString() {
        return "InventoryItemRequest{" +
                "productId=" + productId +
                ", productName='" + productName + '\'' +
                ", initialStock=" + initialStock +
                '}';
    }
}
EOF

cat > model/dto/InventoryItemResponse.java << 'EOF'
package dev.alefiengo.inventoryservice.model.dto;

import java.time.Instant;

public class InventoryItemResponse {

    private Long id;
    private Long productId;
    private String productName;
    private Integer availableStock;
    private Integer reservedStock;
    private Integer totalStock;
    private Instant createdAt;
    private Instant updatedAt;

    // Constructor vacío
    public InventoryItemResponse() {
    }

    // Constructor completo
    public InventoryItemResponse(Long id, Long productId, String productName,
                                  Integer availableStock, Integer reservedStock,
                                  Integer totalStock, Instant createdAt, Instant updatedAt) {
        this.id = id;
        this.productId = productId;
        this.productName = productName;
        this.availableStock = availableStock;
        this.reservedStock = reservedStock;
        this.totalStock = totalStock;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    // Getters y Setters

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }

    public Integer getAvailableStock() {
        return availableStock;
    }

    public void setAvailableStock(Integer availableStock) {
        this.availableStock = availableStock;
    }

    public Integer getReservedStock() {
        return reservedStock;
    }

    public void setReservedStock(Integer reservedStock) {
        this.reservedStock = reservedStock;
    }

    public Integer getTotalStock() {
        return totalStock;
    }

    public void setTotalStock(Integer totalStock) {
        this.totalStock = totalStock;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Instant createdAt) {
        this.createdAt = createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(Instant updatedAt) {
        this.updatedAt = updatedAt;
    }

    @Override
    public String toString() {
        return "InventoryItemResponse{" +
                "id=" + id +
                ", productId=" + productId +
                ", productName='" + productName + '\'' +
                ", availableStock=" + availableStock +
                ", reservedStock=" + reservedStock +
                ", totalStock=" + totalStock +
                '}';
    }
}
EOF
```

**Output esperado**: Sin salida, archivos creados.

### Paso 7: Crear Repository

```bash
cat > repository/InventoryRepository.java << 'EOF'
package dev.alefiengo.inventoryservice.repository;

import dev.alefiengo.inventoryservice.model.entity.InventoryItem;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface InventoryRepository extends JpaRepository<InventoryItem, Long> {

    /**
     * Buscar item de inventario por productId (FK lógico)
     */
    Optional<InventoryItem> findByProductId(Long productId);

    /**
     * Verificar si existe item para un producto específico
     */
    boolean existsByProductId(Long productId);
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 8: Crear Service

```bash
cat > service/InventoryService.java << 'EOF'
package dev.alefiengo.inventoryservice.service;

import dev.alefiengo.inventoryservice.model.dto.InventoryItemRequest;
import dev.alefiengo.inventoryservice.model.dto.InventoryItemResponse;
import dev.alefiengo.inventoryservice.model.entity.InventoryItem;
import dev.alefiengo.inventoryservice.repository.InventoryRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class InventoryService {

    private static final Logger log = LoggerFactory.getLogger(InventoryService.class);

    private final InventoryRepository inventoryRepository;

    // Constructor injection
    public InventoryService(InventoryRepository inventoryRepository) {
        this.inventoryRepository = inventoryRepository;
    }

    /**
     * Crear un nuevo item de inventario
     */
    @Transactional
    public InventoryItemResponse createInventoryItem(InventoryItemRequest request) {
        log.info("Creating inventory item for productId: {}", request.getProductId());

        // Verificar que no exista ya un item para este producto
        if (inventoryRepository.existsByProductId(request.getProductId())) {
            throw new RuntimeException("Inventory item already exists for productId: " + request.getProductId());
        }

        InventoryItem item = new InventoryItem(
            request.getProductId(),
            request.getProductName(),
            request.getInitialStock()
        );

        InventoryItem saved = inventoryRepository.save(item);
        log.info("Inventory item created: {}", saved);

        return toResponse(saved);
    }

    /**
     * Obtener todos los items de inventario
     */
    @Transactional(readOnly = true)
    public List<InventoryItemResponse> getAllInventoryItems() {
        log.debug("Fetching all inventory items");
        return inventoryRepository.findAll()
                .stream()
                .map(this::toResponse)
                .collect(Collectors.toList());
    }

    /**
     * Obtener item por ID
     */
    @Transactional(readOnly = true)
    public InventoryItemResponse getInventoryItemById(Long id) {
        log.debug("Fetching inventory item by id: {}", id);
        InventoryItem item = inventoryRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Inventory item not found with id: " + id));
        return toResponse(item);
    }

    /**
     * Obtener item por productId
     */
    @Transactional(readOnly = true)
    public InventoryItemResponse getInventoryItemByProductId(Long productId) {
        log.debug("Fetching inventory item by productId: {}", productId);
        InventoryItem item = inventoryRepository.findByProductId(productId)
                .orElseThrow(() -> new RuntimeException("Inventory item not found for productId: " + productId));
        return toResponse(item);
    }

    /**
     * Eliminar item de inventario
     */
    @Transactional
    public void deleteInventoryItem(Long id) {
        log.info("Deleting inventory item with id: {}", id);
        if (!inventoryRepository.existsById(id)) {
            throw new RuntimeException("Inventory item not found with id: " + id);
        }
        inventoryRepository.deleteById(id);
        log.info("Inventory item deleted: id={}", id);
    }

    /**
     * Mapper: Entity → DTO Response
     */
    private InventoryItemResponse toResponse(InventoryItem item) {
        return new InventoryItemResponse(
            item.getId(),
            item.getProductId(),
            item.getProductName(),
            item.getAvailableStock(),
            item.getReservedStock(),
            item.getTotalStock(),
            item.getCreatedAt(),
            item.getUpdatedAt()
        );
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 9: Crear Controller

```bash
cat > controller/InventoryController.java << 'EOF'
package dev.alefiengo.inventoryservice.controller;

import dev.alefiengo.inventoryservice.model.dto.InventoryItemRequest;
import dev.alefiengo.inventoryservice.model.dto.InventoryItemResponse;
import dev.alefiengo.inventoryservice.service.InventoryService;
import jakarta.validation.Valid;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/inventory")
public class InventoryController {

    private static final Logger log = LoggerFactory.getLogger(InventoryController.class);

    private final InventoryService inventoryService;

    // Constructor injection
    public InventoryController(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    /**
     * POST /api/inventory - Crear item de inventario
     */
    @PostMapping
    public ResponseEntity<InventoryItemResponse> createInventoryItem(@Valid @RequestBody InventoryItemRequest request) {
        log.info("POST /api/inventory - Creating inventory item: {}", request);
        InventoryItemResponse response = inventoryService.createInventoryItem(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    /**
     * GET /api/inventory - Listar todos los items
     */
    @GetMapping
    public ResponseEntity<List<InventoryItemResponse>> getAllInventoryItems() {
        log.debug("GET /api/inventory - Fetching all inventory items");
        List<InventoryItemResponse> items = inventoryService.getAllInventoryItems();
        return ResponseEntity.ok(items);
    }

    /**
     * GET /api/inventory/{id} - Obtener item por ID
     */
    @GetMapping("/{id}")
    public ResponseEntity<InventoryItemResponse> getInventoryItemById(@PathVariable Long id) {
        log.debug("GET /api/inventory/{} - Fetching inventory item", id);
        InventoryItemResponse response = inventoryService.getInventoryItemById(id);
        return ResponseEntity.ok(response);
    }

    /**
     * GET /api/inventory/product/{productId} - Obtener item por productId
     */
    @GetMapping("/product/{productId}")
    public ResponseEntity<InventoryItemResponse> getInventoryItemByProductId(@PathVariable Long productId) {
        log.debug("GET /api/inventory/product/{} - Fetching inventory item", productId);
        InventoryItemResponse response = inventoryService.getInventoryItemByProductId(productId);
        return ResponseEntity.ok(response);
    }

    /**
     * DELETE /api/inventory/{id} - Eliminar item
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteInventoryItem(@PathVariable Long id) {
        log.info("DELETE /api/inventory/{} - Deleting inventory item", id);
        inventoryService.deleteInventoryItem(id);
        return ResponseEntity.noContent().build();
    }
}
EOF
```

**Output esperado**: Sin salida, archivo creado.

### Paso 10: Compilar y ejecutar

```bash
cd ~/workspace/inventory-service

# Compilar
mvn clean install

# Ejecutar
mvn spring-boot:run
```

**Output esperado**:
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.x.x)

...
2025-11-08T10:30:00.123  INFO 12345 --- [main] d.a.i.InventoryServiceApplication        : Started InventoryServiceApplication in 3.456 seconds
```

### Paso 11: Probar endpoints REST

**Terminal 2** (dejar corriendo el servidor en terminal 1):

```bash
# Crear item de inventario para producto 1
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "initialStock": 50
  }'

# Crear otro item
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 2,
    "productName": "Mouse Logitech MX Master",
    "initialStock": 100
  }'

# Listar todos los items
curl http://localhost:8082/api/inventory

# Consultar item por ID
curl http://localhost:8082/api/inventory/1

# Consultar item por productId
curl http://localhost:8082/api/inventory/product/1
```

**Output esperado** (crear item):
```json
{
  "id": 1,
  "productId": 1,
  "productName": "Laptop Dell XPS 15",
  "availableStock": 50,
  "reservedStock": 0,
  "totalStock": 50,
  "createdAt": "2025-11-08T10:30:15.123Z",
  "updatedAt": "2025-11-08T10:30:15.123Z"
}
```

**Output esperado** (listar todos):
```json
[
  {
    "id": 1,
    "productId": 1,
    "productName": "Laptop Dell XPS 15",
    "availableStock": 50,
    "reservedStock": 0,
    "totalStock": 50,
    "createdAt": "2025-11-08T10:30:15.123Z",
    "updatedAt": "2025-11-08T10:30:15.123Z"
  },
  {
    "id": 2,
    "productId": 2,
    "productName": "Mouse Logitech MX Master",
    "availableStock": 100,
    "reservedStock": 0,
    "totalStock": 100,
    "createdAt": "2025-11-08T10:30:20.456Z",
    "updatedAt": "2025-11-08T10:30:20.456Z"
  }
]
```

### Paso 12: Verificar en PostgreSQL

```bash
# Conectar a base de datos
docker exec -it postgres psql -U postgres -d ecommerce_inventory

# Ver estructura de tabla
\d inventory_items

# Ver datos
SELECT id, product_id, product_name, available_stock, reserved_stock FROM inventory_items;
```

**Output esperado**:
```
 id | product_id |      product_name       | available_stock | reserved_stock
----+------------+-------------------------+-----------------+----------------
  1 |          1 | Laptop Dell XPS 15      |              50 |              0
  2 |          2 | Mouse Logitech MX Master|             100 |              0
(2 rows)
```

---

## Desglose del comando

### Spring Initializr curl

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `-d dependencies` | `web,data-jpa,postgresql,kafka,validation` | Spring Web, JPA, PostgreSQL driver, Kafka, Bean Validation |
| `-d groupId` | `dev.alefiengo` | Identificador de grupo (estándar del curso) |
| `-d artifactId` | `inventory-service` | Nombre del proyecto |
| `-d name` | `InventoryServiceApplication` | Nombre de la clase principal |
| `-d packageName` | `dev.alefiengo.inventoryservice` | Paquete base Java |
| `-d javaVersion` | `17` | Java 17 (LTS) |
| `-d type` | `maven-project` | Proyecto Maven (no Gradle) |
| `-o` | `inventory-service.zip` | Archivo de salida |

### application.yml clave

| Propiedad | Valor | Descripción |
|-----------|-------|-------------|
| `spring.application.name` | `inventory-service` | Nombre del microservicio |
| `spring.datasource.url` | `jdbc:postgresql://localhost:5432/ecommerce_inventory` | Conexión a BD propia |
| `spring.jpa.hibernate.ddl-auto` | `update` | Hibernate crea/actualiza tablas automáticamente |
| `server.port` | `8082` | Puerto del microservicio (diferente a product-service:8080 y order-service:8081) |

### Entidad InventoryItem

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | `Long` | Primary Key (autoincremental) |
| `productId` | `Long` | FK lógico a product-service (NO hay FK física entre bases de datos) |
| `productName` | `String` | Nombre del producto (desnormalizado para performance) |
| `availableStock` | `Integer` | Stock disponible para venta |
| `reservedStock` | `Integer` | Stock reservado por órdenes pendientes |
| `createdAt` | `Instant` | Timestamp de creación |
| `updatedAt` | `Instant` | Timestamp de última actualización |

**Métodos de negocio**:
- `hasAvailableStock(quantity)`: Valida si hay stock suficiente
- `reserveStock(quantity)`: Mueve stock de disponible a reservado
- `releaseStock(quantity)`: Mueve stock de reservado a disponible (cuando orden se cancela)
- `getTotalStock()`: Calcula stock total (disponible + reservado)

---

## Explicación detallada

### Arquitectura de 3 microservicios

Ahora tenemos 3 microservicios independientes:

```
┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
│  product-service    │      │   order-service     │      │ inventory-service   │
│  (puerto 8080)      │      │   (puerto 8081)     │      │  (puerto 8082)      │
├─────────────────────┤      ├─────────────────────┤      ├─────────────────────┤
│ PostgreSQL          │      │ PostgreSQL          │      │ PostgreSQL          │
│ ecommerce           │      │ ecommerce_orders    │      │ ecommerce_inventory │
├─────────────────────┤      ├─────────────────────┤      ├─────────────────────┤
│ Tabla: products     │      │ Tabla: customer_orders│     │ Tabla: inventory_items│
└─────────────────────┘      └─────────────────────┘      └─────────────────────┘
```

**Principio de Database per Service**:
- Cada microservicio tiene su propia base de datos
- NO hay foreign keys físicas entre bases de datos
- La relación productId es solo lógica (no enforced por BD)
- Esto permite escalabilidad y autonomía de servicios

### Modelo de datos: InventoryItem

**Campo productId**:
```java
@Column(nullable = false, unique = true)
private Long productId;  // FK lógico a product-service
```

Este campo es una **foreign key lógica**, no física:
- `unique = true`: Solo puede haber UN item de inventario por producto
- NO hay constraint FOREIGN KEY en PostgreSQL
- La validación de que el producto existe se hace a nivel aplicación (o via eventos)

**Campos de stock**:
```java
private Integer availableStock;  // Stock disponible para venta
private Integer reservedStock;   // Stock reservado por órdenes
```

**Flujo de reserva de stock**:
1. Cliente crea orden → order-service
2. order-service publica OrderPlacedEvent
3. inventory-service consume evento
4. inventory-service llama `item.reserveStock(quantity)`
5. availableStock disminuye, reservedStock aumenta
6. Se guarda en BD con `repository.save(item)`

**Ejemplo**:
```
Estado inicial: availableStock=50, reservedStock=0
Reservar 5 unidades: availableStock=45, reservedStock=5
Reservar 3 más: availableStock=42, reservedStock=8
Total stock: 42 + 8 = 50 (se mantiene constante)
```

### Métodos de negocio en la entidad

**Patrón Domain-Driven Design (DDD)**:

La entidad `InventoryItem` contiene lógica de negocio (no es un simple DTO):

```java
public void reserveStock(Integer quantity) {
    if (!hasAvailableStock(quantity)) {
        throw new IllegalStateException("Insufficient stock");
    }
    this.availableStock -= quantity;
    this.reservedStock += quantity;
}
```

**Ventajas**:
- Encapsula reglas de negocio en la entidad
- Previene estados inválidos (ej. stock negativo)
- Métodos reutilizables desde Service o Consumer

**IMPORTANTE**: Estos métodos NO persisten automáticamente. Después de llamar `reserveStock()`, debes llamar `repository.save(item)`.

### Separación Entity vs DTO

**Entity** (`InventoryItem.java`):
- Usa anotaciones JPA (`@Entity`, `@Table`, `@Column`)
- Tiene métodos de negocio
- Se persiste en base de datos
- NUNCA se expone directamente en REST

**DTO Request** (`InventoryItemRequest.java`):
- Validaciones con Bean Validation (`@NotNull`, `@Min`)
- Solo campos necesarios para crear item (productId, productName, initialStock)
- Recibido en POST requests

**DTO Response** (`InventoryItemResponse.java`):
- Todos los campos que el cliente necesita ver
- Incluye campos calculados (totalStock)
- Retornado en GET requests

**Mapper en Service**:
```java
private InventoryItemResponse toResponse(InventoryItem item) {
    return new InventoryItemResponse(
        item.getId(),
        item.getProductId(),
        item.getProductName(),
        item.getAvailableStock(),
        item.getReservedStock(),
        item.getTotalStock(),  // ← Campo calculado
        item.getCreatedAt(),
        item.getUpdatedAt()
    );
}
```

### Constructor injection

```java
@Service
public class InventoryService {
    private final InventoryRepository inventoryRepository;

    // ✅ CORRECTO: Constructor injection
    public InventoryService(InventoryRepository inventoryRepository) {
        this.inventoryRepository = inventoryRepository;
    }
}
```

**Por qué es mejor que `@Autowired` en fields**:
- Permite marcar dependencia como `final` (inmutable)
- Facilita testing (puedes pasar mocks en constructor)
- Spring recomienda constructor injection desde versión 4.3+
- Hace dependencias explícitas

### Queries derivadas en Repository

```java
public interface InventoryRepository extends JpaRepository<InventoryItem, Long> {
    Optional<InventoryItem> findByProductId(Long productId);
    boolean existsByProductId(Long productId);
}
```

**Spring Data JPA** genera implementación automática basándose en el nombre del método:

- `findByProductId` → `SELECT * FROM inventory_items WHERE product_id = ?`
- `existsByProductId` → `SELECT COUNT(*) > 0 FROM inventory_items WHERE product_id = ?`

No necesitas escribir `@Query`, Spring lo deduce del nombre.

### Gestión de excepciones simplificada

En este lab usamos `RuntimeException` para simplicidad:

```java
throw new RuntimeException("Inventory item not found with id: " + id);
```

**En producción** (ya aprendido en Clase 3):
- Crear excepciones custom: `InventoryItemNotFoundException extends RuntimeException`
- Usar `@ControllerAdvice` para manejar excepciones globalmente
- Retornar ErrorResponse con status HTTP apropiado

**Para este lab**: RuntimeException es suficiente (foco en arquitectura, no manejo de errores).

### Timestamps automáticos

```java
@PrePersist
protected void onCreate() {
    createdAt = Instant.now();
    updatedAt = Instant.now();
}

@PreUpdate
protected void onUpdate() {
    updatedAt = Instant.now();
}
```

**JPA Lifecycle Callbacks**:
- `@PrePersist`: Se ejecuta ANTES de INSERT
- `@PreUpdate`: Se ejecuta ANTES de UPDATE
- Automáticamente gestiona timestamps sin código manual en Service

### Variables de entorno con fallback

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce_inventory}
```

**Patrón** `${VAR:default}`:
- Si variable `DB_HOST` existe → usa su valor
- Si NO existe → usa `localhost`

**Ventajas**:
- Funciona en desarrollo local (sin configurar variables)
- Funciona en Docker (con variables de entorno)
- Funciona en Kubernetes (con ConfigMaps/Secrets)
- Patrón 12-Factor App

---

## Conceptos aprendidos

- **Database per Service**: Cada microservicio tiene su propia base de datos independiente
- **Foreign Key lógica**: productId relaciona servicios sin constraint físico en BD
- **Stock management**: Separación entre availableStock y reservedStock
- **Domain logic en entities**: Métodos de negocio (reserveStock, releaseStock) en entidad JPA
- **DTO pattern**: Separación entre Entity (persistencia) y Request/Response (API)
- **Constructor injection**: Mejor práctica para inyección de dependencias en Spring
- **Derived queries**: Spring Data genera queries basándose en nombres de métodos
- **JPA Lifecycle callbacks**: @PrePersist y @PreUpdate para timestamps automáticos
- **Environment variables con fallback**: Patrón ${VAR:default} para configuración flexible
- **Puerto diferente por servicio**: 8080 (product), 8081 (order), 8082 (inventory)

---

## Troubleshooting

### Error: "database ecommerce_inventory does not exist"

**Causa**: No se creó la base de datos antes de iniciar la aplicación.

**Solución**:
```bash
docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce_inventory;"
```

### Error: "Port 8082 already in use"

**Causa**: Otro proceso está usando el puerto 8082.

**Solución 1** (encontrar y matar proceso):
```bash
# Linux/Mac
lsof -i :8082
kill -9 [PID]

# Windows
netstat -ano | findstr :8082
taskkill /PID [PID] /F
```

**Solución 2** (cambiar puerto):
```yaml
# application.yml
server:
  port: 8083  # Usar puerto diferente
```

### Error: "Inventory item already exists for productId: 1"

**Causa**: Intentaste crear dos items de inventario para el mismo productId.

**Solución**: Cada producto debe tener UN SOLO item de inventario. Si necesitas actualizar stock, crea un endpoint PUT (no está en este lab, viene en labs futuros).

### Error: "Column 'product_id' cannot be null"

**Causa**: El request JSON no incluye productId.

**Solución**:
```bash
# ❌ INCORRECTO
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{"productName": "Test", "initialStock": 10}'

# ✅ CORRECTO
curl -X POST http://localhost:8082/api/inventory \
  -H "Content-Type: application/json" \
  -d '{"productId": 1, "productName": "Test", "initialStock": 10}'
```

### Warning: "spring.jpa.open-in-view is enabled by default"

**No es error, es warning informativo**.

**Explicación**: Open Session In View mantiene sesión de Hibernate abierta durante el request HTTP completo (no solo en @Transactional).

**Opciones**:
1. Ignorar warning (comportamiento por defecto funciona bien para este curso)
2. Desactivar explícitamente:
```yaml
spring:
  jpa:
    open-in-view: false
```

Para este lab, puedes ignorar el warning.

### Error: "Insufficient available stock. Available: 10, Requested: 15"

**Causa**: La entidad lanza excepción cuando intentas reservar más stock del disponible.

**Esto es correcto**. Los métodos de negocio validan reglas:
```java
public void reserveStock(Integer quantity) {
    if (!hasAvailableStock(quantity)) {
        throw new IllegalStateException("Insufficient stock");
    }
    // ...
}
```

**Solución**: Este error es intencional. En labs futuros, inventory-service publicará `OrderCancelledEvent` cuando no haya stock.

### No aparecen datos en GET /api/inventory

**Verificar**:
1. ¿Creaste items con POST primero?
2. ¿El servidor está corriendo en puerto 8082?
3. ¿La base de datos tiene datos?

```bash
# Verificar en PostgreSQL
docker exec -it postgres psql -U postgres -d ecommerce_inventory \
  -c "SELECT * FROM inventory_items;"
```

Si la tabla está vacía, crea items con POST.

---

## Desafío adicional

### Desafío 1: Implementar endpoint PUT para actualizar stock

Crear endpoint `PUT /api/inventory/{id}/stock` que reciba:
```json
{
  "adjustment": 10  // Puede ser positivo (agregar) o negativo (remover)
}
```

Y actualice `availableStock` directamente (sin reservar).

**Pistas**:
- Crear DTO `StockAdjustmentRequest`
- Agregar método en Service: `adjustStock(Long id, Integer adjustment)`
- Validar que adjustment no deje stock negativo
- Agregar endpoint PUT en Controller

### Desafío 2: Agregar validación de producto existente

Antes de crear InventoryItem, verificar que el producto existe en product-service.

**Opciones**:
1. Llamada REST síncrona a product-service (GET /api/products/{id})
2. Asumir que solo se crean items para productos válidos (dejar validación para labs futuros con eventos)

**Pistas para opción 1**:
- Usar `RestTemplate` o `WebClient`
- Inyectar en Service
- Llamar en `createInventoryItem()` antes de guardar

### Desafío 3: Endpoint de verificación de stock

Crear endpoint `GET /api/inventory/product/{productId}/available?quantity=5` que retorne:
```json
{
  "productId": 1,
  "requestedQuantity": 5,
  "available": true,
  "currentAvailableStock": 50
}
```

**Pistas**:
- Agregar método en Service: `checkAvailability(Long productId, Integer quantity)`
- Crear DTO `StockAvailabilityResponse`
- Usar método `hasAvailableStock()` de la entidad

### Desafío 4: Implementar paginación

Modificar `GET /api/inventory` para soportar paginación:
```bash
curl "http://localhost:8082/api/inventory?page=0&size=10"
```

**Pistas**:
- Cambiar Repository para retornar `Page<InventoryItem>`
- Usar `Pageable` en método del Service
- Controller recibe `@RequestParam int page, @RequestParam int size`
- Retornar `Page<InventoryItemResponse>`

---

## Recursos adicionales

- [Spring Data JPA - Derived Queries](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation)
- [Spring Data JPA - Lifecycle Callbacks](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.entity-persistence.saving-entites.strategies)
- [Bean Validation](https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html)
- [12-Factor App - Config](https://12factor.net/config)
- [Domain-Driven Design - Entities](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Microservices Pattern - Database per Service](https://microservices.io/patterns/data/database-per-service.html)
- [Spring Boot - Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
