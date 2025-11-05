# Proyecto Integrador · Spring Boot & Apache Kafka

El proyecto integrador del curso consiste en construir **progresivamente** un sistema de e-commerce basado en microservicios. A lo largo de las 8 clases, cada estudiante desarrollará los componentes del sistema mediante laboratorios prácticos.

**Importante**: No hay un repositorio separado de "proyecto integrador". El proyecto **ES** el trabajo acumulativo de todas las clases y labs.

---

## Objetivo

Construir un sistema de comercio electrónico que evoluciona desde un único microservicio (`product-service`) hasta una arquitectura completa de microservicios conectados mediante Apache Kafka, aplicando los conceptos de cada clase.

---

## Evolución del sistema por bloque

### Bloque 1: Spring Boot Fundamentals (Clases 1-4)

**Estado actual**: ✅ **Completado hasta Clase 4**

| Clase | Componente | Avances |
|-------|------------|---------|
| 1 | Setup | Entorno de desarrollo configurado |
| 2 | `product-service` | REST API + JPA + PostgreSQL |
| 3 | `product-service` | Arquitectura en capas, DTOs, validaciones, relación Product-Category |
| 4 | `product-service` | Perfiles (dev/prod), Actuator, variables de entorno. Kafka instalado (Docker) |

**Resultado**: Un microservicio completo y production-ready con catálogo de productos.

### Bloque 2: Apache Kafka y Mensajería (Clases 5-6)

**Estado**: 📋 **Planificado**

| Clase | Componentes | Avances |
|-------|-------------|---------|
| 5 | `product-service` + `order-service` | Productores y consumidores Kafka, eventos de productos y órdenes |
| 6 | `order-service` + `inventory-service` | Comunicación event-driven completa, validación de stock |

**Resultado**: Sistema de 3 microservicios con mensajería asíncrona.

### Bloque 3: Streams, Seguridad y Cierre (Clases 7-8)

**Estado**: 📋 **Planificado**

| Clase | Componentes | Avances |
|-------|-------------|---------|
| 7 | `analytics-service` | Kafka Streams, CQRS, agregación de datos |
| 8 | Todos los servicios | JWT, Spring Security, proyecto final |

**Resultado**: Sistema completo e-commerce con 4 microservicios securizados.

---

## Arquitectura final del sistema

Al completar el curso, habrás construido:

### Microservicios

1. **product-service** (Clases 2-4)
   - Catálogo de productos y categorías
   - REST API completa (CRUD)
   - Eventos: `ecommerce.products.created`, `ecommerce.products.updated`

2. **order-service** (Clases 5-6)
   - Gestión de órdenes de compra
   - Publica: `ecommerce.orders.placed`
   - Consume: `ecommerce.orders.confirmed`, `ecommerce.orders.rejected`

3. **inventory-service** (Clase 6)
   - Control de inventario y stock
   - Consume: `ecommerce.orders.placed`
   - Publica: `ecommerce.orders.confirmed`, `ecommerce.orders.rejected`, `ecommerce.inventory.updated`

4. **analytics-service** (Clase 7)
   - Kafka Streams para agregación
   - CQRS: Vista materializada de métricas de negocio
   - Dashboard de ventas y productos más vendidos

### Infraestructura

- **Apache Kafka**: Bus de eventos (topics del dominio e-commerce)
- **PostgreSQL**: Base de datos independiente por microservicio (`product_db`, `order_db`, `inventory_db`, `analytics_db`)
- **Docker Compose**: Infraestructura completa dockerizada
- **Spring Boot Actuator**: Monitoreo y health checks

### Seguridad (Clase 8)

- **JWT**: Autenticación en todos los servicios
- **Spring Security**: Protección de endpoints
- **API Gateway** (opcional): Punto único de entrada

---

## Entrega final

La evaluación del proyecto integrador se realizará en la **Clase 8**, donde cada estudiante presentará:

1. **Código fuente**: Repositorio personal con los 4 microservicios
2. **Docker Compose**: Sistema completo ejecutable
3. **Colección Postman**: Pruebas de todos los endpoints
4. **Documentación**: README con arquitectura e instrucciones

**Criterios de evaluación**: Se publicarán en la Clase 8 junto con la rúbrica detallada.

---

## Trabajo acumulativo

Cada clase construye sobre la anterior. Es fundamental:

- ✅ Completar los labs de cada clase antes de avanzar
- ✅ Realizar las tareas para casa
- ✅ Mantener tu repositorio personal actualizado
- ✅ Hacer preguntas en el foro si algo no funciona

**No hay atajos**: El proyecto integrador ES la suma de todo tu trabajo en el curso.
