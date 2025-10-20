# FAQ - Preguntas Frecuentes: Clase 1

Respuestas a los problemas más comunes encontrados en la Clase 1.

---

## Problemas de Instalación

### ❓ "java: command not found"

**Problema:** Java no está instalado o no está en el PATH.

**Solución:**

1. Verificar instalación:
```bash
java -version
```

2. Si no está instalado:
   - **macOS:** `brew install openjdk@17`
   - **Linux (Ubuntu/Debian):** `sudo apt install openjdk-17-jdk`
   - **Windows:** Descargar de [adoptium.net](https://adoptium.net/)

3. Configurar JAVA_HOME:
```bash
# Linux/macOS (~/.bashrc o ~/.zshrc)
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export PATH=$JAVA_HOME/bin:$PATH

# Windows (Variables de entorno del sistema)
JAVA_HOME=C:\Program Files\Java\jdk-17
Path=%JAVA_HOME%\bin;%Path%
```

---

### ❓ "mvn: command not found"

**Problema:** Maven no está instalado.

**Solución:**

**Opción 1:** Usar Maven Wrapper (viene con proyectos de Spring Initializr)
```bash
./mvnw spring-boot:run   # Linux/macOS
mvnw.cmd spring-boot:run # Windows
```

**Opción 2:** Instalar Maven
- **macOS:** `brew install maven`
- **Linux:** `sudo apt install maven`
- **Windows:** Descargar de [maven.apache.org](https://maven.apache.org/)

---

## Problemas al Ejecutar

### ❓ "Port 8080 was already in use"

**Problema:** El puerto 8080 ya está ocupado por otra aplicación.

**Solución 1:** Cambiar el puerto en `application.yml`:
```yaml
server:
  port: 8081  # O cualquier otro puerto libre
```

**Solución 2:** Encontrar y detener el proceso que usa el puerto:
```bash
# Linux/macOS
lsof -i :8080
kill -9 <PID>

# Windows
netstat -ano | findstr :8080
taskkill /PID <PID> /F
```

---

### ❓ "Failed to execute goal org.springframework.boot:spring-boot-maven-plugin"

**Problema:** Maven no puede ejecutar el plugin de Spring Boot.

**Solución:**

1. Limpia el proyecto:
```bash
mvn clean
```

2. Vuelve a intentar:
```bash
mvn spring-boot:run
```

3. Si persiste, borra la carpeta `.m2`:
```bash
rm -rf ~/.m2/repository
mvn clean install
```

---

### ❓ "Application run failed - ClassNotFoundException"

**Problema:** Clase principal no encontrada.

**Solución:**

Verifica que tu clase principal tenga `@SpringBootApplication`:
```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

---

## Problemas de Código

### ❓ "Required a bean of type 'TaskService' that could not be found"

**Problema:** Spring no puede inyectar el servicio.

**Solución:**

Asegúrate que tu servicio tenga `@Service`:
```java
@Service  // ← No olvides esta anotación
public class TaskService {
    // ...
}
```

---

### ❓ "Whitelabel Error Page - 404 Not Found"

**Problema:** El endpoint no existe o la ruta es incorrecta.

**Checklist:**
1. ¿El método tiene `@GetMapping` o similar?
2. ¿La clase tiene `@RestController`?
3. ¿La ruta es correcta? (case-sensitive)
4. ¿La aplicación está corriendo?
5. ¿Estás probando en el puerto correcto?

**Ejemplo correcto:**
```java
@RestController
public class TaskController {

    @GetMapping("/tasks")  // ← No olvides la anotación
    public List<Task> getTasks() {
        return List.of();
    }
}
```

---

### ❓ "Cannot resolve symbol 'RestController'"

**Problema:** Imports faltantes.

**Solución:**

Agrega los imports necesarios:
```java
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.PathVariable;
```

**En IntelliJ/Eclipse:** `Alt + Enter` sobre el error y selecciona "Import class"

---

## 📦 Problemas con Maven

### ❓ "Could not resolve dependencies"

**Problema:** Maven no puede descargar dependencias.

**Solución:**

1. Verifica conexión a internet

2. Limpia y reconstruye:
```bash
mvn clean install -U
```

3. Si usas proxy, configura en `~/.m2/settings.xml`:
```xml
<settings>
  <proxies>
    <proxy>
      <host>proxy.company.com</host>
      <port>8080</port>
    </proxy>
  </proxies>
</settings>
```

---

### ❓ "Build Success but application doesn't start"

**Problema:** Compilación correcta pero no ejecuta.

**Solución:**

Usa el plugin correcto:
```bash
# CORRECTO
mvn spring-boot:run

# INCORRECTO
mvn run
mvn start
```

---

## 🌐 Problemas con Postman / curl

### ❓ "Could not get any response" en Postman

**Problema:** No puede conectar con la aplicación.

**Checklist:**
1. ¿La aplicación está corriendo? (ver logs)
2. ¿La URL es correcta? (`http://localhost:8080`)
3. ¿El puerto es el correcto?
4. ¿Hay firewall bloqueando?

---

### ❓ "415 Unsupported Media Type" en POST

**Problema:** Falta Content-Type header.

**Solución en Postman:**
1. Selecciona la pestaña "Body"
2. Elige "raw"
3. Selecciona "JSON" en el dropdown
4. Verifica que el header `Content-Type: application/json` esté presente

**Solución en curl:**
```bash
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \  # ← Importante
  -d '{"title":"Task 1","completed":false}'
```

---

### ❓ "400 Bad Request" al enviar JSON

**Problema:** JSON malformado o campos incorrectos.

**Solución:**

Verifica el JSON:
```json
{
  "title": "Aprender Spring Boot",  // Comillas dobles
  "completed": false                 // Sin comillas para boolean
}
```

Errores comunes:
```json
{
  'title': 'Test',      // Comillas simples
  completed: 'false'    // Boolean como string
}
```

---

## 🔍 Problemas con Actuator

### ❓ "404 on /actuator/health"

**Problema:** Actuator no está configurado.

**Solución:**

1. Verifica dependencia en `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2. Configura en `application.yml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

3. Reinicia la aplicación

---

## Problemas Conceptuales

### ❓ "¿Cuál es la diferencia entre @RestController y @Controller?"

**Respuesta:**

| Anotación | Uso | Retorna |
|-----------|-----|---------|
| `@Controller` | MVC (vistas HTML) | Nombres de vistas (Thymeleaf, JSP) |
| `@RestController` | REST APIs | Datos (JSON, XML) |

`@RestController` = `@Controller` + `@ResponseBody`

---

### ❓ "¿Cuándo uso @RequestParam vs @PathVariable?"

**Respuesta:**

**@PathVariable** - Parte de la URL:
```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    // URL: /users/123
}
```

**@RequestParam** - Query string:
```java
@GetMapping("/users")
public List<User> searchUsers(@RequestParam String name) {
    // URL: /users?name=John
}
```

---

### ❓ "¿Por qué usar inyección de dependencias?"

**Respuesta:**

**Sin inyección (acoplado):**
```java
public class TaskController {
    private TaskService service = new TaskService();  // Acoplado
}
```

**Con inyección (desacoplado):**
```java
public class TaskController {
    private final TaskService service;

    public TaskController(TaskService service) {  // Desacoplado
        this.service = service;
    }
}
```

**Ventajas:**
- Fácil de testear (inyectar mocks)
- Fácil de cambiar implementaciones
- Spring gestiona el ciclo de vida

---

## Aún Tengo Problemas

Si tu problema no está aquí:

1. **Revisa los logs** - El error suele estar ahí
2. **Compara con la solución** - En `practicas/XX/solucion/`
3. **Consulta el cheatsheet** - [cheatsheet.md](cheatsheet.md)
4. **Pregunta en clase** - Al instructor o compañeros

---

## Recursos Adicionales

- [Spring Boot Common Application Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html)
- [Spring Web MVC](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [Stack Overflow - Spring Boot](https://stackoverflow.com/questions/tagged/spring-boot)

---

[← Volver a Clase 1](README.md)
