# Tarea - Clase 4: Perfiles, Configuración y Apache Kafka

## Objetivo

Aplicar los conceptos aprendidos en la Clase 4 para preparar `product-service` para producción con perfiles, observabilidad con Actuator, externalización de configuración, y desplegar una infraestructura de Kafka lista para integración en la siguiente clase.

---

## Contexto

Has completado los laboratorios de la Clase 4 y ahora tu `product-service` necesita estar production-ready antes de integrarlo con Kafka en la Clase 5. Además, debes preparar la infraestructura de Kafka con los topics del dominio e-commerce.

---

## Requisitos previos

- Clase 4 completada (Labs 00-05)
- `product-service` de la Clase 3 funcionando
- Docker y Docker Compose instalados
- Maven y Java 17

---

## Parte 1: Configuración de perfiles (40%)

### Tarea 1.1: Crear perfiles dev y prod

Configura `product-service` con dos perfiles diferenciados:

**Perfil `dev` (desarrollo)**:

- Base de datos: `ecommerce_dev`
- Puerto: `8080`
- Mostrar SQL: `true`
- Formatear SQL: `true`
- Nivel de log: `DEBUG` para `dev.alefiengo`
- Nivel de log: `DEBUG` para `org.springframework.web`
- Nivel de log: `DEBUG` para `org.hibernate.SQL`

**Perfil `prod` (producción)**:

- Base de datos: externalizada con variable `${DB_NAME:ecommerce}`
- Puerto: externalizado con variable `${SERVER_PORT:8080}`
- Mostrar SQL: `false`
- Nivel de log: `INFO` para `dev.alefiengo`
- Nivel de log: `WARN` para `org.springframework.web`
- Nivel de log: `WARN` para `org.hibernate`

**Archivos a entregar**:

- `src/main/resources/application.yml` (configuración común)
- `src/main/resources/application-dev.yml`
- `src/main/resources/application-prod.yml`

### Tarea 1.2: Validar perfiles

Ejecuta la aplicación con ambos perfiles y documenta:

1. Logs de inicio mostrando perfil activo
2. Diferencia en logs de SQL entre `dev` y `prod`
3. Captura de pantalla de `GET /api/products` con logs de cada perfil

**Comandos a ejecutar**:

```bash
# Desarrollo
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Producción
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

---

## Parte 2: Spring Boot Actuator (30%)

### Tarea 2.1: Integrar Actuator

Agrega Spring Boot Actuator a `product-service` y configura los siguientes endpoints:

**Endpoints a habilitar**:

- `/actuator/health` (con detalles completos)
- `/actuator/info`
- `/actuator/metrics`

**NO exponer**:

- `/actuator/env` (seguridad)

**Configuración diferenciada por perfil**:

**Perfil `dev`**:

- Exponer: `health,info,metrics,env` (incluye env para debugging)
- Health details: `always`

**Perfil `prod`**:

- Exponer: `health,info,metrics` (SIN env)
- Health details: `when-authorized`

### Tarea 2.2: Agregar información personalizada

Configura el endpoint `/actuator/info` con:

```yaml
info:
  app:
    name: product-service
    description: Microservicio de catálogo de productos para e-commerce
    version: 1.0.0
    author: [Tu Nombre]
  contact:
    email: [Tu Email]
  environment: ${spring.profiles.active:default}
```

### Tarea 2.3: Crear custom health indicator

Implementa un health indicator personalizado que verifique:

- Si hay al menos 1 producto en la base de datos → `UP`
- Si la base de datos está vacía → `DOWN` con mensaje "Catálogo vacío"

**Archivo a crear**:

`src/main/java/dev/alefiengo/productservice/health/ProductHealthIndicator.java`

### Tarea 2.4: Validar Actuator

Ejecuta la aplicación y documenta:

1. Captura de `GET /actuator/health` mostrando componentes
2. Captura de `GET /actuator/info` con información personalizada
3. Captura de `GET /actuator/metrics/jvm.memory.used`
4. Captura de `GET /actuator/metrics/http.server.requests` después de hacer varios requests

---

## Parte 3: Variables de entorno (15%)

### Tarea 3.1: Externalizar configuración

Modifica `application.yml` para que TODAS las configuraciones críticas usen variables de entorno con fallbacks:

**Variables a externalizar**:

- `DB_HOST` (default: `localhost`)
- `DB_PORT` (default: `5432`)
- `DB_NAME` (default: `ecommerce`)
- `DB_USER` (default: `postgres`)
- `DB_PASSWORD` (default: `postgres`)
- `SERVER_PORT` (default: `8080`)

**Archivo a modificar**:

`src/main/resources/application.yml`

### Tarea 3.2: Crear archivo .env

Crea un archivo `.env` en la raíz del proyecto con:

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce_dev
DB_USER=postgres
DB_PASSWORD=postgres
SERVER_PORT=8080
SPRING_PROFILES_ACTIVE=dev
```

**IMPORTANTE**: Agregar `.env` al `.gitignore`.

### Tarea 3.3: Validar variables

Ejecuta la aplicación con variables de entorno diferentes:

```bash
export DB_NAME=product_db_test
export SERVER_PORT=8090
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

Documenta que la aplicación:

1. Se conecta a la base de datos `product_db_test`
2. Escucha en puerto `8090`

---

## Parte 4: Infraestructura de Kafka (15%)

### Tarea 4.1: Desplegar Kafka con Docker Compose

Crea un directorio `kafka-infrastructure/` en la raíz del proyecto y agrega:

**Archivo**: `kafka-infrastructure/docker-compose.yml`

Configuración:

- Zookeeper en puerto `2181`
- Kafka en puerto `9092`
- `KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"`
- Volúmenes para persistencia

