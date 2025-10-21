# FAQ - Clase 1: Preguntas Frecuentes

Soluciones a problemas comunes de instalación y configuración del entorno.

---

## Problemas de Instalación

### "java: command not found"

**Causa**: Java no está instalado o no está en el PATH del sistema.

**Solución**:

1. Verifica si Java está instalado:
   ```bash
   which java    # Linux/macOS
   where java    # Windows
   ```

2. Si no está instalado, descarga e instala JDK 17:
   - OpenJDK: https://adoptium.net/
   - Oracle JDK: https://www.oracle.com/java/technologies/downloads/

3. Configura JAVA_HOME:
   ```bash
   # Linux/macOS (.bashrc o .zshrc)
   export JAVA_HOME=/path/to/jdk-17
   export PATH=$JAVA_HOME/bin:$PATH

   # Windows (Variables de entorno)
   JAVA_HOME=C:\Program Files\Java\jdk-17
   PATH=%JAVA_HOME%\bin;%PATH%
   ```

4. Reinicia la terminal y verifica:
   ```bash
   java -version
   ```

---

### "mvn: command not found"

**Causa**: Maven no está instalado o no está en el PATH.

**Solución**:

1. Descarga Maven: https://maven.apache.org/download.cgi

2. Descomprime en un directorio (ej: `/opt/maven` o `C:\Program Files\Maven`)

3. Configura PATH:
   ```bash
   # Linux/macOS
   export PATH=/path/to/maven/bin:$PATH

   # Windows
   PATH=C:\Program Files\Maven\bin;%PATH%
   ```

4. Verifica:
   ```bash
   mvn -version
   ```

---

### "JAVA_HOME is not set"

**Causa**: Variable de entorno JAVA_HOME no está configurada.

**Solución Linux/macOS**:
```bash
# Encuentra la ubicación de Java
/usr/libexec/java_home -V  # macOS
which java                  # Linux

# Agrega a .bashrc o .zshrc
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home  # macOS
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk                                # Linux
export PATH=$JAVA_HOME/bin:$PATH

# Recarga configuración
source ~/.bashrc
```

**Solución Windows**:
1. Panel de Control → Sistema → Configuración avanzada del sistema
2. Variables de entorno → Nuevo (Variables del sistema)
3. Nombre: `JAVA_HOME`
4. Valor: `C:\Program Files\Java\jdk-17` (tu ruta)
5. Edita PATH y agrega: `%JAVA_HOME%\bin`
6. Reinicia terminal (cmd/PowerShell)

---

### IntelliJ IDEA no reconoce JDK 17

**Causa**: IDE no está configurado para usar el JDK correcto.

**Solución**:

1. File → Project Structure (Ctrl+Alt+Shift+S)
2. Project → SDK → Add SDK → Download JDK
3. Selecciona versión 17, vendor: Temurin (recomendado)
4. Aplica cambios

**O usar JDK ya instalado**:
1. File → Project Structure → SDK → Add SDK → JDK
2. Navega a tu instalación de JDK 17
3. OK → Apply

---

### Error: "java: invalid source release: 17"

**Causa**: Proyecto configurado para usar Java 17 pero el IDE/Maven usa otra versión.

**Solución en IntelliJ**:
1. File → Settings → Build, Execution, Deployment → Compiler → Java Compiler
2. Project bytecode version: 17
3. File → Project Structure → Project → SDK: 17

**Solución en pom.xml**:
Verifica que tenga:
```xml
<properties>
    <java.version>17</java.version>
</properties>
```

---

## Problemas al Crear Proyecto

### Spring Initializr no descarga el ZIP

**Causa**: Problema de red o configuración de proxy.

**Solución**:
1. Verifica tu conexión a internet
2. Intenta con otro navegador
3. Usa IntelliJ IDEA directamente: File → New → Project → Spring Initializr

---

### "Cannot resolve Spring Boot dependencies" en IntelliJ

**Causa**: Maven no pudo descargar dependencias.

**Solución**:
1. Verifica conexión a internet
2. Click derecho en proyecto → Maven → Reload Project
3. Si persiste: File → Invalidate Caches → Invalidate and Restart

---

### Maven descarga es muy lenta

**Causa**: Servidor Maven central puede ser lento dependiendo de tu ubicación.

**Solución**:
Usar un mirror (opcional):
```xml
<!-- En ~/.m2/settings.xml -->
<settings>
  <mirrors>
    <mirror>
      <id>aliyun-maven</id>
      <mirrorOf>central</mirrorOf>
      <name>Aliyun Maven</name>
      <url>https://maven.aliyun.com/repository/central</url>
    </mirror>
  </mirrors>
</settings>
```

---

## Problemas al Ejecutar

### "Port 8080 was already in use"

**Causa**: Otra aplicación está usando el puerto 8080.

**Solución**:

**Opción 1 - Detener proceso anterior**:
```bash
# Linux/macOS
lsof -ti:8080 | xargs kill -9

# Windows
netstat -ano | findstr :8080
# Anota el PID y luego:
taskkill /PID <número> /F
```

**Opción 2 - Cambiar puerto**:
Crea `src/main/resources/application.properties`:
```properties
server.port=8081
```

---

### "Failed to execute goal org.springframework.boot:spring-boot-maven-plugin"

**Causa**: Problema con el plugin de Maven o dependencias faltantes.

**Solución**:
```bash
# Limpiar y reinstalar
mvn clean install

# Si falla, eliminar caché de Maven
rm -rf ~/.m2/repository/org/springframework/boot
mvn clean install
```

---

### "Application run failed - ClassNotFoundException"

