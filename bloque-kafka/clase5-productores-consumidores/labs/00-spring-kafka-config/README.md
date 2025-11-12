# Lab 00: Configurar spring-kafka en product-service

## Objetivo

Integrar la dependencia spring-kafka en el proyecto product-service existente, configurar el productor de Kafka mediante application.yml y verificar que KafkaTemplate se puede inyectar correctamente como bean de Spring.

---

## Comandos a ejecutar

```bash
# 0. PREREQUISITO: Verificar que Kafka y PostgreSQL están corriendo
docker compose ps
# Debería mostrar kafka y postgres "Up"
# Si no están corriendo, navegar a ~/workspace/kafka-infrastructure y ejecutar:
# docker compose up -d

# 1. Verificar que la base de datos 'ecommerce' existe (creada en Clase 2)
docker exec -it postgres psql -U postgres -c "\l" | grep ecommerce

# Si no existe, crearla:
# docker exec -it postgres psql -U postgres -c "CREATE DATABASE ecommerce;"

# 2. Navegar al directorio del proyecto product-service
# (Asume que product-service está en el directorio de trabajo de las clases 2-3)
cd ~/workspace/product-service

# 3. Editar pom.xml para agregar dependencia spring-kafka
# (Ver sección "Desglose del comando" para el contenido exacto)

# 4. Recargar dependencias Maven
mvn clean install

# 5. Editar src/main/resources/application.yml
# (Ver sección "Desglose del comando" para configuración)

# 6. Ejecutar la aplicación para verificar que inicia sin errores
mvn spring-boot:run

# 7. Verificar logs de inicio (buscar "KafkaProducer")
# Deberías ver: "Started KafkaProducerFactory"
```

---

## Desglose del comando

### 1. Dependencia Maven (pom.xml)

Agregar dentro de `<dependencies>`:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

| Elemento | Descripción |
|----------|-------------|
| `groupId` | Organización propietaria (Spring Framework) |
| `artifactId` | Nombre del artefacto spring-kafka |
| `version` | Heredada de spring-boot-starter-parent |

**¿Por qué no especificamos version?**

Spring Boot ya define la versión compatible de spring-kafka en su BOM (Bill of Materials), garantizando compatibilidad entre todas las dependencias de Spring.

### 2. Configuración del Producer (application.yml)

Agregar al final del archivo:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        spring.json.type.mapping: productCreatedEvent:dev.alefiengo.productservice.kafka.event.ProductCreatedEvent
```

| Propiedad | Valor | Descripción |
|-----------|-------|-------------|
| `bootstrap-servers` | `${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}` | Dirección del broker de Kafka (variable de entorno con fallback) |
| `key-serializer` | `StringSerializer` | Convierte la clave a String |
| `value-serializer` | `JsonSerializer` | Convierte el evento a JSON automáticamente |
| `spring.json.type.mapping` | `productCreatedEvent:{full.package}` | **CRÍTICO**: Mapea el nombre del evento al tipo Java para deserialización en consumers |

**JsonSerializer vs StringSerializer**:

- `JsonSerializer`: Spring convierte automáticamente objetos Java a JSON
- `StringSerializer`: Requeriría serializar manualmente con ObjectMapper

### 3. Verificación de KafkaTemplate

Spring Boot autoconfigura `KafkaTemplate` como bean cuando detecta:

1. Dependencia spring-kafka en classpath
2. Configuración `spring.kafka.bootstrap-servers` en application.yml

No se requiere clase `@Configuration` adicional.

---

## Explicación detallada

### Paso 1: ¿Qué es spring-kafka?

**spring-kafka** es el proyecto oficial de Spring para integrar Apache Kafka con aplicaciones Spring Boot. Proporciona:

- **KafkaTemplate**: Clase principal para enviar mensajes (análogo a JdbcTemplate)
- **@KafkaListener**: Anotación para consumidores
- **Autoconfiguración**: Spring Boot configura automáticamente productores y consumidores
- **Serialización JSON**: Convierte objetos Java a JSON sin código adicional

**Ventajas sobre Kafka client nativo**:

| Aspecto | Kafka Client Nativo | spring-kafka |
|---------|---------------------|--------------|
| Configuración | Properties manual | application.yml |
| Inyección | `new KafkaProducer<>()` | `@Autowired KafkaTemplate` |
| Serialización JSON | Manual con ObjectMapper | Automática con JsonSerializer |
| Testing | Kafka embebido manual | `@EmbeddedKafka` |

### Paso 2: Agregar dependencia

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

Al agregar esta dependencia:

1. Maven descarga `spring-kafka` y `kafka-clients`
2. Spring Boot detecta la librería en classpath
3. Activa autoconfiguración `KafkaAutoConfiguration`
4. Crea bean `KafkaTemplate` listo para inyectar

**Verificar descarga**:

```bash
mvn dependency:tree | grep kafka
```

Deberías ver:

```
[INFO] +- org.springframework.kafka:spring-kafka:jar:3.x.x
[INFO] +- org.apache.kafka:kafka-clients:jar:3.x.x
```

### Paso 3: Configurar Producer

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

**¿Qué hace cada propiedad?**

**bootstrap-servers**:

- Dirección del broker de Kafka
- Puede ser lista: `localhost:9092,localhost:9093`
- Producer se conecta aquí para enviar mensajes
- Usa variable de entorno `${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}` para flexibilidad

**¿Cómo configurar las variables de entorno?**

```bash
# Opción 1: Variable de entorno del sistema
export KAFKA_BOOTSTRAP_SERVERS=localhost:9093
mvn spring-boot:run

