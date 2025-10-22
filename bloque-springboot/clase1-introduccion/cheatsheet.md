# Cheatsheet - Clase 1: Setup y Fundamentos

Referencia rápida de comandos esenciales para configurar tu entorno de desarrollo.

---

## Verificación de Instalación

```bash
# Verificar Java
java -version
javac -version
# Debe mostrar: openjdk version "17.0.x" o superior

# Verificar Maven
mvn -version
# Debe mostrar: Apache Maven 3.9.x o superior

# Verificar Git
git --version

# Ver variables de entorno (opcional)
echo $JAVA_HOME   # Linux/macOS
echo %JAVA_HOME%  # Windows
```

---

## Comandos Maven Básicos

```bash
# Compilar proyecto
mvn compile

# Ejecutar aplicación Spring Boot
mvn spring-boot:run

# Limpiar archivos compilados
mvn clean

# Compilar + empaquetar JAR
mvn clean package

# Ver árbol de dependencias
mvn dependency:tree

# Descargar dependencias
mvn dependency:resolve
```

---

## Comandos Git Básicos

```bash
# Inicializar repositorio
git init

# Configurar usuario
git config --global user.name "Tu Nombre"
git config --global user.email "tu@email.com"

# Ver configuración
git config --list

# Agregar archivos
git add .
git add archivo.java

# Hacer commit
git commit -m "mensaje descriptivo"

# Ver estado
git status

# Ver historial
git log --oneline

# Crear repositorio remoto y subir
git remote add origin https://github.com/usuario/repo.git
git branch -M main
git push -u origin main
```

---

## Estructura de Proyecto Maven

```
mi-proyecto/
├── src/
│   ├── main/
│   │   ├── java/              # Código fuente Java
│   │   │   └── dev/alefiengo/app/
│   │   │       └── Application.java
│   │   └── resources/         # Archivos de configuración
│   │       └── application.properties
│   └── test/
│       └── java/              # Tests
├── target/                    # Archivos compilados (generado)
├── pom.xml                    # Configuración Maven
├── .gitignore
└── README.md
```

---

## Archivo pom.xml (Estructura Básica)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.x.x</version>
    </parent>

    <groupId>dev.alefiengo</groupId>
    <artifactId>mi-proyecto</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Mi Proyecto</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Clase Principal Spring Boot

```java
package dev.alefiengo.miapp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MiAppApplication {
    public static void main(String[] args) {
        SpringApplication.run(MiAppApplication.class, args);
    }
}
```

**¿Qué hace @SpringBootApplication?**
- Combina 3 anotaciones: @Configuration, @EnableAutoConfiguration, @ComponentScan
- Habilita auto-configuración de Spring Boot
- Escanea componentes en el paquete actual y subpaquetes

---

## .gitignore Recomendado

```gitignore
# Maven
target/

# IntelliJ IDEA
.idea/
*.iml

# Eclipse
.project
.classpath
.settings/

# VS Code
.vscode/

# macOS
.DS_Store

# Logs
*.log

# Local config
application-local.properties
application-local.yml
```

---

## Troubleshooting Común

### Error: "java: invalid source release: 17"

**Causa**: IntelliJ no está usando JDK 17

**Solución**:
1. File → Project Structure → Project → SDK: 17
2. File → Settings → Build, Execution, Deployment → Compiler → Java Compiler → Project bytecode version: 17

---

### Error: "JAVA_HOME is not set"

**Causa**: Variable de entorno JAVA_HOME no configurada

**Solución Linux/macOS**:
```bash
export JAVA_HOME=/path/to/jdk-17
export PATH=$JAVA_HOME/bin:$PATH
```

**Solución Windows**:
1. Panel de Control → Sistema → Variables de entorno
2. Nueva variable de sistema: `JAVA_HOME` = `C:\Program Files\Java\jdk-17`
3. Agregar a PATH: `%JAVA_HOME%\bin`

---

### Error: "mvn: command not found"

**Causa**: Maven no está en PATH

