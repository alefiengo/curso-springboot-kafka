# Proyecto Final · E-commerce con Spring Boot y Kafka

Este proyecto final solo requiere **pulir y presentar** el trabajo que ya hicimos en clase. Tendrán **1 semana** desde hoy para ajustar lo que falte, documentarlo y entregar el repositorio final.

---

## 1. Cómo se Evalúa (100 pts)

Todo el puntaje se toma **del repositorio público** (GitHub/GitLab). Si la documentación es confusa o no se puede reproducir, el puntaje de cada criterio se reduce aunque el código funcione en tu máquina.

| # | Criterio | Puntos | ¿Qué reviso? |
|---|-------|--------|--------------|
| 1 | Documentación clara + evidencias | 25 | README raíz, pasos reproducibles, capturas/logs, enlaces funcionando |
| 2 | Arquitectura y estructura | 20 | 3 microservicios pedidos, misma convención de paquetes, capas completas |
| 3 | Funcionalidad REST + validaciones | 20 | CRUD + estados, Bean Validation, GlobalExceptionHandler funcionando |
| 4 | Integración Kafka end-to-end | 15 | Producers/consumers configurados, topics correctos, type mapping |
| 5 | Bases de datos y modelos | 10 | 3 bases separadas, entidades/documentación actualizada |
| 6 | Configuración / variables | 10 | `application.yml` con fallbacks, perfiles y uso de env vars |

> **Regla de oro**: si un evaluador no puede seguir tu README para levantar y probar el sistema, no puede asignar el puntaje completo.

---

## 2. Plazos y Entregables

- **Duración**: 1 semana.
- **Entrega**: link al repositorio en Moodle que contiene el README con instrucciones.
- **Revisión**: clonamos el repo, seguimos tu guía, verificamos Postman y Kafka.Los 3 servicios deben compilar (`mvn clean install`) y correr (`mvn spring-boot:run`) sin errores.

Checklist express:

1. Ajusta los microservicios (si falta algo de la lista obligatoria).
2. Documenta TODO (README raíz + Postman + capturas/logs).
3. Sube a GitHub/GitLab y verifica que el repo se clone limpio.

---

## 3. Requisitos obligatorios (por criterio)

### 3.1 Documentación (25 pts)

- README raíz con: propósito breve, diagrama sencillo, stack usado, requisitos previos, pasos exactos para compilar/ejecutar, tabla de endpoints, flujo Kafka, modelo de datos, enlace a Postman.
- Carpeta `postman/` con la colección actualizada.
- Evidencias visuales: capturas de Docker/Kafka, logs de consumo, screenshot de Postman o CLI.
- Si tienes configuraciones especiales (perfiles, variables, scripts), documenta el comando exacto.

### 3.2 Arquitectura y estructura (20 pts)

- Tres microservicios obligatorios: `product-service`, `order-service`, `inventory-service`.
- Paquetes base con tu dominio (`com.tuempresa.productservice`, etc.).
- Capas mínimas en cada servicio:
  - `controller`, `service`, `repository`, `model`, `dto`, `mapper`, `exception`, `kafka`.
- Archivos de configuración en `src/main/resources` (`application.yml`, `application-dev.yml`, `application-prod.yml`, `ValidationMessages.properties`).

### 3.3 Funcionalidad REST y validaciones (20 pts)

- **product-service**: CRUD completo de productos + categoría, validaciones en DTOs, eventos `ecommerce.products.created`.
- **order-service**: creación/listado/búsqueda de órdenes con estados PENDING/CONFIRMED/CANCELLED, producción de `ecommerce.orders.placed`, consumo de confirmaciones/cancelaciones.
- **inventory-service**: administración de stock (crear, listar, consultar por productId), consumo de órdenes y publicación de confirmaciones/cancelaciones.
- Bean Validation con mensajes en `ValidationMessages.properties`.
- `GlobalExceptionHandler` con respuestas claras para errores comunes.

### 3.4 Kafka (15 pts)

- Topics obligatorios (5 particiones, 1 réplica):
  - `ecommerce.products.created`
  - `ecommerce.orders.placed`
  - `ecommerce.orders.confirmed`
  - `ecommerce.orders.cancelled`
- Comandos documentados para crear/validar topics (`docker exec kafka ... kafka-topics ...`).
- Producers y consumers configurados con `spring.kafka` y `spring.json.type.mapping`.
- Evidencia en README (logs o captura) mostrando el flujo completo: orden → validación inventario → confirmación/cancelación.

### 3.5 Bases de datos y modelos (10 pts)

- Tres bases PostgreSQL (`ecommerce`, `ecommerce_orders`, `ecommerce_inventory`). Explica cómo se crean (scripts SQL, docker exec, etc.).
- Entidades con relaciones correctas (Product–Category 1:N, etc.).
- Documenta el modelo de datos (tabla o diagrama) y cualquier constraint relevante.

### 3.6 Configuración y variables (10 pts)

- Cada `application.yml` debe usar variables con fallback:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
```

- Ajusta el `DB_NAME` por servicio (`ecommerce`, `ecommerce_orders`, `ecommerce_inventory`).
- Usa perfiles (`spring.profiles.active`) para dev/prod y documenta cómo activarlos.

---

## 4. Puntos Extra (Opcionales)

Suma hasta **+10 pts** si implementas y documentas alguno de estos extras.

1. **Perfiles avanzados (+5 pts)**: dev/test/prod con configuraciones diferenciadas (logging, Kafka, DB). Explica cómo se activa cada uno.
2. **Documentación API con Swagger (+5 pts)**: integra Springdoc/OpenAPI, documenta todos los endpoints y enlaza la URL (`/swagger-ui.html`) en el README.

> Los puntos extra se cuentan solo si están claramente explicados (qué hiciste, cómo se prueba, capturas o logs).

---

## 5. Recursos Útiles

- Código desarrollado en clase (repos individuales de cada estudiante).
- Apuntes o notas personales.
- Documentación oficial:
  - [Spring Boot](https://docs.spring.io/spring-boot/)
  - [Spring Kafka](https://docs.spring.io/spring-kafka/reference/html/)
  - [Apache Kafka](https://kafka.apache.org/documentation/)

---

## 6. Checklist Final Antes de Enviar

- [ ] Repo público listo y README actualizado.
- [ ] `mvn clean install` OK en cada microservicio.
- [ ] `mvn spring-boot:run` OK (o comando equivalente documentado).
- [ ] Topics Kafka creados y verificados.
- [ ] Colección Postman incluida y probada.
- [ ] Evidencias anexadas (capturas/logs).
- [ ] Link compartido en Moodle dentro del plazo.

---

**¡Éxitos!** Recuerden que este proyecto resume todo lo que ya construimos. Concéntrense en ordenar el código, validar los flujos y explicarlo claramente en la documentación. Si la guía es reproducible, la revisión es directa.
