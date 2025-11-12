# Lab 02: Variables de Entorno

---

## 1. Objetivo

Externalizar toda la configuración de los microservicios usando variables de entorno, siguiendo los principios de 12-Factor App para separar configuración del código y facilitar despliegues en diferentes entornos.

---

## 2. Comandos a ejecutar

```bash
# Modificar application.yml con variables de entorno
cd ~/workspace/product-service/src/main/resources

# Editar application.yml (reemplazar valores hardcodeados)
cat > application.yml << 'EOF'
spring:
  application:
    name: product-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: ${JPA_SHOW_SQL:false}

server:
  port: ${SERVER_PORT:8080}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
EOF

# Crear archivo .env de ejemplo (NO committear)
cat > .env.example << 'EOF'
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=postgres

# Server Configuration
SERVER_PORT=8080

# JPA Configuration
JPA_SHOW_SQL=true
EOF

# Ejecutar con variables de entorno (Linux/Mac)
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=ecommerce
export DB_USER=postgres
export DB_PASSWORD=postgres
export SERVER_PORT=8080
export JPA_SHOW_SQL=true

mvn spring-boot:run

# Ejecutar con variables inline
DB_HOST=localhost DB_PORT=5432 DB_NAME=ecommerce mvn spring-boot:run

# Verificar que usa las variables
curl http://localhost:8080/actuator/env | grep DB_HOST
```

**Salida esperada**:

```
Started ProductServiceApplication in 3.5 seconds (JVM running for 4.1)
```

---

## 3. Desglose del comando

| Elemento | Descripción |
|----------|-------------|
| `${DB_HOST:localhost}` | Variable de entorno DB_HOST con fallback a localhost |
| `:localhost` | Valor por defecto si la variable no está definida |
| `export DB_HOST=valor` | Define variable de entorno en Linux/Mac |
| `set DB_HOST=valor` | Define variable de entorno en Windows |
| `.env.example` | Plantilla de variables (commiteable, sin valores reales) |
| `.env` | Archivo con valores reales (NO committear, en .gitignore) |

---

## 4. Explicación detallada

### 4.1 ¿Qué son las Variables de Entorno?

Las **variables de entorno** son valores que existen fuera del código de la aplicación y se proporcionan en tiempo de ejecución. Son esenciales para:

- **Separar configuración del código**: No hardcodear credenciales
- **Diferentes entornos**: Dev, test, staging, producción
- **Seguridad**: No committear passwords, secrets
- **Portabilidad**: Misma imagen de Docker, diferentes configuraciones

### 4.2 Principios de 12-Factor App