### Tarea 4.2: Crear topics del dominio e-commerce

Usando Kafka CLI, crea los siguientes topics:

| Topic | Particiones | Replication Factor |
|-------|-------------|-------------------|
| `ecommerce.products.created` | 3 | 1 |
| `ecommerce.products.updated` | 3 | 1 |
| `ecommerce.orders.placed` | 3 | 1 |
| `ecommerce.orders.confirmed` | 3 | 1 |
| `ecommerce.orders.cancelled` | 3 | 1 |
| `ecommerce.inventory.updated` | 3 | 1 |

### Tarea 4.3: Verificar topics

Documenta:

1. Salida de `kafka-topics --list` mostrando todos los topics creados
2. Salida de `kafka-topics --describe --topic ecommerce.products.created`
3. Captura de logs de Kafka confirmando inicio exitoso

---

## Entrega

### Formato de entrega

Crear un repositorio Git privado con la siguiente estructura:

```
product-service/
├── src/
│   ├── main/
│   │   ├── java/dev/alefiengo/productservice/
│   │   │   ├── controller/
│   │   │   ├── service/
│   │   │   ├── repository/
│   │   │   ├── model/
│   │   │   ├── health/
│   │   │   │   └── ProductHealthIndicator.java
│   │   │   └── Application.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       └── application-prod.yml
│   └── test/
├── kafka-infrastructure/
│   └── docker-compose.yml
├── .env.example
├── .gitignore
├── pom.xml
└── README.md (documentación de la tarea)
```

### Documentación requerida (README.md)

Tu README.md debe incluir:

#### 1. Instrucciones de ejecución

```markdown
## Cómo ejecutar product-service

### Con perfil dev
```bash
export $(cat .env | xargs)
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

### Con perfil prod
```bash
export DB_HOST=localhost
export DB_NAME=ecommerce
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```
```

#### 2. Endpoints disponibles

Lista todos los endpoints:

- REST API: `/api/products`, `/api/categories`
- Actuator: `/actuator/health`, `/actuator/info`, `/actuator/metrics`

#### 3. Configuración de Kafka

Instrucciones para iniciar Kafka:

```bash
cd kafka-infrastructure
docker compose up -d
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list
```

#### 4. Capturas de pantalla

Incluir capturas de:

- Actuator health check con componentes UP
- Actuator info con datos personalizados
- Kafka topics listados
- Product health indicator mostrando estado

#### 5. Reflexión personal

Responde (mínimo 200 palabras):

1. ¿Cuál es la principal ventaja de usar perfiles de Spring Boot?
2. ¿Por qué es importante NO exponer `/actuator/env` en producción?
3. ¿Qué ventajas tiene externalizar la configuración con variables de entorno?
4. ¿Qué diferencias observaste entre usar Kafka CLI desde el contenedor vs instalarlo localmente?
5. ¿Qué estrategia usarías para definir el número de particiones en un topic de producción?

---

## Criterios de evaluación

| Criterio | Puntaje | Descripción |
|----------|---------|-------------|
| **Perfiles (40%)** | | |
| Configuración correcta de perfiles dev/prod | 15 | Archivos yml bien estructurados |
| Validación de perfiles | 10 | Evidencia de ejecución con ambos perfiles |
| Diferenciación de logs | 10 | Logs muestran diferencias claras |
| Documentación | 5 | README explica cómo usar perfiles |
| **Actuator (30%)** | | |
| Integración de Actuator | 10 | Dependencia y configuración correcta |
| Información personalizada | 5 | Info endpoint con datos propios |
| Custom health indicator | 10 | Implementación funcional |
| Validación de endpoints | 5 | Capturas de todos los endpoints |
| **Variables de entorno (15%)** | | |
| Externalización correcta | 5 | Todas las variables con fallbacks |
| Archivo .env | 3 | Configuración local funcional |
| .gitignore | 2 | .env ignorado correctamente |
| Validación | 5 | Evidencia de uso de variables |
| **Kafka (15%)** | | |
| Docker Compose | 5 | Configuración correcta de Zookeeper y Kafka |
| Topics creados | 5 | 6 topics con configuración correcta |
| Validación | 5 | Evidencia de topics listados y descritos |
| **TOTAL** | **100** | |

---

## Bonus (opcional, hasta +10 puntos)

### Bonus 1: Métrica personalizada (+3 puntos)

Implementa una métrica custom que cuente:

- Total de productos creados
- Total de productos actualizados
- Total de productos eliminados

Usar `Counter` de Micrometer.

### Bonus 2: Múltiples brokers de Kafka (+4 puntos)

Modifica `docker-compose.yml` para tener 3 brokers de Kafka y recrea los topics con `--replication-factor 3`.

Documenta la configuración y demuestra que las particiones tienen réplicas en diferentes brokers.

### Bonus 3: Kafka UI (+3 puntos)

Agrega [Kafka UI](https://github.com/provectus/kafka-ui) a tu `docker-compose.yml` y proporciona capturas de pantalla de:

- Topics visualizados en UI
- Configuración de un topic
- Brokers conectados

---

## Fecha de entrega

**[Fecha definida por el instructor]**

**Método de entrega**:

1. Subir código a repositorio Git privado
2. Compartir acceso con el instructor
3. Subir README.md con documentación y capturas a Moodle

---

## Recursos de ayuda

- [Labs de la Clase 4](../labs/)
- [Cheatsheet - Clase 4](../cheatsheet.md)
- [FAQ - Clase 4](../FAQ.md)
- [Spring Boot Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Kafka Documentation](https://kafka.apache.org/documentation/)

---

**Última actualización**: Noviembre 2025
**Curso**: Spring Boot & Apache Kafka - i-Quattro
