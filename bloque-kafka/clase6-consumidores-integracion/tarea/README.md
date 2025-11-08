# Tarea - Clase 6: Implementar InventoryUpdatedEvent

Implementar un nuevo evento `InventoryUpdatedEvent` que se publique cada vez que cambie el stock de un producto, y crear un consumer opcional en product-service para registrar estos cambios.

---

## Objetivo

Practicar el patrón completo de producer-consumer implementando un evento de auditoría que registre todos los cambios en el inventario, aplicando los conceptos de eventos, serialización JSON y consumer groups aprendidos en clase.

---

## Descripción

### Parte 1: Crear InventoryUpdatedEvent (Obligatorio)

Crear una clase de evento que contenga información completa sobre cambios en el inventario:

**Campos requeridos**:
- `productId` (Long): ID del producto afectado
- `productName` (String): Nombre del producto
- `previousStock` (Integer): Stock antes del cambio
- `currentStock` (Integer): Stock después del cambio
- `delta` (Integer): Cambio (positivo o negativo)
- `reason` (String): Razón del cambio (ver opciones abajo)
- `timestamp` (Instant): Cuándo ocurrió el cambio

**Razones válidas**:
- `"ORDER_PLACED"`: Stock reservado por una orden
- `"ORDER_CANCELLED"`: Stock liberado por cancelación
- `"MANUAL_ADJUSTMENT"`: Ajuste manual por API

---

### Parte 2: Implementar Producer en inventory-service (Obligatorio)

1. Crear `InventoryEventProducer.java` con método `publishInventoryUpdated()`
2. Publicar evento al topic `ecommerce.inventory.updated`
3. Integrar en `InventoryService.processOrderPlaced()`:
   - Publicar evento DESPUÉS de reservar stock (reason: `"ORDER_PLACED"`)
   - Publicar evento si no hay stock pero NO se reservó (opcional)

**Configuración en application.yml**:
```yaml
spring:
  kafka:
    producer:
      # Ya existente, no cambiar
```

---

### Parte 3: Crear Topic (Obligatorio)

Crear topic `ecommerce.inventory.updated` con:
- 3 particiones
- Replication factor: 1

**Comando**:
```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.inventory.updated \
  --partitions 3 \
  --replication-factor 1
```

---

### Parte 4: Endpoint de Ajuste Manual (Obligatorio)

Implementar endpoint REST en inventory-service para ajustar stock manualmente:

**Endpoint**: `PUT /api/inventory/{productId}/adjust`

**Request body**:
```json
{
  "delta": 10,
  "reason": "Restock from supplier"
}
```

**Lógica**:
1. Buscar InventoryItem por productId
2. Guardar stock anterior
3. Aplicar delta (puede ser negativo)
4. Guardar en BD
5. Publicar `InventoryUpdatedEvent` con reason `"MANUAL_ADJUSTMENT"`

---

### Parte 5: Consumer en product-service (Bonus Opcional)

Crear consumer en product-service que escuche `ecommerce.inventory.updated` y registre cambios en logs.

**Clase**: `InventoryEventConsumer.java`

**Funcionalidad mínima**:
- Loguear: "Inventory updated: productId={}, delta={}, reason={}"
- NO modificar base de datos (solo logging)

**Configuración en application.yml de product-service**:
```yaml
spring:
  kafka:
    consumer:
      group-id: product-service
      auto-offset-reset: earliest
      # ... resto de configuración
```

---

## Requisitos Técnicos

### Código Java

1. **InventoryUpdatedEvent.java**:
   - Package: `dev.alefiengo.inventoryservice.kafka.event`
   - Constructor vacío (requerido para deserialización)
   - Constructor completo (para crear instancias)
   - Getters y setters
   - toString() para debugging

2. **InventoryEventProducer.java**:
   - Usar `KafkaTemplate<String, InventoryUpdatedEvent>`
   - Constructor injection
   - Logging con SLF4J
   - Manejo asíncrono con `whenComplete()`

3. **InventoryController.java**:
   - Endpoint PUT `/api/inventory/{productId}/adjust`
   - Validación con `@Valid`
   - DTO `AdjustStockRequest` con campos `delta` y `reason`

4. **Consumer opcional** (product-service):
   - `@KafkaListener` con topic `ecommerce.inventory.updated`
   - Group ID: `product-service`
   - Solo logging, NO modificar BD

---

## Criterios de Evaluación

| Criterio | Puntos | Descripción |
|----------|--------|-------------|
| InventoryUpdatedEvent creado correctamente | 15 | Todos los campos, constructores, getters/setters |
| InventoryEventProducer implementado | 15 | KafkaTemplate, logging, manejo async |
| Evento publicado en processOrderPlaced() | 20 | Integración correcta, reason apropiado |
| Topic creado con particiones correctas | 10 | 3 particiones, replication-factor 1 |
| Endpoint PUT /api/inventory/{id}/adjust | 25 | Request/response correcto, validación, publica evento |
| Código compila sin errores | 10 | mvn clean install exitoso |
| Evento visible en kafka-console-consumer | 5 | Verificación con CLI |
| **BONUS: Consumer en product-service** | +10 | Consumer funcional con logging |
| **TOTAL** | **100** (+10 bonus) |

---

## Entrega

### Formato

Crear archivo `SOLUCION.md` en tu repositorio personal con:

