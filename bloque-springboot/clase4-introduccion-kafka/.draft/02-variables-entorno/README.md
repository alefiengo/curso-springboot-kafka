# Lab 02: Variables de Entorno

> **Nota**: Este laboratorio se verá en la **Clase 8** como parte de la preparación para producción. Si estás siguiendo el curso en orden, continúa con el [Lab 00: Conceptos de Kafka](../labs/00-conceptos-kafka/).

---

## Objetivo

Externalizar la configuración de `product-service` usando variables de entorno y valores por defecto (fallback), siguiendo el principio **12-Factor App** para preparar la aplicación para despliegue en contenedores y entornos cloud.

---

## Comandos a ejecutar

```bash
# 1. Navegar al directorio del proyecto
cd product-service

# 2. Modificar application.yml para usar variables de entorno con fallbacks
# Abrir src/main/resources/application.yml y reemplazar las líneas hardcodeadas:
#
# Reemplazar:
#   url: jdbc:postgresql://localhost:5432/product_db
#   username: postgres
#   password: postgres
#   port: 8080
#
# Por:
#   url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:product_db}
#   username: ${DB_USER:postgres}
#   password: ${DB_PASSWORD:postgres}
#   port: ${SERVER_PORT:8080}

# 3. Ejecutar sin variables de entorno (usa valores por defecto)
mvn spring-boot:run

# 4. Verificar que usa localhost y puerto 5432
# Observar logs: "HikkaRI pool starting with URL: jdbc:postgresql://localhost:5432/product_db"

# 5. Detener aplicación (Ctrl+C)

# 6. Definir variables de entorno (Linux/Mac)
export DB_HOST=postgres
export DB_PORT=5432
export DB_NAME=product_db
export DB_USER=postgres
export DB_PASSWORD=my_secure_password
export SERVER_PORT=8090

# 7. Ejecutar con variables de entorno
mvn spring-boot:run

# 8. Verificar que usa las variables
# Observar logs: "HikkaRI pool starting with URL: jdbc:postgresql://postgres:5432/product_db"

# 9. En otra terminal, probar endpoint en nuevo puerto
curl http://localhost:8090/api/products

# 10. Limpiar variables de entorno
unset DB_HOST DB_PORT DB_NAME DB_USER DB_PASSWORD SERVER_PORT
```

**Para Windows (PowerShell)**:

```powershell
# Definir variables
$env:DB_HOST="postgres"
$env:DB_PORT="5432"
$env:DB_NAME="product_db"
$env:DB_USER="postgres"
$env:DB_PASSWORD="my_secure_password"
$env:SERVER_PORT="8090"

# Ejecutar aplicación
mvn spring-boot:run

# Limpiar variables
Remove-Item Env:\DB_HOST
Remove-Item Env:\DB_PORT
Remove-Item Env:\DB_NAME
Remove-Item Env:\DB_USER
Remove-Item Env:\DB_PASSWORD
Remove-Item Env:\SERVER_PORT
```

**Salida esperada con valores por defecto**:

```
2025-11-05T11:00:15.234  INFO 12345 --- [main] com.zaxxer.hikari.HikariDataSource
    : HikariPool-1 - Starting...
2025-11-05T11:00:15.456  INFO 12345 --- [main] com.zaxxer.hikari.HikariDataSource
    : HikariPool-1 - Start completed.
    URL: jdbc:postgresql://localhost:5432/product_db

2025-11-05T11:00:16.789  INFO 12345 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer
    : Tomcat started on port(s): 8080 (http)
```

**Salida esperada con variables de entorno**:

```
2025-11-05T11:05:20.123  INFO 12346 --- [main] com.zaxxer.hikari.HikariDataSource
    : HikariPool-1 - Starting...
2025-11-05T11:05:20.345  INFO 12346 --- [main] com.zaxxer.hikari.HikariDataSource
    : HikariPool-1 - Start completed.
    URL: jdbc:postgresql://postgres:5432/product_db

2025-11-05T11:05:21.678  INFO 12346 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer
    : Tomcat started on port(s): 8090 (http)
```

