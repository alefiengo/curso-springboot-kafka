# Práctica 1: Hola Mundo con Spring Boot

**Nivel:** Principiante

---

## 1. Objetivo

Crear tu primera aplicación Spring Boot desde cero usando Spring Initializr, implementar un endpoint REST básico y ejecutar la aplicación para verificar que funciona correctamente.

**Al finalizar esta práctica podrás:**
- Generar un proyecto Spring Boot con Spring Initializr
- Entender la estructura básica de un proyecto Maven
- Crear tu primer controlador REST
- Ejecutar una aplicación Spring Boot
- Probar un endpoint REST con el navegador y curl

---

## 2. Comandos a Ejecutar

### Paso 1: Generar el proyecto con Spring Initializr

**Opción A: Usando la web**

1. Abre [https://start.spring.io](https://start.spring.io) en tu navegador
2. Configura:
   - **Project:** Maven
   - **Language:** Java
   - **Spring Boot:** 3.3.x (o la versión estable más reciente)
   - **Group:** `dev.alefiengo`
   - **Artifact:** `hola-mundo`
   - **Name:** `hola-mundo`
   - **Package name:** `dev.alefiengo.holamundo`
   - **Java:** 17
3. Agrega dependencias:
   - Spring Web
   - Spring Boot DevTools
4. Click en "Generate" y descarga el archivo ZIP

**Opción B: Usando curl**

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,devtools \
  -d groupId=dev.alefiengo \
  -d artifactId=hola-mundo \
  -d name=hola-mundo \
  -d packageName=dev.alefiengo.holamundo \
  -d javaVersion=17 \
  -d type=maven-project \
  -o hola-mundo.zip
```

### Paso 2: Descomprimir y navegar

```bash
# Descomprimir
unzip hola-mundo.zip -d hola-mundo
cd hola-mundo

# Ver estructura
tree -L 3 -d
```

**Salida esperada:**
```
hola-mundo/
├── src
│   ├── main
│   │   ├── java
│   │   └── resources
│   └── test
│       └── java
└── target
```

### Paso 3: Crear el controlador

```bash
# Crear directorio para controladores
mkdir -p src/main/java/dev/alefiengo/holamundo/controller

# Crear el archivo del controlador (puedes usar tu IDE o editor)
```

**Archivo:** `src/main/java/dev/alefiengo/holamundo/controller/HelloController.java`

```java
package dev.alefiengo.holamundo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hola Mundo desde Spring Boot!";
    }

    @GetMapping("/")
    public String home() {
        return "Bienvenido a tu primera aplicación Spring Boot";
    }
}
```

### Paso 4: Ejecutar la aplicación

```bash
# Usando Maven
mvn spring-boot:run
```

**Salida esperada:**
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.3.0)

2025-10-20T19:05:30.123  INFO 12345 --- [           main] d.a.holamundo.HolaMundoApplication       : Starting HolaMundoApplication
2025-10-20T19:05:32.456  INFO 12345 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http)
2025-10-20T19:05:32.789  INFO 12345 --- [           main] d.a.holamundo.HolaMundoApplication       : Started HolaMundoApplication in 2.5 seconds
```

### Paso 5: Probar los endpoints