1. **Código completo** de todas las clases nuevas (copiar código completo)
2. **Screenshots** de:
   - Endpoint PUT funcionando (curl o Postman)
   - kafka-console-consumer mostrando InventoryUpdatedEvent
   - Logs de inventory-service publicando evento
   - (Bonus) Logs de product-service consumiendo evento
3. **Comandos ejecutados** para crear topic y probar

### Estructura del archivo

```markdown
# Solución Tarea Clase 6 - InventoryUpdatedEvent

## 1. InventoryUpdatedEvent.java

[código completo aquí]

## 2. InventoryEventProducer.java

[código completo aquí]

## 3. Modificación en InventoryService.java

[mostrar método processOrderPlaced() completo]

## 4. AdjustStockRequest.java (DTO)

[código completo aquí]

## 5. Modificación en InventoryController.java

[mostrar endpoint PUT completo]

## 6. Configuración application.yml

[mostrar sección kafka completa]

## 7. Topic creado

Comando ejecutado:
[comando kafka-topics --create...]

Output:
[output del comando]

## 8. Pruebas

### Prueba 1: Ajuste manual de stock

Comando curl:
[comando ejecutado]

Respuesta:
[JSON response]

### Prueba 2: Evento en Kafka

Comando consumer:
[kafka-console-consumer...]

Output:
[JSON del evento]

### Prueba 3: Flujo completo (orden → inventory update)

[describir pasos y resultados]

## 9. BONUS: Consumer en product-service

[código de InventoryEventConsumer.java]

Screenshot de logs:
[logs mostrando eventos consumidos]
```

### Subir a Moodle

- Archivo: `SOLUCION.md`
- Fecha límite: [definir en clase]
- Formato: Markdown plano (NO PDF)

---

## Recursos de Ayuda

- [Cheatsheet Clase 6](../cheatsheet.md) - Comandos Kafka y configuración
- [FAQ Clase 6](../FAQ.md) - Troubleshooting de consumers
- [Labs de la clase](../labs/) - Referencia de código
- [Spring Kafka Docs](https://docs.spring.io/spring-kafka/reference/) - Documentación oficial

---

## Preguntas Frecuentes

**¿Puedo usar tipos diferentes a Integer para delta?**

No, debe ser `Integer` para permitir valores negativos y positivos.

**¿El endpoint PUT debe validar que el delta no deje stock negativo?**

Sí, agregar validación:
```java
int newStock = item.getAvailableStock() + request.getDelta();
if (newStock < 0) {
    throw new IllegalArgumentException("Resulting stock cannot be negative");
}
```

**¿Debo crear custom exceptions?**

No es obligatorio. Puedes usar `RuntimeException` con mensajes descriptivos (como en labs).

**¿El consumer en product-service debe hacer algo más que logging?**

No, solo logging es suficiente para el bonus. La idea es practicar consumers.

**¿Puedo agregar más campos a InventoryUpdatedEvent?**

Sí, pero los campos listados son obligatorios. Puedes agregar campos opcionales como `userId`, `notes`, etc.

---

## Rúbrica Detallada

### Código InventoryUpdatedEvent (15 pts)

- [5 pts] Todos los campos requeridos presentes
- [3 pts] Constructor vacío + constructor completo
- [3 pts] Getters y setters
- [2 pts] toString() implementado
- [2 pts] Package correcto

### Producer Implementado (15 pts)

- [5 pts] KafkaTemplate con tipos genéricos correctos
- [3 pts] Constructor injection (no @Autowired en fields)
- [3 pts] Logging con SLF4J
- [2 pts] Manejo asíncrono con whenComplete()
- [2 pts] Topic como constante

### Integración en processOrderPlaced() (20 pts)

- [10 pts] Evento publicado DESPUÉS de reservar stock
- [5 pts] Campos del evento llenados correctamente
- [3 pts] Reason = "ORDER_PLACED"
- [2 pts] No rompe funcionalidad existente

### Topic Creado (10 pts)

- [5 pts] Nombre correcto: ecommerce.inventory.updated
- [3 pts] 3 particiones
- [2 pts] Replication factor 1

### Endpoint PUT /api/inventory/{id}/adjust (25 pts)

- [8 pts] Endpoint funcional con path variable
- [5 pts] DTO AdjustStockRequest con validación
- [5 pts] Lógica correcta (calcular delta, guardar)
- [5 pts] Publica InventoryUpdatedEvent
- [2 pts] Manejo de errores (producto no existe)

### Compilación (10 pts)

- [10 pts] mvn clean install sin errores
- [0 pts] Si no compila

### Verificación (5 pts)

- [5 pts] Screenshot de kafka-console-consumer mostrando evento
- [0 pts] Si no se verifica

### BONUS: Consumer en product-service (+10 pts)

- [5 pts] @KafkaListener configurado correctamente
- [3 pts] Logging de eventos
- [2 pts] Configuración consumer en application.yml

---

## Consejos

1. **Empieza por el evento**: Crea la clase completa antes de usar en producer
2. **Prueba el producer solo**: Usa kafka-console-consumer para verificar
3. **Luego el endpoint**: Agrégalo después de que el producer funcione
4. **Bonus al final**: Solo si terminaste todo lo obligatorio

**Tiempo estimado**: 2-3 horas

Éxito!