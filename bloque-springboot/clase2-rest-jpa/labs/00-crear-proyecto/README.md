# Lab 00 · Generar el proyecto product-service

## 1. Objetivo

Crear el esqueleto del microservicio `product-service` con Spring Boot 3, listo para ejecutar y extender durante la clase.

---

## 2. Comandos a ejecutar

```bash
# 1. Preparar carpeta de trabajo (ajusta la ruta a tu preferencia)
mkdir -p ~/workspace && cd ~/workspace

# 2. Generar el proyecto con Spring Initializr (alternativa CLI)
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d groupId=dev.alefiengo \
  -d artifactId=product-service \
  -d name=product-service \
  -d packageName=dev.alefiengo.productservice \
  -d javaVersion=17 \
  -d dependencies=web \
  -o product-service.zip

# 3. Descomprimir y entrar al directorio
unzip -q product-service.zip
cd product-service

# 4. Ejecutar la aplicación para validar la generación
mvn spring-boot:run
```

---

## 3. Desglose del comando

| Comando | Descripción |
|---------|-------------|
| `curl https://start.spring.io/starter.zip ...` | Solicita a Spring Initializr un proyecto Maven con Java 17 y la dependencia `spring-boot-starter-web`. |
| `unzip -q product-service.zip` | Extrae el ZIP del proyecto sin mostrar salida detallada (`-q`). |
| `cd product-service` | Cambia al directorio raíz del servicio generado. |
| `mvn spring-boot:run` | Compila y arranca la aplicación usando el plugin de Spring Boot, levantando Tomcat embebido en el puerto 8080. |

---

## 4. Explicación detallada

1. **Generación**  
   Spring Initializr construye un proyecto Maven con `pom.xml`, estructura `src/` y la clase `ProductServiceApplication` anotada con `@SpringBootApplication`.
2. **Exploración**  
   Abre el proyecto en IntelliJ IDEA. Verifica la estructura `src/main/java` y `src/main/resources` junto con el archivo `application.properties`.
3. **Ejecución**  
   Al ejecutar `mvn spring-boot:run`, Maven descarga dependencias la primera vez y arranca la aplicación mostrando el banner de Spring Boot.
4. **Validación rápida**  
   Visita `http://localhost:8080`. Un mensaje 404 indica que el servidor está arriba (no hay endpoints todavía, pero la aplicación responde).

---

## 5. Conceptos aprendidos

- Uso de Spring Initializr para crear proyectos estándar.
- Relación entre `groupId`, `artifactId`, nombre del paquete y escaneo de componentes.
- Participación del plugin `spring-boot-maven-plugin` al ejecutar `spring-boot:run`.
- Confirmación del servidor embebido Tomcat como parte de `spring-boot-starter-web`.

---

## 6. Troubleshooting

- **`mvn: command not found`**  
  Asegúrate de tener Maven 3.9+ instalado y en el `PATH`. Revisa la guía de instalación de la Clase 1.
- **Spring Boot no arranca y muestra error de JDK**  
  Configura `JAVA_HOME` apuntando a JDK 17 y reinicia la terminal. En IntelliJ, revisa *Project Structure → Project SDK*.
- **Puerto 8080 ocupado**  
  Identifica el proceso con `lsof -i :8080` (Linux/macOS) o `netstat -ano | findstr 8080` (Windows) y detenlo. Como alternativa, define `server.port=8081` en `src/main/resources/application.properties`.
- **ZIP corrupto o vacío**  
  Ejecuta nuevamente el comando `curl`. Si persiste, genera el proyecto desde la interfaz web de Spring Initializr y repite los pasos posteriores.

---

## 7. Desafío adicional/final

Inicializa Git en el proyecto, crea un primer commit y documenta instrucciones básicas en un `README.md` propio.

```bash
git init
git add .
git commit -m "feat: bootstrap product-service"
```

Sube el repositorio a GitHub o GitLab y guarda el enlace para la tarea de la clase.

---

## 8. Recursos adicionales

- [Spring Initializr](https://start.spring.io/)
- [Guía de Spring Boot Starter Web](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#web)
- [Maven – Getting Started](https://maven.apache.org/guides/getting-started/)