[12-Factor App](https://12factor.net/) es una metodología para aplicaciones SaaS. El **Factor III** establece:

> **Store config in the environment**
>
> Una aplicación twelve-factor almacena la configuración en variables de entorno.

**Beneficios**:
- Despliegue en cualquier entorno sin cambiar código
- No riesgo de committear secretos al repositorio
- Fácil cambio de configuración sin rebuild

### 4.3 Sintaxis en Spring Boot

```yaml
propiedad: ${VARIABLE_ENV:valor_por_defecto}
```

**Ejemplos**:

```yaml
# Sin valor por defecto (error si no existe)
spring:
  datasource:
    password: ${DB_PASSWORD}

# Con valor por defecto (fallback)
spring:
  datasource:
    password: ${DB_PASSWORD:postgres}

# Múltiples variables en una URL
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
```

### 4.4 Configuración Completa con Variables

#### application.yml (product-service)

```yaml
spring:
  application:
    name: product-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:ecommerce}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: ${JPA_DDL_AUTO:update}
    show-sql: ${JPA_SHOW_SQL:false}

server:
  port: ${SERVER_PORT:8080}

management:
  endpoints:
    web:
      exposure:
        include: ${ACTUATOR_ENDPOINTS:health,info,metrics}
```

#### application.yml (order-service, inventory-service, analytics-service)

Agregar adicionalmente:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: ${KAFKA_CONSUMER_GROUP:order-service}
      auto-offset-reset: ${KAFKA_AUTO_OFFSET_RESET:earliest}
```

### 4.5 Formas de Definir Variables de Entorno

#### Linux/Mac (Terminal)

```bash
# Exportar variable (permanece en sesión)
export DB_HOST=localhost
export DB_PORT=5432

# Variable inline (solo para ese comando)
DB_HOST=localhost mvn spring-boot:run

# Desde archivo .env (usando source)
set -a
source .env
set +a
mvn spring-boot:run
```

#### Windows (CMD)

```cmd
set DB_HOST=localhost
set DB_PORT=5432
mvn spring-boot:run
```

#### Windows (PowerShell)

```powershell
$env:DB_HOST="localhost"
$env:DB_PORT="5432"
mvn spring-boot:run
```

#### Docker Compose

```yaml
services:
  product-service:
    image: product-service:latest
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=ecommerce
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    ports:
      - "8080:8080"
```

#### Kubernetes (ConfigMap + Secret)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-service-config
data:
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: ecommerce
---
apiVersion: v1
kind: Secret
metadata:
  name: product-service-secrets
type: Opaque
stringData:
  DB_USER: postgres
  DB_PASSWORD: postgres
```

### 4.6 Archivo .env (Desarrollo Local)

**IMPORTANTE**: NO committear `.env` a Git. Solo committear `.env.example`.

#### .gitignore

```
.env
*.env
```

#### .env.example (commiteable)

```bash
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=postgres

# Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Server Configuration
SERVER_PORT=8080

# JPA Configuration
JPA_DDL_AUTO=update
JPA_SHOW_SQL=true
```

#### .env (NO commiteable, valores reales)

```bash
DB_HOST=prod-db.example.com
DB_PORT=5432
DB_NAME=ecommerce_prod
DB_USER=prod_user
DB_PASSWORD=SuperSecurePassword123!
KAFKA_BOOTSTRAP_SERVERS=kafka-prod.example.com:9092
```

### 4.7 Verificar Variables Cargadas

```bash
# Ver todas las propiedades de la aplicación
curl http://localhost:8080/actuator/env

# Filtrar variables específicas
curl http://localhost:8080/actuator/env | grep DB_HOST
```

**PRECAUCIÓN**: `/actuator/env` puede exponer información sensible. Proteger en producción.

---

## 5. Conceptos aprendidos

- **12-Factor App**: Metodología para aplicaciones cloud-native
- **Variables de entorno**: Configuración externa al código
- **Externalización de configuración**: Separar config del código
- **Fallback values**: Valores por defecto con `${VAR:default}`
- **Secrets management**: No committear passwords al repositorio
- **Portabilidad**: Misma aplicación, diferentes configuraciones
- **.env files**: Archivos de configuración local (no commitear)

---

## 6. Troubleshooting

### Problema 1: Variable no se resuelve

**Error**: `Could not resolve placeholder 'DB_HOST' in value "jdbc:postgresql://${DB_HOST}:5432"`

**Causa**: Variable de entorno no definida y no hay fallback.

**Solución 1**: Agregar valor por defecto:

```yaml
url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}
```

**Solución 2**: Exportar variable:

```bash
export DB_HOST=localhost
mvn spring-boot:run
```

### Problema 2: Variables no persisten

**Síntoma**: Variables funcionan en una terminal pero no en otra.

**Causa**: Variables exportadas solo existen en la sesión actual.

**Solución**: Usar archivo `.env` y source:

```bash
set -a
source .env
set +a
```

O agregar a `~/.bashrc` (Linux) o `~/.zshrc` (Mac):

```bash
export DB_HOST=localhost
export DB_PORT=5432
```

### Problema 3: Docker Compose no ve las variables

**Síntoma**: Aplicación en Docker no puede conectar a PostgreSQL.

**Causa**: Variables de host no se pasan automáticamente a contenedores.

**Solución**: Definir en `docker-compose.yml`:

```yaml
services:
  product-service:
    environment:
      - DB_HOST=postgres  # Nombre del servicio, no localhost
      - DB_PORT=5432
```

### Problema 4: Valores sensibles expuestos en /actuator/env

**Síntoma**: Passwords visibles en endpoint de Actuator.

**Solución**: Spring Boot ofusca automáticamente propiedades con:
- `password`
- `secret`
- `token`
- `key`

Verificar que uses estos nombres. O deshabilitar endpoint en producción:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # NO incluir 'env'
```

---

## 7. Desafío adicional

1. **Crea un script de inicio** que valide variables requeridas:

```bash
#!/bin/bash
# start-product-service.sh

required_vars=("DB_HOST" "DB_PORT" "DB_NAME" "DB_USER" "DB_PASSWORD")

for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "Error: Variable $var no está definida"
        exit 1
    fi
done

echo "Todas las variables están configuradas. Iniciando servicio..."
mvn spring-boot:run
```

Ejecutar:
```bash
chmod +x start-product-service.sh
./start-product-service.sh
```

2. **Implementa configuración por perfil** con variables de entorno:

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

```bash
export SPRING_PROFILES_ACTIVE=prod
mvn spring-boot:run
```

3. **Integra con HashiCorp Vault** (avanzado):

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    vault:
      uri: http://localhost:8200
      token: ${VAULT_TOKEN}
      database:
        enabled: true
```

---

## 8. Recursos adicionales

- [12-Factor App - Config](https://12factor.net/config)
- [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
- [Docker Compose - Environment Variables](https://docs.docker.com/compose/environment-variables/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

---

**Siguiente**: [Lab 03: JWT Security](../03-jwt-security/README.md)
