# Tarea para Casa - Clase 5

## Objetivo

Consolidar los conocimientos de spring-kafka implementando eventos en tu propio microservicio o extendiendo los microservicios del curso.

---

## Actividad

Implementa **productores de eventos** en product-service y order-service siguiendo los laboratorios de la clase. Luego, realiza las siguientes extensiones:

### Parte 1: Extender product-service

**Tarea**: Agregar evento `ProductUpdatedEvent`

1. Crear clase `ProductUpdatedEvent` con:
   - `productId`
   - `name`
   - `price`
   - `updatedAt`

2. Crear topic `ecommerce.products.updated`:
```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic ecommerce.products.updated \
  --partitions 3 --replication-factor 1
```

3. Modificar `ProductEventProducer` para agregar método `publishProductUpdated()`

4. Modificar `ProductService.updateProduct()` para publicar el evento después de actualizar en BD

5. Probar:
   - Actualizar un producto vía REST
   - Verificar que el evento se publica con `kafka-console-consumer`

### Parte 2: Extender order-service

**Tarea**: Agregar evento `OrderCancelledEvent`

1. Crear clase `OrderCancelledEvent` con:
   - `orderId`
   - `reason`
   - `cancelledAt`

2. Crear topic `ecommerce.orders.cancelled`:
```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic ecommerce.orders.cancelled \
  --partitions 3 --replication-factor 1
```

3. Crear endpoint `DELETE /api/orders/{id}` en OrderController

4. Implementar lógica en OrderService para:
   - Cambiar estado de orden a CANCELLED en BD
   - Publicar `OrderCancelledEvent`

5. Probar:
   - Cancelar una orden vía REST
   - Verificar que el evento se publica

### Parte 3: Documentación

Crea un archivo `EVENTOS.md` en tu repositorio documentando:

1. **Topics creados**: Nombre, propósito, particiones
2. **Eventos publicados**: Estructura JSON de cada evento
3. **Productores**: Qué microservicio publica cada evento
4. **Pruebas realizadas**: Capturas o logs de eventos publicados

---

## Entrega

### Formato

1. Repositorio Git personal con:
   - Código de product-service modificado
   - Código de order-service modificado
   - Archivo `EVENTOS.md` con documentación
   - README.md con instrucciones de ejecución

2. Subir enlace del repositorio a Moodle

### Criterios de evaluación

| Criterio | Puntos |
|----------|--------|
| ProductUpdatedEvent implementado correctamente | 2.5 |
| Topic ecommerce.products.updated creado y funcional | 1.0 |
| OrderCancelledEvent implementado correctamente | 2.5 |
| Topic ecommerce.orders.cancelled creado y funcional | 1.0 |
| Logging adecuado en productores | 1.0 |
| Documentación EVENTOS.md completa | 1.5 |
| Código sigue estándares (dev.alefiengo, sin emojis) | 0.5 |
| **Total** | **10.0** |

---

## Desafío adicional (opcional)

### Agregar timestamp a todos los mensajes

Configura un header `timestamp` en todos los eventos publicados:

```java
ProducerRecord<String, ProductCreatedEvent> record =
    new ProducerRecord<>(topic, key, event);
record.headers().add("timestamp",
    String.valueOf(System.currentTimeMillis()).getBytes());

kafkaTemplate.send(record);
```

Verifica con:
```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic ecommerce.products.created \
  --from-beginning \
  --property print.headers=true
```

---

## Recursos de apoyo

- [Cheatsheet Clase 5](../cheatsheet.md)
- [FAQ Clase 5](../FAQ.md)
- [Spring Kafka Reference](https://docs.spring.io/spring-kafka/reference/html/)

---

## Fecha de entrega

Según calendario del curso en Moodle.

---

## Preguntas frecuentes

**¿Puedo usar mi propio dominio en lugar de e-commerce?**

Sí, puedes implementar eventos para un dominio propio (biblioteca, hotel, etc.) siempre que:
- Uses naming convention: `app.domain.event` (ej: `library.books.borrowed`)
- Implementes al menos 2 tipos de eventos
- Documentes todo en EVENTOS.md

**¿Debo implementar consumidores?**

No para esta tarea. Los consumidores se verán en la próxima clase. Por ahora solo productores y verificación con `kafka-console-consumer`.

**¿Puedo trabajar en equipo?**

No, esta tarea es individual. Cada estudiante debe tener su propio repositorio.
