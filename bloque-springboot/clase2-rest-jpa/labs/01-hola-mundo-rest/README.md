# Lab 01 · Hola Mundo REST

## 1. Objetivo

Implementar el primer controlador REST del microservicio `product-service`, familiarizándote con `@RestController`, `@RequestMapping` y `@GetMapping`.

---

## 2. Comandos a ejecutar

```bash
# 1. Ir al proyecto generado
cd ~/workspace/product-service

# 2. Crear la carpeta del controlador si aún no existe
mkdir -p src/main/java/dev/alefiengo/productservice/controller

# 3. Crear el archivo HelloController.java
cat <<'EOF' > src/main/java/dev/alefiengo/productservice/controller/HelloController.java
package dev.alefiengo.productservice.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hola Spring Boot!";
    }

    @GetMapping("/hello/{name}")
    public String helloName(@PathVariable String name) {
        return "Hola " + name + "!";
    }
}
EOF

# 4. Ejecutar el servicio
mvn spring-boot:run

# 5. Probar los endpoints desde otra terminal
curl http://localhost:8080/api/hello
curl http://localhost:8080/api/hello/Alejandro
```

---

## 3. Desglose del comando

| Comando | Descripción |
|---------|-------------|
| `mkdir -p .../controller` | Crea el paquete `controller` respetando la convención de paquetes de Spring (escaneo dentro de `dev.alefiengo.productservice`). |
| `cat <<'EOF' > .../HelloController.java` | Genera la clase `HelloController` con dos endpoints GET. |
| `mvn spring-boot:run` | Levanta el servicio y expone los endpoints mediante Tomcat embebido. |
| `curl http://localhost:8080/api/hello` | Realiza una petición GET para validar la respuesta simple. |
| `curl http://localhost:8080/api/hello/Alejandro` | Prueba la versión con parámetro de ruta (`@PathVariable`). |

---

## 4. Explicación detallada

1. **Estructura de paquetes**  
   Colocar el controlador dentro de `dev.alefiengo.productservice.controller` garantiza que el `@ComponentScan` implícito (`@SpringBootApplication`) lo detecte.
2. **Anotaciones clave**  
   - `@RestController` combina `@Controller` y `@ResponseBody`, devolviendo el contenido directamente en la respuesta HTTP.  
   - `@RequestMapping("/api")` define un prefijo; todos los métodos quedan bajo `/api/...`.  
   - `@GetMapping("/hello")` indica que el método responde a solicitudes GET con ruta `/api/hello`.
3. **Parámetros en la ruta**  
   `@PathVariable` extrae el valor `name` de la URL y lo inyecta en el método Java.
4. **Respuesta**  
   Spring Boot serializa automáticamente el `String` retornado y envía estado 200. Para respuestas más elaboradas se recomendará devolver DTOs en próximos labs.

---

## 5. Conceptos aprendidos

- Controladores REST en Spring Boot y su escaneo automático.
- Uso de `@PathVariable` para parámetros dinámicos en la ruta.
- Prefijos de ruta con `@RequestMapping`.
- Ciclo de vida de una petición HTTP en Spring MVC (DispatcherServlet → controlador → respuesta).

---

## 6. Troubleshooting

- **404 a pesar de tener el controlador**  
  Verifica que la clase se encuentra dentro de `dev.alefiengo.productservice` o subpaquetes. Si la ruta está mal escrita, revisa que la URL incluya `/api/`.
- **`Cannot resolve symbol RestController`**  
  Revisa que `spring-boot-starter-web` se encuentre en el `pom.xml`. Ejecuta `mvn dependency:tree` para confirmarlo.
- **No aparece la respuesta en español**  
  Compila de nuevo (`mvn clean spring-boot:run`) para asegurarte de que el cambio fue tomado; Spring no recarga automáticamente sin DevTools.
- **La aplicación no arranca**  
  Consulta los logs. Errores de compilación comúnmente se deben a nombres de paquetes o imports incorrectos.

---

## 7. Desafío adicional/final

Modifica el controlador para que el endpoint `/api/hello/{name}` devuelva un objeto JSON con la estructura:

```json
{
  "mensaje": "Hola Alejandro",
  "timestamp": "2025-10-21T19:30:00Z"
}
```

Hint: crea una clase `GreetingResponse` con campos `mensaje` y `timestamp`, y retorna una instancia desde el método.

---

## 8. Recursos adicionales

- [Guía oficial: Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)
- [Documentación de `@RestController`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestController.html)
- [Documentación de `@PathVariable`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/PathVariable.html)
