# Clase 1: Introducción a Spring Boot y Microservicios

Bienvenido a la primera clase del curso. En esta sesión prepararás tu entorno de desarrollo y crearás tu primer proyecto Spring Boot starter.

---

## En Esta Clase Aprenderás

- Configurar el entorno de desarrollo (Java 17, Maven 3.9+, IntelliJ IDEA)
- Comprender qué es un microservicio y sus características
- Conocer Spring Boot y su ecosistema
- Crear un proyecto Spring Boot con Spring Initializr (web, IntelliJ, VS Code)
- Entender la estructura de un proyecto Maven/Spring Boot
- Ejecutar una aplicación Spring Boot vacía

---

## Prerrequisitos

### Conocimientos
- Java básico (sintaxis, clases, objetos)
- Conceptos básicos de HTTP
- Manejo básico de terminal/línea de comandos

### Herramientas a Instalar
- **Java 17+** - JDK (OpenJDK o Oracle JDK)
- **Maven 3.9+** - Gestor de dependencias
- **IntelliJ IDEA** - IDE recomendado (Community o Ultimate)
- **Git** - Control de versiones

**[Guía de instalación completa →](../../INSTALL_JAVA_MAVEN.md)**

---

## Comienza Aquí

### Paso 1: Verifica tu Entorno

```bash
# Verificar Java
java -version
# Debe mostrar: openjdk version "17.0.x" o superior

# Verificar Maven
mvn -version
# Debe mostrar: Apache Maven 3.9.x o superior

# Verificar Git
git --version
```

### Paso 2: Crea tu Primer Proyecto

Usa Spring Initializr para generar un proyecto starter:
- Web: https://start.spring.io/
- IntelliJ IDEA: File → New → Project → Spring Initializr
- VS Code: Command Palette → Spring Initializr

**Configuración:**
- Group: `dev.alefiengo`
- Artifact: `mi-primer-springboot`
- Dependencies: **Spring Web**

---

## Contenido de la Clase

### 1. Teoría de Microservicios

**[CONCEPTOS.md](CONCEPTOS.md)** - Lectura fundamental

**Temas cubiertos:**
- ¿Qué es un microservicio?
- Monolito vs Microservicios
- Características de los microservicios
- Por qué Spring Boot
- Arquitectura de Spring Boot

### 2. Estructura de un Proyecto Spring Boot

Explorarás y entenderás:

```
mi-primer-springboot/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── dev/alefiengo/miprimerspringboot/
│   │   │       └── MiPrimerSpringbootApplication.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/
├── target/                    # Archivos compilados
├── pom.xml                    # Configuración Maven
└── .gitignore
```

**Archivos clave:**
- `pom.xml` - Define dependencias y configuración del proyecto
- `MiPrimerSpringbootApplication.java` - Punto de entrada (@SpringBootApplication)
- `application.properties` - Configuración de la aplicación

### 3. Ejecutar el Proyecto

```bash
mvn spring-boot:run
```

**Deberías ver:**
```
Tomcat started on port(s): 8080 (http)
Started MiPrimerSpringbootApplication in X.XXX seconds
```

---

## Tarea para Casa

**[Proyecto Individual](tarea/)**

**Objetivo:** Configurar tu entorno y crear tu proyecto starter

**Entregas:**
1. Screenshots de versiones instaladas (Java, Maven, Git)
2. Proyecto Spring Boot funcionando (screenshot de Tomcat started)
3. README documentando la estructura del proyecto
4. Repositorio en GitHub/GitLab

**Peso:** 5% de la nota final

---

## Material de Referencia

### Durante la Clase
- **[Cheatsheet](cheatsheet.md)** - Comandos esenciales de Maven y Git
- **[FAQ](FAQ.md)** - Errores comunes de instalación y soluciones

### Después de la Clase
- **[CONCEPTOS.md](CONCEPTOS.md)** - Teoría profunda sobre microservicios y Spring Boot

---

## Conceptos Clave

Al finalizar esta clase comprenderás:

| Concepto | Descripción |
|----------|-------------|
| **Microservicio** | Aplicación pequeña e independiente con una responsabilidad específica |
| **Spring Boot** | Framework que simplifica el desarrollo Spring con auto-configuración |
| **Maven** | Herramienta de gestión de proyectos y construcción para Java |
| **@SpringBootApplication** | Anotación principal que habilita auto-configuración |
| **Embedded Server** | Tomcat incluido en el JAR, no requiere instalación externa |
| **Starter** | Dependencia que agrupa otras relacionadas (ej: spring-boot-starter-web) |

**Ver glosario completo →** [cheatsheet.md#glosario-técnico](cheatsheet.md#glosario-técnico)

---

## Checklist de Finalización

Antes de terminar la clase, asegúrate de haber:

- [ ] Instalado Java 17+ correctamente
- [ ] Instalado Maven 3.9+ correctamente
- [ ] Configurado IntelliJ IDEA con JDK
- [ ] Creado tu primer proyecto Spring Boot
- [ ] Ejecutado exitosamente `mvn spring-boot:run`
- [ ] Visto en consola "Tomcat started on port(s): 8080"
- [ ] Comprendido la estructura básica del proyecto
- [ ] Leído CONCEPTOS.md (al menos introducción)
- [ ] Revisado las instrucciones de la tarea

---

## ¿Necesitas Ayuda?

### Durante la Clase
1. Revisa el **[FAQ](FAQ.md)** - Errores comunes de instalación ya documentados
2. Pregunta al instructor
3. Consulta con compañeros

### Después de la Clase
1. Consulta **[CONCEPTOS.md](CONCEPTOS.md)** para profundizar en teoría
2. Revisa la **[guía de instalación](../../INSTALL_JAVA_MAVEN.md)** paso a paso
3. Usa el **[cheatsheet](cheatsheet.md)** como referencia rápida
4. Pregunta en el foro de Moodle

---

## Próxima Clase

**Clase 2: REST APIs con Spring Boot**
- Primer endpoint "Hola Mundo"
- Endpoints con parámetros (PathVariable, RequestParam)
- CRUD básico (GET, POST)
- Configuración con application.yml

**Preparación:** Asegúrate de tener tu entorno funcionando antes de Clase 2. En la próxima sesión empezaremos a escribir código Java.

---

## Estructura de Archivos de la Clase

```
clase1-introduccion/
├── README.md (estás aquí)
├── CONCEPTOS.md (teoría fundamental)
├── cheatsheet.md (referencia rápida)
├── FAQ.md (errores comunes)
└── tarea/
    └── README.md (instrucciones de tarea)
```

---

**¡Bienvenido al curso! Nos vemos en Clase 2 para empezar a programar.**

[← Volver al Curso](../../README.md)