**Usando el navegador:**
- Abre: [http://localhost:8080/](http://localhost:8080/)
- Abre: [http://localhost:8080/hello](http://localhost:8080/hello)

**Usando curl:**

```bash
# Endpoint raíz
curl http://localhost:8080/

# Endpoint hello
curl http://localhost:8080/hello
```

**Respuestas esperadas:**
```
Bienvenido a tu primera aplicación Spring Boot
Hola Mundo desde Spring Boot!
```

---

## 3. Desglose del Comando

### Spring Initializr (curl)

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `-d dependencies=web,devtools` | web, devtools | Spring Web para REST APIs, DevTools para hot reload |
| `-d groupId=dev.alefiengo` | dev.alefiengo | Identificador de grupo (organización/dominio invertido) |
| `-d artifactId=hola-mundo` | hola-mundo | Nombre del proyecto/artefacto |
| `-d name=hola-mundo` | hola-mundo | Nombre de la clase principal |
| `-d packageName=dev.alefiengo.holamundo` | dev.alefiengo.holamundo | Paquete base de la aplicación |
| `-d javaVersion=17` | 17 | Versión de Java a usar |
| `-d type=maven-project` | maven-project | Tipo de proyecto (Maven en lugar de Gradle) |
| `-o hola-mundo.zip` | hola-mundo.zip | Archivo de salida |

### Maven Command

| Comando | Descripción |
|---------|-------------|
| `mvn` | Maven command-line tool |
| `spring-boot:run` | Goal del plugin de Spring Boot para ejecutar la aplicación |

---

## 4. Explicación Detallada

### ¿Qué acabas de hacer?

1. **Generaste un proyecto Spring Boot**
   - Spring Initializr creó la estructura completa del proyecto
   - Incluye todas las dependencias necesarias en `pom.xml`
   - Configuró la clase principal con `@SpringBootApplication`

2. **Creaste un controlador REST**
   - `@RestController` marca la clase como controlador REST
   - Combina `@Controller` + `@ResponseBody`
   - Los métodos retornan datos directamente (no vistas HTML)

3. **Definiste endpoints**
   - `@GetMapping("/hello")` mapea peticiones HTTP GET a `/hello`
   - El método `hello()` se ejecuta cuando accedes a esa ruta
   - Spring automáticamente convierte el String retornado a respuesta HTTP

4. **Ejecutaste la aplicación**
   - Spring Boot arrancó un servidor Tomcat embebido
   - Por defecto escucha en el puerto 8080
   - La aplicación está lista para recibir peticiones

### Flujo de una Petición HTTP

```
[Cliente]
    |
    | HTTP GET http://localhost:8080/hello
    v
[Tomcat Embebido] (puerto 8080)
    |
    | Spring Boot enruta la petición
    v
[DispatcherServlet]
    |
    | Busca método con @GetMapping("/hello")
    v
[HelloController]
    |
    | Ejecuta método hello()
    | return "Hola Mundo desde Spring Boot!"
    v
[Cliente recibe respuesta]
    Hola Mundo desde Spring Boot!
```

### Archivos Clave del Proyecto

**pom.xml** - Configuración de Maven
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**HolaMundoApplication.java** - Clase principal
```java
@SpringBootApplication  // Combina @Configuration + @EnableAutoConfiguration + @ComponentScan
public class HolaMundoApplication {
    public static void main(String[] args) {
        SpringApplication.run(HolaMundoApplication.class, args);
    }
}
```

**application.properties** - Configuración (vacío por ahora)

---

## 5. Conceptos Aprendidos

### Anotaciones

- **@SpringBootApplication**: Marca la clase principal de Spring Boot
  - Habilita auto-configuración
  - Escanea componentes en el paquete actual y sub-paquetes
  - Configura Spring Boot con defaults inteligentes

- **@RestController**: Marca una clase como controlador REST
  - Los métodos retornan datos (JSON, texto, XML)
  - No necesita `@ResponseBody` en cada método

- **@GetMapping**: Mapea peticiones HTTP GET a un método
  - Acepta un path como parámetro
  - Versión simplificada de `@RequestMapping(method = RequestMethod.GET)`

### Conceptos Clave

- **Spring Boot**: Framework que simplifica desarrollo Spring
  - Auto-configuración basada en dependencias
  - Servidor embebido (Tomcat, Jetty, Undertow)
  - Sin XML, configuración mediante anotaciones

- **REST API**: Arquitectura para servicios web
  - Usa métodos HTTP (GET, POST, PUT, DELETE)
  - Recursos identificados por URLs
  - Stateless (sin estado entre peticiones)

- **Maven**: Herramienta de build y gestión de dependencias
  - `pom.xml` define dependencias y plugins
  - Descarga librerías automáticamente
  - Compila, empaqueta y ejecuta la aplicación

- **Embedded Server**: Servidor web integrado en el JAR
  - No necesitas instalar Tomcat por separado
  - La aplicación es auto-contenida
  - Facilita el despliegue

### Estructura de Paquetes

```
dev.alefiengo.holamundo/
├── HolaMundoApplication.java   # Clase principal
└── controller/
    └── HelloController.java    # Controladores REST
```

**Convención**: La clase principal debe estar en el paquete raíz para que el component scan funcione correctamente.

---

## 6. Troubleshooting

### Problema 1: "Port 8080 was already in use"

**Error:**
```
Web server failed to start. Port 8080 was already in use.
```

**Solución 1:** Cambiar el puerto

Crea el archivo `src/main/resources/application.properties`:
```properties
server.port=8081
```

**Solución 2:** Detener el proceso que usa el puerto

```bash
# Linux/macOS
lsof -i :8080
kill -9 <PID>

# Windows
netstat -ano | findstr :8080
taskkill /PID <PID> /F
```

---

### Problema 2: "mvn: command not found"

**Causa:** Maven no está instalado o no está en el PATH.

**Solución:** Usa Maven Wrapper (viene con el proyecto)

```bash
# Linux/macOS
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

Si quieres instalar Maven globalmente: [INSTALL_JAVA_MAVEN.md](../../../INSTALL_JAVA_MAVEN.md)

---

### Problema 3: "404 Not Found" al acceder a /hello

**Causas posibles:**

1. **La aplicación no está corriendo**
   - Verifica que veas "Started HolaMundoApplication" en los logs
   - Verifica que no haya errores en la consola

2. **La URL es incorrecta**
   - Verifica: `http://localhost:8080/hello` (case-sensitive)
   - NO: `http://localhost:8080/Hello`

3. **El controlador no fue escaneado**
   - Verifica que `HelloController.java` esté en `dev.alefiengo.holamundo.controller`
   - El paquete debe ser sub-paquete del paquete principal

---

### Problema 4: Cambios en el código no se reflejan

**Causa:** DevTools no está funcionando o necesitas recompilar.

**Solución:**

1. **Recompila en tu IDE:**
   - IntelliJ: `Ctrl+F9` (Build Project)
   - Eclipse: Guarda el archivo (auto-build)

2. **Reinicia manualmente:**
   - Detén la aplicación (`Ctrl+C`)
   - Ejecuta `mvn spring-boot:run` nuevamente

---

### Problema 5: "Cannot resolve symbol 'RestController'"

**Causa:** Imports faltantes.

**Solución:**

Agrega el import al inicio del archivo:
```java
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.GetMapping;
```

En IntelliJ/Eclipse: `Alt+Enter` sobre el error y selecciona "Import class"

---

## 7. Desafío Adicional

### Desafío 1: Agregar endpoint con tu nombre

Crea un nuevo endpoint que retorne un saludo personalizado:

```java
@GetMapping("/saludo")
public String saludo() {
    return "Hola, mi nombre es [Tu Nombre]";
}
```

Prueba: `curl http://localhost:8080/saludo`

---

### Desafío 2: Agregar endpoint con fecha actual

```java
@GetMapping("/fecha")
public String fecha() {
    return "Fecha actual: " + LocalDateTime.now();
}
```

No olvides el import:
```java
import java.time.LocalDateTime;
```

Prueba: `curl http://localhost:8080/fecha`

---

### Desafío 3: Agregar información de la aplicación

```java
@GetMapping("/info")
public String info() {
    return "Aplicación: Hola Mundo | Versión: 1.0 | Autor: [Tu Nombre]";
}
```

Prueba: `curl http://localhost:8080/info`

---

## 8. Recursos Adicionales

### Documentación Oficial

- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Web MVC](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [Spring Initializr](https://start.spring.io/)

### Tutoriales

- [Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)
- [Spring Boot Guides](https://spring.io/guides)

### Herramientas

- [Maven Documentation](https://maven.apache.org/guides/)
- [IntelliJ IDEA Spring Boot Plugin](https://www.jetbrains.com/help/idea/spring-boot.html)

### Próximos Pasos

- **Práctica 2:** REST APIs con parámetros dinámicos
- **Práctica 3:** CRUD completo con ArrayList
- **Cheatsheet:** [Referencia rápida de comandos](../../cheatsheet.md)
- **FAQ:** [Problemas comunes y soluciones](../../FAQ.md)

---

## Código de Solución Completo

El código completo de esta práctica está disponible en: `solucion/`

Estructura:
```
solucion/
├── pom.xml
└── src/
    └── main/
        ├── java/dev/alefiengo/holamundo/
        │   ├── HolaMundoApplication.java
        │   └── controller/
        │       └── HelloController.java
        └── resources/
            └── application.properties
```

---

**¡Felicidades!** Has creado tu primera aplicación Spring Boot.

[← Volver a Clase 1](../../README.md) | [Práctica 2: Parámetros →](../02-parametros/)