---

## Desglose del comando

### Sintaxis de variable de entorno en Spring Boot

**Formato**: `${VARIABLE:valor_por_defecto}`

| Componente | Descripción |
|------------|-------------|
| `${}` | Sintaxis de Spring para placeholder de propiedad |
| `VARIABLE` | Nombre de la variable de entorno (mayúsculas por convención) |
| `:` | Separador entre variable y valor por defecto |
| `valor_por_defecto` | Valor usado si la variable de entorno NO existe |

**Ejemplos**:

```yaml
# Si DB_HOST existe, usa su valor; si no, usa "localhost"
url: jdbc:postgresql://${DB_HOST:localhost}:5432/product_db

# Si SERVER_PORT existe, usa su valor; si no, usa 8080
server:
  port: ${SERVER_PORT:8080}

# Si SPRING_PROFILES_ACTIVE existe, usa su valor; si no, usa "dev"
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
```

---

## Explicación detallada

### Paso 1: Comprender el principio 12-Factor App

El principio **III. Config** de [12-Factor App](https://12factor.net/config) establece:

> "Almacenar configuración en el entorno, no en el código"

**Ventajas**:

- **Seguridad**: Credenciales nunca van al control de versiones (Git)
- **Portabilidad**: Mismo binario (JAR) funciona en dev, staging, producción
- **Flexibilidad**: Cambiar configuración sin recompilar
- **Conformidad cloud**: Compatibilidad con Docker, Kubernetes, Heroku, AWS

**Qué externalizar**:

- Credenciales de base de datos
- URLs de servicios externos
- Claves API
- Configuración de infraestructura (hosts, puertos)
- Flags de features

**Qué NO externalizar**:

- Lógica de negocio
- Constantes del dominio
- Configuración estructural de la aplicación

### Paso 2: Modificar application.yml

Reemplaza las configuraciones hardcodeadas con variables de entorno y fallbacks.

**Antes (hardcoded)**:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/product_db
    username: postgres
    password: postgres

server:
  port: 8080
```

**Después (externalizado)**:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:product_db}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}

server:
  port: ${SERVER_PORT:8080}
```

**Desglose de la URL de base de datos**:

```
jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:product_db}
                 └─────┬──────┘  └─────┬──────┘  └──────┬───────┘
                  Host con        Puerto con      Base de datos
                  fallback        fallback        con fallback
```

Si ejecutas la aplicación sin definir variables:

- `DB_HOST` → `localhost`
- `DB_PORT` → `5432`
- `DB_NAME` → `product_db`
- `DB_USER` → `postgres`
- `DB_PASSWORD` → `postgres`

URL resultante: `jdbc:postgresql://localhost:5432/product_db`

Si defines las variables:

```bash
export DB_HOST=prod-db-server
export DB_PORT=5433
export DB_NAME=product_db_prod
```

URL resultante: `jdbc:postgresql://prod-db-server:5433/product_db_prod`

### Paso 3: Configuración completa externalizada

Archivo `application.yml` completo con variables de entorno:

```yaml
spring:
  application:
    name: product-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:product_db}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: ${DDL_AUTO:update}
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

server:
  port: ${SERVER_PORT:8080}

logging:
  level:
    dev.alefiengo: ${LOG_LEVEL:INFO}
```

**Variables soportadas**:

| Variable | Valor por defecto | Descripción |
|----------|-------------------|-------------|
| `DB_HOST` | `localhost` | Host del servidor PostgreSQL |
| `DB_PORT` | `5432` | Puerto de PostgreSQL |
| `DB_NAME` | `product_db` | Nombre de la base de datos |
| `DB_USER` | `postgres` | Usuario de la base de datos |
| `DB_PASSWORD` | `postgres` | Contraseña de la base de datos |
| `SERVER_PORT` | `8080` | Puerto HTTP de la aplicación |
| `DDL_AUTO` | `update` | Estrategia de Hibernate DDL |
| `LOG_LEVEL` | `INFO` | Nivel de log de la aplicación |

### Paso 4: Definir variables de entorno en desarrollo local

#### Linux / macOS

```bash
# Definir variables para la sesión actual
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=product_db_dev
export DB_USER=postgres
export DB_PASSWORD=postgres

# Ejecutar aplicación
mvn spring-boot:run

# Variables persisten hasta cerrar la terminal
```

#### Windows (PowerShell)

```powershell
$env:DB_HOST="localhost"
$env:DB_PORT="5432"
$env:DB_NAME="product_db_dev"
$env:DB_USER="postgres"
$env:DB_PASSWORD="postgres"

mvn spring-boot:run
```

#### Windows (CMD)

```cmd
set DB_HOST=localhost
set DB_PORT=5432
set DB_NAME=product_db_dev
set DB_USER=postgres
set DB_PASSWORD=postgres

mvn spring-boot:run
```

### Paso 5: Usar archivo .env para desarrollo local

Para evitar definir variables manualmente cada vez, puedes crear un archivo `.env`:

**Crear `.env` en la raíz del proyecto**:

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=product_db_dev
DB_USER=postgres
DB_PASSWORD=postgres
SERVER_PORT=8080
LOG_LEVEL=DEBUG
```

**IMPORTANTE**: Agregar `.env` al `.gitignore`:

```bash
echo ".env" >> .gitignore
git add .gitignore
git commit -m "chore: ignore .env file"
```

**Cargar variables desde .env** (Linux/Mac con herramienta `direnv` o manualmente):

```bash
# Opción 1: Cargar manualmente con source
export $(cat .env | xargs)
mvn spring-boot:run

# Opción 2: Usar direnv (instalar primero)
direnv allow
mvn spring-boot:run
```

**Alternativa con Maven plugin**:

Agregar `dotenv-maven-plugin` a `pom.xml`:

```xml
<plugin>
    <groupId>me.yzhao</groupId>
    <artifactId>dotenv-maven-plugin</artifactId>
    <version>1.0.0</version>
    <executions>
        <execution>
            <goals>
                <goal>load</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Ahora Maven cargará `.env` automáticamente al ejecutar `mvn spring-boot:run`.

### Paso 6: Configurar variables en Docker Compose

**docker-compose.yml**:

```yaml
services:
  postgres:
    image: postgres:15
    container_name: postgres
    environment:
      POSTGRES_DB: product_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  product-service:
    build: .
    container_name: product-service
    depends_on:
      - postgres
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: product_db
      DB_USER: postgres
      DB_PASSWORD: postgres
      SERVER_PORT: 8080
      SPRING_PROFILES_ACTIVE: prod
    ports:
      - "8080:8080"

volumes:
  postgres-data:
```

**Explicación**:

- `DB_HOST: postgres` → Nombre del servicio de PostgreSQL en Docker Compose
- Spring Boot lee estas variables y las usa en lugar de los fallbacks
- No necesitas modificar `application.yml` entre entornos

### Paso 7: Configurar variables en Kubernetes

**deployment.yaml**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
        - name: product-service
          image: product-service:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "product_db"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
```

**Secret para credenciales**:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  username: cG9zdGdyZXM=   # base64 de "postgres"
  password: bXlfc2VjdXJl   # base64 de "my_secure"
```

### Paso 8: Verificar variables activas con Actuator

Si tienes Actuator habilitado (Lab 01), puedes ver las variables activas:

```bash
curl http://localhost:8080/actuator/env
```

**ADVERTENCIA**: NO exponer `/actuator/env` en producción sin autenticación, ya que puede revelar credenciales.

**Respuesta (extracto)**:

```json
{
  "propertySources": [
    {
      "name": "systemEnvironment",
      "properties": {
        "DB_HOST": { "value": "postgres" },
        "DB_PORT": { "value": "5432" },
        "DB_NAME": { "value": "product_db" },
        "DB_USER": { "value": "postgres" }
      }
    }
  ]
}
```

Notarás que `DB_PASSWORD` aparece como `******` por seguridad (Spring Boot oculta propiedades sensibles).

---

## Conceptos aprendidos

- **12-Factor App**: Principio de externalización de configuración
- **Variables de entorno**: Configuración fuera del código fuente
- **Fallback values**: Valores por defecto cuando la variable no existe
- **Sintaxis `${VAR:default}`**: Placeholder de Spring Boot
- **Archivo .env**: Gestión de variables en desarrollo local
- **Docker Compose environment**: Variables en servicios containerizados
- **Kubernetes Secrets**: Gestión segura de credenciales en producción
- **Portabilidad**: Mismo JAR funciona en todos los entornos
- **Seguridad**: Credenciales nunca en control de versiones

---

## Troubleshooting

### Problema 1: Variable de entorno no reconocida

**Síntoma**:

```
2025-11-05T11:10:23.456  WARN 12347 --- [main] o.h.engine.jdbc.env.internal.JdbcEnvironmentInitiator
    : HHH000342: Could not obtain connection to query metadata
```

Aplicación usa valor por defecto en lugar de variable de entorno.

**Causa**: Variable definida en una terminal, pero ejecutas la aplicación en otra.

**Solución**:

Verificar que la variable está definida en la terminal actual:

```bash
# Linux/Mac
echo $DB_HOST

# Windows PowerShell
echo $env:DB_HOST

# Windows CMD
echo %DB_HOST%
```

Si no aparece nada, definir nuevamente la variable en esa terminal.

### Problema 2: Variables no persisten entre sesiones

**Síntoma**: Cada vez que abres una terminal, debes definir las variables nuevamente.

**Causa**: Las variables de entorno de la sesión no se guardan automáticamente.

**Solución**:

**Linux/Mac** (agregar a `~/.bashrc` o `~/.zshrc`):

```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=product_db_dev
export DB_USER=postgres
export DB_PASSWORD=postgres
```

Recargar configuración:

```bash
source ~/.bashrc  # o ~/.zshrc
```

**Windows** (variables permanentes):

```powershell
[System.Environment]::SetEnvironmentVariable('DB_HOST', 'localhost', 'User')
[System.Environment]::SetEnvironmentVariable('DB_PORT', '5432', 'User')
```

### Problema 3: Credenciales commiteadas a Git accidentalmente

**Síntoma**: Pusheaste `application.yml` con contraseñas hardcodeadas.

**Solución**:

1. **Cambiar inmediatamente las credenciales reales en producción**
2. **Remover contraseñas del historial de Git**:

```bash
# Instalar git-filter-repo (no usar git filter-branch, está obsoleto)
pip install git-filter-repo

# Remover archivo del historial
git filter-repo --path application.yml --invert-paths

# Forzar push (CUIDADO: reescribe historia)
git push origin --force --all
```

3. **Usar variables de entorno en adelante**:

```yaml
# application.yml - NUNCA hardcodear credenciales
spring:
  datasource:
    password: ${DB_PASSWORD:postgres}  # Fallback solo para desarrollo local
```

### Problema 4: Docker Compose no usa variables

**Síntoma**: Al ejecutar `docker compose up`, la aplicación usa valores por defecto en lugar de los definidos en `docker-compose.yml`.

**Causa**: Variables mal indentadas o sintaxis YAML incorrecta.

**Solución**:

Verificar sintaxis (espacios, NO tabs):

```yaml
# ✅ CORRECTO
services:
  product-service:
    environment:
      DB_HOST: postgres
      DB_PORT: 5432

# ❌ INCORRECTO (tabs en lugar de espacios)
services:
	product-service:
		environment:
			DB_HOST: postgres
```

Validar YAML en [https://www.yamllint.com/](https://www.yamllint.com/)

### Problema 5: Fallback incorrecto en producción

**Síntoma**: Aplicación en producción se conecta a base de datos de desarrollo porque usa el fallback `localhost`.

**Causa**: Variables de entorno no definidas en el entorno de producción.

**Solución**:

**Validar al iniciar la aplicación**:

```java
@Component
public class ConfigValidator {

    @Value("${DB_HOST}")
    private String dbHost;

    @PostConstruct
    public void validate() {
        if ("localhost".equals(dbHost) && isProdProfile()) {
            throw new IllegalStateException("DB_HOST no configurado en producción");
        }
    }

    private boolean isProdProfile() {
        return Arrays.asList(environment.getActiveProfiles()).contains("prod");
    }
}
```

Así la aplicación no iniciará si faltan variables críticas en producción.

### Problema 6: Actuator expone credenciales

**Síntoma**: `curl http://localhost:8080/actuator/env` muestra contraseñas en texto claro.

**Causa**: Spring Boot 2.x mostraba todas las propiedades. Spring Boot 3.x oculta propiedades sensibles por defecto.

**Solución**:

1. **Actualizar a Spring Boot 3.x**
2. **NO exponer `/actuator/env` públicamente**:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # NO incluir "env"
```

3. **Agregar custom sanitizer** (Spring Boot 2.x):

```java
@Bean
public SanitizingFunction sanitizer() {
    return data -> {
        if (data.getKey().toLowerCase().contains("password")) {
            return data.withValue("******");
        }
        return data;
    };
}
```

---

## Desafío adicional

### Desafío 1: Crear perfil con variables específicas

Crea un perfil `staging` que:

- Use base de datos `product_db_staging`
- Puerto `8081`
- Nivel de log `INFO`

**application-staging.yml**:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/product_db_staging
server:
  port: 8081
logging:
  level:
    dev.alefiengo: INFO
```

Ejecutar:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=staging
```

### Desafío 2: Validar configuración al inicio

Crea un componente que valide que todas las variables críticas están definidas:

**ConfigurationValidator.java**:

```java
package dev.alefiengo.productservice.config;

import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("prod")
public class ConfigurationValidator {

    @Value("${DB_HOST}")
    private String dbHost;

    @Value("${DB_NAME}")
    private String dbName;

    @Value("${DB_USER}")
    private String dbUser;

    @Value("${DB_PASSWORD}")
    private String dbPassword;

    @PostConstruct
    public void validate() {
        if ("localhost".equals(dbHost)) {
            throw new IllegalStateException(
                "DB_HOST debe estar configurado en producción (no puede ser localhost)"
            );
        }
        if ("postgres".equals(dbPassword)) {
            throw new IllegalStateException(
                "DB_PASSWORD debe ser una contraseña segura en producción"
            );
        }
        System.out.println("✅ Configuración de producción validada correctamente");
    }
}
```

Este componente solo se activa con perfil `prod` y lanza excepción si detecta configuración insegura.

### Desafío 3: Usar Spring Cloud Config Server

Para gestionar configuración de múltiples microservicios centralizadamente, investiga **Spring Cloud Config Server**:

- Almacena configuración en repositorio Git
- Los microservicios consultan config server al iniciar
- Permite cambiar configuración sin rebuild

**Recurso**: [Spring Cloud Config](https://spring.io/projects/spring-cloud-config)

---

## Recursos adicionales

- [12-Factor App - Config](https://12factor.net/config)
- [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)
- [Spring Boot Configuration Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html)
- [Docker Compose Environment Variables](https://docs.docker.com/compose/environment-variables/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [direnv - Environment Variables per Directory](https://direnv.net/)
- [Baeldung - Spring Boot Configuration](https://www.baeldung.com/configuration-properties-in-spring-boot)