**Solución**:
1. Verifica instalación: busca carpeta de Maven
2. Agrega a PATH:
   ```bash
   export PATH=/path/to/maven/bin:$PATH  # Linux/macOS
   ```

---

### Error: "Failed to execute goal... package does not exist"

**Causa**: Falta dependencia en pom.xml

**Solución**:
1. Verifica que la dependencia esté en `<dependencies>`
2. Ejecuta: `mvn clean install`
3. Refresca proyecto en IDE (Reimport)

---

### Aplicación no inicia: "Port 8080 was already in use"

**Causa**: Otra aplicación usando puerto 8080

**Solución**:

**Opción 1**: Detener aplicación anterior
```bash
# Linux/macOS
lsof -ti:8080 | xargs kill

# Windows
netstat -ano | findstr :8080
taskkill /PID <PID> /F
```

**Opción 2**: Cambiar puerto en `application.properties`
```properties
server.port=8081
```

---

## Glosario Técnico

| Término | Descripción |
|---------|-------------|
| Microservicio | Aplicación pequeña e independiente con una responsabilidad específica. |
| Spring Boot | Framework que simplifica desarrollo Spring con auto-configuración. |
| Maven | Herramienta de gestión de proyectos y construcción para Java. |
| Dependencia | Biblioteca externa que el proyecto necesita para compilarse o ejecutarse. |
| Artefacto | Resultado del proceso de construcción (por ejemplo, archivo JAR). |
| POM | Archivo `pom.xml` que describe configuración y dependencias de Maven. |
| Starter | Dependencia que agrupa otras relacionadas (ejemplo: `spring-boot-starter-web`). |
| Empaquetado | Formato final del artefacto (jar, war). |
| Servidor embebido | Servidor HTTP incluido en el JAR, sin instalación externa (Tomcat, Jetty). |
| Auto-configuración | Capacidad de Spring Boot para configurar componentes según las dependencias presentes. |

---

## Conceptos Clave de Microservicios

**Microservicio:**
- Aplicación pequeña, independiente
- Una responsabilidad (Single Responsibility Principle)
- Desplegable de forma independiente
- Comunica vía APIs (HTTP/REST, mensajería)

**Características:**
- Independencia tecnológica
- Escalabilidad horizontal
- Resiliencia (fallo aislado)
- Facilita CI/CD

**Spring Boot para Microservicios:**
- Embedded server (no necesita Tomcat externo)
- Auto-configuración (menos código)
- Starter dependencies (fácil setup)
- Actuator (monitoreo)

---

## Spring Initializr

**Web:** https://start.spring.io/

**Configuración típica:**
- Project: Maven
- Language: Java
- Spring Boot: 3.x (última estable)
- Group: `dev.alefiengo`
- Artifact: nombre-proyecto
- Packaging: Jar
- Java: 17

**Dependencies básicas:**
- Spring Web (REST APIs)
- Spring Boot DevTools (hot reload)
- Spring Boot Actuator (monitoreo)

---

## Comandos Útiles del Sistema

```bash
# Ver procesos Java corriendo
jps

# Ver uso de puerto
netstat -an | grep 8080    # Linux/macOS
netstat -an | findstr 8080 # Windows

# Forzar descarga limpia de dependencias (opcional)
mvn dependency:purge-local-repository

# Ver versión de dependencias
mvn versions:display-dependency-updates
```

---

## Recursos Adicionales

### Documentación Oficial
- [Spring Boot Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Maven Docs](https://maven.apache.org/guides/)
- [Git Docs](https://git-scm.com/doc)

### Tutoriales
- [Spring Boot Getting Started](https://spring.io/guides/gs/spring-boot/)
- [Maven in 5 Minutes](https://maven.apache.org/guides/getting-started/maven-in-five-minutes.html)

### Herramientas
- [Spring Initializr](https://start.spring.io/)
- [IntelliJ IDEA](https://www.jetbrains.com/idea/)
- [Git Documentation](https://git-scm.com/)

---

**Nota:** Este cheatsheet cubre solo lo visto en Clase 1 (setup y fundamentos). Se expandirá en clases siguientes.

[← Volver a Clase 1](README.md)
