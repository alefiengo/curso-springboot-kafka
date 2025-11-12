# Tarea - Clase 8: Preparar Proyecto Integrador

---

## Objetivo

Preparar una propuesta detallada para el **Proyecto Integrador Final** del curso, aplicando todos los conocimientos adquiridos en las 8 clases sobre microservicios, Spring Boot, y Apache Kafka.

---

## Descripción

El proyecto integrador es tu oportunidad de demostrar dominio completo de arquitecturas de microservicios event-driven. Diseñarás e implementarás un sistema completo desde cero, aplicando patrones y prácticas aprendidas en el curso.

---

## Opciones de dominio

Elige **UNO** de los siguientes dominios:

### Opción 1: Sistema de Delivery

**Microservicios sugeridos**:
- restaurant-service: Gestión de restaurantes y menús
- order-service: Órdenes de clientes
- delivery-service: Asignación y tracking de repartidores
- payment-service: Procesamiento de pagos
- notification-service: Notificaciones a clientes

**Eventos Kafka sugeridos**:
- delivery.orders.placed
- delivery.orders.assigned
- delivery.orders.delivered
- delivery.payments.processed
- delivery.restaurants.menu-updated

### Opción 2: Sistema Bancario

**Microservicios sugeridos**:
- account-service: Gestión de cuentas
- transaction-service: Transacciones (depósitos, retiros, transferencias)
- fraud-detection-service: Detección de fraude (Kafka Streams)
- notification-service: Alertas a clientes
- analytics-service: Reportes y estadísticas

**Eventos Kafka sugeridos**:
- banking.transactions.created
- banking.transactions.completed
- banking.fraud.detected
- banking.accounts.balance-updated
- banking.notifications.sent

### Opción 3: Sistema de Reservas

**Microservicios sugeridos**:
- hotel-service: Gestión de hoteles y habitaciones
- flight-service: Gestión de vuelos
- booking-service: Reservas de viajes
- payment-service: Procesamiento de pagos
- analytics-service: Estadísticas de reservas

**Eventos Kafka sugeridos**:
- travel.bookings.created
- travel.bookings.confirmed
- travel.bookings.cancelled
- travel.payments.completed
- travel.inventory.updated

### Opción 4: Sistema de Salud

**Microservicios sugeridos**:
- patient-service: Gestión de pacientes
- appointment-service: Agenda de citas
- doctor-service: Gestión de médicos
- prescription-service: Recetas médicas
- notification-service: Recordatorios de citas

**Eventos Kafka sugeridos**:
- health.appointments.scheduled
- health.appointments.completed
- health.prescriptions.issued
- health.patients.registered
- health.notifications.sent

---

## Requisitos del proyecto

### Arquitectura

**Mínimo requerido**:
- 3-4 microservicios comunicándose vía Kafka
- Al menos 4 Kafka topics
- 2-3 bases de datos PostgreSQL independientes
- Event-driven architecture con patrones (saga, CQRS, etc.)

**Opcional (puntos extra)**:
- Kafka Streams para analytics/agregaciones
- CQRS read model
- Saga pattern para transacciones distribuidas

### Tecnología

**Obligatorio**:
- Spring Boot 3.x
- Apache Kafka
- PostgreSQL
- Docker Compose
- Maven

**Configuración**:
- Spring Boot Profiles (dev/prod)
- Spring Boot Actuator
- Variables de entorno
- Logging estructurado

**Opcional**:
- JWT Security
- Swagger/OpenAPI documentation
- Integration tests

---

## Entregables

### 1. Documento de propuesta (PDF o Markdown)

Debe incluir:

#### 1.1 Descripción del dominio
- ¿Qué problema resuelve tu sistema?
- ¿Quiénes son los usuarios?
- Funcionalidades principales

#### 1.2 Arquitectura de microservicios

Diagrama mostrando:
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  service-1   │     │  service-2   │     │  service-3   │
│  (puerto)    │     │  (puerto)    │     │  (puerto)    │
│  PostgreSQL  │     │  PostgreSQL  │     │  PostgreSQL  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ↓
                       ┌─────────┐
                       │  Kafka  │
                       │ Topics  │
                       └─────────┘
```

#### 1.3 Diagrama de flujo de eventos

Ejemplo para Sistema de Delivery:
```
1. Cliente crea orden
   → order-service publica: delivery.orders.placed

2. delivery-service consume evento
   → Asigna repartidor
   → Publica: delivery.orders.assigned

3. delivery-service tracking
   → Repartidor recoge orden
   → Publica: delivery.orders.picked-up

4. delivery-service completa entrega
   → Publica: delivery.orders.delivered

5. notification-service consume todos los eventos
   → Envía notificaciones a cliente