**Causa**: Clase principal no se encuentra o ruta incorrecta en pom.xml.

**Solución**:
1. Verifica que existe la clase con `@SpringBootApplication`
2. Verifica en pom.xml (debe generarse automático):
   ```xml
   <build>
       <plugins>
           <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
           </plugin>
       </plugins>
   </build>
   ```
3. Limpia y recompila:
   ```bash
   mvn clean compile
   mvn spring-boot:run
   ```

---

### Aplicación inicia pero no responde en localhost:8080

**Causa**: Aplicación starter vacía no tiene endpoints.

**Explicación**: Esto es **NORMAL** en Clase 1. Creaste un proyecto vacío sin controllers. Verás "Tomcat started" pero al acceder a http://localhost:8080 obtendrás error 404.

**Solución**: En Clase 2 agregaremos endpoints. Por ahora solo verifica que la aplicación inicie correctamente (sin errores en consola).

---

## Problemas con Maven

### "Could not resolve dependencies"

**Causa**: Maven no puede descargar bibliotecas.

**Solución**:
1. Verifica conexión a internet
2. Intenta actualizar:
   ```bash
   mvn clean install -U
   ```
   (La bandera `-U` fuerza actualización de dependencias)

3. Si usa proxy corporativo, configura en `~/.m2/settings.xml`:
   ```xml
   <settings>
     <proxies>
       <proxy>
         <id>myproxy</id>
         <active>true</active>
         <protocol>http</protocol>
         <host>proxy.company.com</host>
         <port>8080</port>
       </proxy>
     </proxies>
   </settings>
   ```

---

### "Build Success" pero aplicación no inicia

**Causa**: Error en la configuración de la aplicación o dependencias.

**Solución**:
1. Revisa logs completos en consola
2. Busca líneas con "ERROR" o "Exception"
3. Verifica que pom.xml tenga al menos:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   ```

---

## Problemas con Git

### "Git not found" en IntelliJ

**Causa**: IntelliJ no encuentra la instalación de Git.

**Solución**:
1. File → Settings → Version Control → Git
2. Path to Git executable: Busca y selecciona git.exe (Windows) o /usr/bin/git (Linux/macOS)
3. Click "Test" para verificar

---

### No puedo hacer push a GitHub

**Causa**: Autenticación fallida o repositorio remoto no configurado.

**Solución**:
1. Configura credenciales:
   ```bash
   git config --global user.name "Tu Nombre"
   git config --global user.email "tu@email.com"
   ```

2. Si usa HTTPS, necesitas Personal Access Token (no password):
   - GitHub → Settings → Developer settings → Personal access tokens
   - Generate new token → Copia el token
   - Úsalo como password al hacer push

3. Si el repo no existe en GitHub:
   - Crea el repositorio en GitHub primero (vacío, sin README)
   - Luego conecta:
     ```bash
     git remote add origin https://github.com/usuario/repo.git
     git branch -M main
     git push -u origin main
     ```

---

## Problemas Conceptuales

### ¿Por qué mi aplicación no hace nada?

**Respuesta**: En Clase 1 solo creamos el proyecto starter (esqueleto). No tiene lógica de negocio ni endpoints aún. En Clase 2 empezaremos a escribir código Java y crear endpoints REST.

---

### ¿Qué es "Tomcat started on port 8080"?

**Respuesta**: Tomcat es el servidor web embebido en Spring Boot. El mensaje indica que tu aplicación está corriendo y escuchando en el puerto 8080. Aunque aún no tenga endpoints, el servidor está activo y listo para recibir peticiones HTTP.

---

### ¿Debo instalar Tomcat por separado?

**Respuesta**: NO. Spring Boot incluye Tomcat embebido. No necesitas instalar ni configurar Tomcat manualmente.

---

### ¿Cuál es la diferencia entre compile y package?

**Respuesta**:
- `mvn compile`: Solo compila el código Java a bytecode (.class)
- `mvn package`: Compila + empaqueta en un JAR ejecutable (en carpeta target/)

---

### ¿Para qué sirve el archivo pom.xml?

**Respuesta**: Es el archivo de configuración de Maven. Define:
- Información del proyecto (nombre, versión)
- Dependencias (bibliotecas externas)
- Plugins (herramientas de build)
- Propiedades (versión de Java, etc.)

---

### ¿Qué es un starter dependency?

**Respuesta**: Es una dependencia "umbrella" que agrupa otras relacionadas. Por ejemplo:
- `spring-boot-starter-web` incluye: Spring MVC, Tomcat, Jackson (JSON), etc.
- Evita tener que agregar cada dependencia individual.

---

## Aún Tengo Problemas

Si ninguna solución funcionó:

1. **Revisa el [Cheatsheet](cheatsheet.md)** - Comandos básicos y configuración
2. **Lee [CONCEPTOS.md](CONCEPTOS.md)** - Entender la teoría ayuda
3. **Busca el error exacto en Google** - Copia el mensaje de error completo
4. **Pregunta en el foro de Moodle** - Comparte el error y lo que ya intentaste
5. **Consulta en clase** - Trae tu laptop y revisamos juntos

---

## Recursos Adicionales

- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Maven Getting Started](https://maven.apache.org/guides/getting-started/)
- [IntelliJ IDEA Spring Boot Guide](https://www.jetbrains.com/help/idea/spring-boot.html)
- [Git Documentation](https://git-scm.com/doc)

---

**Recuerda**: La mayoría de los problemas se resuelven con:
1. Reiniciar el IDE
2. `mvn clean install`
3. Verificar versiones de Java y Maven

[← Volver a Clase 1](README.md)