# Opción 2: Argumento en línea de comandos
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.kafka.bootstrap-servers=localhost:9093"

# Opción 3: Para Docker/Kubernetes
docker run -e KAFKA_BOOTSTRAP_SERVERS=kafka:9092 product-service
```

**key-serializer**:

- Convierte la clave del mensaje a bytes
- Usamos `StringSerializer` porque nuestras claves serán String (productId, orderId)
- La clave determina a qué partición va el mensaje

**value-serializer**:

- Convierte el valor (nuestro evento) a bytes
- `JsonSerializer` de Spring convierte objetos Java a JSON automáticamente
- Ejemplo: `ProductCreatedEvent` → `{"productId":"123","name":"Laptop",...}`

**¿Por qué JSON y no Avro/Protobuf?**

JSON es más simple para aprendizaje:

- ✅ Legible para humanos
- ✅ No requiere Schema Registry
- ✅ Fácil de debuggear con kafka-console-consumer
- ❌ Mayor tamaño que Avro/Protobuf (no crítico en desarrollo)

### Paso 4: Autoconfiguración de KafkaTemplate

Spring Boot autoconfigura `KafkaTemplate` siguiendo este flujo:

1. Detecta `spring-kafka` en classpath
2. Lee configuración de `spring.kafka.producer.*`
3. Crea `ProducerFactory` con la configuración
4. Crea `KafkaTemplate` usando `ProducerFactory`
5. Registra como bean en ApplicationContext

**Equivalente manual (NO necesario con Spring Boot)**:

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

**Con Spring Boot, esto NO es necesario** - la autoconfiguración lo hace por ti.

**Nota sobre tipos genéricos**:

- La configuración usa `KafkaTemplate<String, Object>` (genérico) porque Spring Boot no sabe qué tipo de eventos enviarás
- En tu código, usa tipos específicos: `KafkaTemplate<String, ProductCreatedEvent>` para type-safety
- Ambos enfoques son válidos, pero los tipos específicos previenen errores en tiempo de compilación

### Paso 5: Verificar logs de inicio

Al ejecutar `mvn spring-boot:run`, buscar en logs:

```
INFO  o.s.k.c.DefaultKafkaProducerFactory   : Creating new KafkaProducer
INFO  o.a.k.clients.producer.ProducerConfig : ProducerConfig values:
    bootstrap.servers = [localhost:9092]
    key.serializer = class org.apache.kafka.common.serialization.StringSerializer
    value.serializer = class org.springframework.kafka.support.serializer.JsonSerializer