```

#### 1.4 Kafka Topics

Tabla de topics:

| Topic | Producer | Consumer(s) | Descripción |
|-------|----------|-------------|-------------|
| domain.entity.event | service-a | service-b, service-c | Descripción del evento |
| ... | ... | ... | ... |

#### 1.5 Endpoints REST principales

Por cada microservicio:

**service-1 (puerto 8080)**:
- POST `/api/resource` - Crear recurso
- GET `/api/resource/{id}` - Obtener recurso
- ...

#### 1.6 Base de datos

Por cada microservicio:
- Nombre de la base de datos
- Tablas principales
- Relaciones (1:N, N:M)

---

### 2. Criterios de evaluación

| Criterio | Puntos | Descripción |
|----------|--------|-------------|
| **Arquitectura** | 25 | Diseño de microservicios, separación de responsabilidades |
| **Eventos Kafka** | 20 | Diseño de topics, flujo de eventos lógico |
| **Viabilidad** | 15 | El proyecto es implementable en 2-3 semanas |
| **Creatividad** | 10 | Propuesta innovadora, no copia exacta del curso |
| **Documentación** | 15 | Claridad, diagramas, explicaciones |
| **Patrones** | 15 | Uso de saga, CQRS, event sourcing (cuando aplique) |
| **TOTAL** | 100 | |

---

## Formato de entrega

### Documento de propuesta

**Formato**: PDF o Markdown

**Secciones obligatorias**:
1. Portada (nombre, dominio elegido)
2. Descripción del dominio
3. Arquitectura de microservicios (diagrama)
4. Flujo de eventos (diagrama secuencial)
5. Kafka Topics (tabla)
6. Endpoints REST (por microservicio)
7. Modelo de datos (por microservicio)

**Longitud**: 5-10 páginas

### Dónde entregar

- **Repositorio personal**: Crear branch `proyecto-integrador-propuesta`
- **Moodle**: Subir PDF con propuesta
- **Fecha límite**: [A definir por instructor]

---

## Ejemplo de propuesta

### Sistema de Delivery - Propuesta

**1. Descripción**

Sistema de delivery para restaurantes que permite a clientes ordenar comida, asignar repartidores automáticamente, y trackear entregas en tiempo real.

**2. Microservicios**

- restaurant-service (8080): 5 restaurantes, 30 menús
- order-service (8081): Órdenes de clientes
- delivery-service (8082): 10 repartidores, asignación automática
- notification-service (8083): Emails/SMS a clientes

**3. Eventos Kafka**

| Topic | Producer | Consumer | Descripción |
|-------|----------|----------|-------------|
| delivery.orders.placed | order-service | delivery-service | Nueva orden creada |
| delivery.orders.assigned | delivery-service | notification-service | Orden asignada a repartidor |
| delivery.orders.delivered | delivery-service | order-service, notification-service | Orden entregada |

**4. Flujo principal**

1. Cliente crea orden → order-service → `delivery.orders.placed`
2. delivery-service asigna repartidor → `delivery.orders.assigned`
3. Repartidor completa entrega → `delivery.orders.delivered`
4. notification-service notifica a cliente

**5. Endpoints principales**

**order-service (8081)**:
- POST `/api/orders` - Crear orden
- GET `/api/orders/{id}` - Ver orden
- GET `/api/orders/customer/{customerId}` - Órdenes de cliente

**delivery-service (8082)**:
- POST `/api/deliveries/assign` - Asignar repartidor
- GET `/api/deliveries/tracking/{orderId}` - Trackear orden
- PATCH `/api/deliveries/{id}/status` - Actualizar estado

---

## Preguntas frecuentes

### ¿Puedo cambiar de dominio después de la propuesta?

Solo con aprobación del instructor y antes de iniciar implementación.

### ¿Puedo trabajar en equipo?

El proyecto es individual. Sin embargo, puedes discutir ideas con compañeros.

### ¿Cuánto código debo entregar ahora?

**En esta tarea**: SOLO propuesta (documento). Sin código.

**En el proyecto final** (2-3 semanas): Código completo funcionando.

### ¿Puedo usar un dominio diferente?

Sí, pero debe ser aprobado por el instructor. Envía propuesta alternativa.

### ¿Es obligatorio usar todos los temas del curso?

Mínimo requerido:
- ✅ Microservicios con Spring Boot
- ✅ Kafka para comunicación
- ✅ PostgreSQL
- ✅ Docker Compose

Opcional pero recomendado:
- Kafka Streams
- CQRS
- JWT Security

---

## Recursos

### Inspiración

- [Event-Driven Architecture - Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html)
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)

### Ejemplos del curso

- E-commerce (Clases 6-7): product, order, inventory, analytics
- Usa como referencia pero NO copies exactamente

### Documentación del curso

- Revisar labs de todas las clases
- Consultar cheatsheets y FAQs
- Ver diagramas de arquitectura

---

## Cronograma sugerido (Post-Clase 8)

**Semana 1**:
- Preparar propuesta
- Diseñar arquitectura
- Definir eventos Kafka
- **Entrega**: Propuesta aprobada

**Semana 2**:
- Implementar microservicios básicos
- Configurar Kafka topics
- CRUD completo

**Semana 3**:
- Integración Kafka
- Testing end-to-end
- Documentación final
- **Entrega**: Proyecto completo

---

**¡Éxito con tu propuesta!** Este es el momento de demostrar todo lo aprendido en el curso.

Consulta [PROYECTO_INTEGRADOR.md](../../../PROYECTO_INTEGRADOR.md) para más detalles del proyecto final.