```

**Si NO ves estos logs**:

1. Verificar que la dependencia está en pom.xml
2. Ejecutar `mvn clean install` para descargar dependencias
3. Verificar configuración en application.yml (indentación YAML correcta)

### Paso 6: ¿Cómo saber si KafkaTemplate está disponible?

**Opción 1: Crear endpoint de prueba**

```java
@RestController
@RequestMapping("/api/kafka-test")
public class KafkaTestController {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public KafkaTestController(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @GetMapping("/test")
    public String test() {
        return "KafkaTemplate inyectado correctamente: " +
               kafkaTemplate.getClass().getName();
    }
}
```

**Probar**:

```bash
curl http://localhost:8080/api/kafka-test/test
```

Respuesta esperada:

```
KafkaTemplate inyectado correctamente: org.springframework.kafka.core.KafkaTemplate
```

**Opción 2: Verificar en logs de Spring Boot**

```
INFO o.s.b.a.k.KafkaAutoConfiguration : Creating Kafka template
```

---

## Conceptos aprendidos

- **spring-kafka** es el proyecto oficial de Spring para integrar Kafka con Spring Boot
- **KafkaTemplate** es la clase principal para enviar mensajes, análoga a JdbcTemplate o RestTemplate
- **Autoconfiguración** de Spring Boot elimina la necesidad de clases @Configuration manuales
- **JsonSerializer** permite enviar objetos Java que se convierten automáticamente a JSON
- **bootstrap-servers** es la dirección del broker de Kafka donde el producer se conecta
- **key-serializer** convierte la clave del mensaje a bytes (usamos StringSerializer)
- **value-serializer** convierte el evento a bytes (usamos JsonSerializer para objetos Java)
- La **versión de spring-kafka** se hereda de spring-boot-starter-parent, garantizando compatibilidad
- Spring Boot registra KafkaTemplate como bean automáticamente al detectar la dependencia
- No se requiere código Java adicional para configurar el producer, solo application.yml

---

## Troubleshooting

### Problema 1: Error "Failed to construct kafka producer"

**Síntoma**:

```
ERROR o.s.k.c.DefaultKafkaProducerFactory : Failed to construct kafka producer
```

**Causa**: Kafka no está corriendo o bootstrap-servers incorrecto.

**Solución**:

```bash
# Verificar que Kafka está corriendo
docker compose ps

# Debería mostrar kafka "Up"
# Si no está corriendo, navegar al directorio de infraestructura de Kafka (de Clase 4)
# y levantar los servicios:
cd ~/workspace/kafka-infrastructure
docker compose up -d
```

### Problema 2: Dependencia no se descarga

**Síntoma**:

```
Could not find artifact org.springframework.kafka:spring-kafka
```

**Causa**: Maven no encuentra la dependencia o repositorio no accesible.

**Solución**:

```bash
# Limpiar cache de Maven y recargar
mvn clean
mvn dependency:purge-local-repository
mvn clean install
```

### Problema 3: Error de indentación en YAML

**Síntoma**:

```
Failed to bind properties under 'spring.kafka.producer'
```

**Causa**: Indentación incorrecta en application.yml (YAML es sensible a espacios).

**Solución**:

```yaml
# ❌ INCORRECTO - indentación inconsistente
spring:
  kafka:
  bootstrap-servers: localhost:9092

# ✅ CORRECTO - indentación consistente (2 espacios)
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
```

**Tip**: Usar editor con soporte YAML (IntelliJ IDEA, VSCode con extensión YAML).

### Problema 4: KafkaTemplate no se inyecta (NullPointerException)

**Síntoma**:

```java
@Autowired
private KafkaTemplate<String, Object> kafkaTemplate; // null
```

**Causa**: Clase no es un bean de Spring (falta @Component, @Service, @RestController).

**Solución**:

```java
// ❌ INCORRECTO - clase común, no es bean
public class MyClass {
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public MyClass(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate; // null - no es bean de Spring
    }
}

// ✅ CORRECTO - clase es bean de Spring
@Service
public class MyService {
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public MyService(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate; // inyectado correctamente
    }
}
```

### Problema 5: Application no inicia - puerto 8080 ocupado

**Síntoma**:

```
Web server failed to start. Port 8080 was already in use.
```

**Causa**: Otra instancia de product-service o aplicación usando puerto 8080.

**Solución**:

```bash
# Opción 1: Matar proceso en puerto 8080
lsof -ti:8080 | xargs kill -9

# Opción 2: Usar otro puerto
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

---

## Desafío adicional

### Desafío 1: Configurar múltiples brokers

Simula un entorno con múltiples brokers:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092,localhost:9093,localhost:9094
```

Documenta qué pasa si uno de los brokers está caído (pista: el producer sigue funcionando con los brokers disponibles).

### Desafío 2: Cambiar serialización a String

Modifica la configuración para usar `StringSerializer` en lugar de `JsonSerializer`:

```yaml
spring:
  kafka:
    producer:
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```

Intenta enviar un mensaje y documenta la diferencia. ¿Qué necesitas hacer manualmente?

### Desafío 3: Endpoint de health check de Kafka

Crea un endpoint que verifique si Kafka está accesible:

```java
@RestController
@RequestMapping("/api/health")
public class KafkaHealthController {

    private final KafkaTemplate<String, String> kafkaTemplate;

    @GetMapping("/kafka")
    public ResponseEntity<String> checkKafka() {
        try {
            // Intenta obtener metadata de Kafka
            kafkaTemplate.partitionsFor("test-topic");
            return ResponseEntity.ok("Kafka está accesible");
        } catch (Exception e) {
            return ResponseEntity.status(503)
                .body("Kafka NO accesible: " + e.getMessage());
        }
    }
}
```

---

## Recursos adicionales

- [Spring Kafka Documentation](https://docs.spring.io/spring-kafka/reference/html/)
- [Spring Boot Kafka Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.integration.spring.kafka)
- [KafkaTemplate API](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/core/KafkaTemplate.html)
- [JsonSerializer Documentation](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/support/serializer/JsonSerializer.html)
- [Kafka Producer Configuration](https://kafka.apache.org/documentation/#producerconfigs)
