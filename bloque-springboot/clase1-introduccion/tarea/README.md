# Tarea para Casa - Clase 1

**Fecha de entrega:** Antes de la próxima clase (verificar en Moodle)
**Peso en la nota final:** 5%
**Modalidad:** Individual

---

## Objetivo

Asegurar que tienes tu entorno de desarrollo completamente configurado y que comprendes la estructura de un proyecto Spring Boot, preparándote para empezar a programar en la Clase 2.

---

## Parte 1: Configuración del Entorno (40%)

### Requisitos

Instala y configura las siguientes herramientas:

1. **Java Development Kit (JDK) 17+**
   - OpenJDK o Oracle JDK
   - Configura `JAVA_HOME`

2. **Apache Maven 3.9+**
   - Configura en PATH
   - Verifica instalación

3. **IntelliJ IDEA**
   - Community (gratuito) o Ultimate
   - Configura JDK en el IDE

4. **Git**
   - Instala y configura usuario/email

### Verificación

Ejecuta estos comandos y captura los resultados:

```bash
java -version
javac -version
mvn -version
git --version
```

**Evidencia:** Screenshot de la terminal mostrando las 4 versiones.

---

## Parte 2: Crear Proyecto Starter (40%)

### Paso 1: Genera el proyecto

Usa **Spring Initializr** (cualquier método):

**Método 1: Web (https://start.spring.io/)**
- Project: Maven
- Language: Java
- Spring Boot: 3.x (última estable)
- Group: `dev.alefiengo`
- Artifact: `mi-primer-springboot`
- Packaging: Jar
- Java: 17
- Dependencies: **Spring Web**

**Método 2: IntelliJ IDEA**
- File → New → Project → Spring Initializr
- Misma configuración que arriba

**Método 3: VS Code**
- Command Palette → Spring Initializr
- Misma configuración

### Paso 2: Importa y ejecuta

1. Descomprime (si usaste web)
2. Abre en IntelliJ IDEA
3. Espera que Maven descargue dependencias
4. Ejecuta: `mvn spring-boot:run`

**Evidencia:** Screenshot mostrando en terminal:
```
Tomcat started on port(s): 8080 (http)
Started MiPrimerSpringbootApplication in X.XXX seconds
```

---

## Parte 3: Exploración y Documentación (20%)

### Crea un README.md en tu proyecto

Documenta lo siguiente:

```markdown
# Mi Primer Proyecto Spring Boot

## Información del Proyecto

- **Nombre:** mi-primer-springboot
- **Versión de Spring Boot:** 3.x.x
- **Java:** 17
- **Build Tool:** Maven

## Estructura del Proyecto

Explica brevemente cada directorio/archivo:

### src/main/java/dev/alefiengo/miprimerspringboot/
- `MiPrimerSpringbootApplication.java`: [Tu explicación]

### src/main/resources/
- `application.properties`: [Tu explicación]

### src/test/
- [Tu explicación]

### pom.xml
- [Tu explicación breve de qué es y para qué sirve]

### target/
- [Tu explicación]

## Cómo Ejecutar

\```bash
mvn spring-boot:run
\```

## Dependencias Principales

Lista las dependencias en `pom.xml` y explica brevemente cada una:
- spring-boot-starter-web: [Tu explicación]
- spring-boot-starter-test: [Tu explicación]

## Conceptos Aprendidos

- ¿Qué es Spring Boot?
- ¿Qué es Maven?
- ¿Qué significa "Tomcat started on port 8080"?
- ¿Para qué sirve la anotación @SpringBootApplication?

## Autor
Tu Nombre - Curso Spring Boot & Kafka
```

---

## Entrega

### 1. Repositorio Git

Crea un repositorio público en GitHub o GitLab:

**Nombre del repo:** `mi-primer-springboot`

**Debe contener:**
```
mi-primer-springboot/
├── src/
├── pom.xml
├── README.md (con la documentación solicitada)
├── .gitignore
└── screenshots/
    ├── 01-versiones.png (java, maven, git)
    └── 02-app-running.png (Tomcat started)
```

### 2. En Moodle

Sube un archivo de texto (.txt) con:

```
Nombre: Tu Nombre Completo
Repositorio: https://github.com/tu-usuario/mi-primer-springboot

Versiones instaladas:
- Java: 17.x.x
- Maven: 3.9.x
- IDE: IntelliJ IDEA Community/Ultimate

Observaciones:
[Cualquier dificultad que hayas tenido o comentario adicional]
```

---

## Criterios de Evaluación

| Criterio | Peso | Descripción |
|----------|------|-------------|
| Entorno configurado correctamente | 40% | Java, Maven, Git instalados y funcionando |
| Proyecto Spring Boot ejecuta | 40% | `mvn spring-boot:run` funciona sin errores |
| README.md completo | 15% | Documentación clara de estructura y conceptos |
| Repositorio Git | 5% | Proyecto subido a GitHub/GitLab |

**Total:** 100%

---

## Preguntas Frecuentes

### ¿Qué hago si `mvn spring-boot:run` falla?

Revisa:
1. ¿Está Maven instalado? (`mvn -version`)
2. ¿Está Java instalado? (`java -version`)
3. ¿Esperaste a que Maven descargue todas las dependencias?

Consulta el [FAQ](../FAQ.md) para errores comunes.

### ¿Puedo usar Eclipse o VS Code en lugar de IntelliJ?

Sí, pero IntelliJ IDEA tiene mejor soporte para Spring Boot. Si usas otro IDE, documéntalo en tu entrega.

### ¿Debo escribir código Java?

**NO**. Para esta tarea solo necesitas crear el proyecto starter y documentar su estructura. El código lo escribiremos en Clase 2.

### ¿Qué pongo en .gitignore?

Usa este contenido:

```gitignore
target/
.idea/
*.iml
.DS_Store
```

### No tengo cuenta en GitHub

Puedes usar GitLab (https://gitlab.com) que es gratuito. O crea una cuenta en GitHub (https://github.com).

---

## Recursos de Apoyo

- [Guía de Instalación Java + Maven](../../INSTALL_JAVA_MAVEN.md)
- [FAQ - Clase 1](../FAQ.md)
- [CONCEPTOS.md - Teoría de Microservicios](../CONCEPTOS.md)
- [Spring Initializr](https://start.spring.io/)
- [IntelliJ IDEA Download](https://www.jetbrains.com/idea/download/)

---

## Consejos

1. **Haz la tarea pronto** - No esperes al último día, pueden surgir problemas de instalación
2. **Documenta los errores** - Si algo falla, anótalo para preguntar en clase
3. **Lee CONCEPTOS.md** - Entender la teoría te ayudará en las próximas clases
4. **Explora el proyecto** - Abre los archivos, léelos, trata de entender qué hace cada cosa
5. **Pregunta en Moodle** - Si tienes dudas, usa el foro del curso

---

## Para Profundizar (Opcional)

Si terminas rápido y quieres adelantarte para Clase 2:

- Lee la documentación oficial: [Spring Boot Getting Started](https://spring.io/guides/gs/spring-boot/)
- Mira qué anotaciones existen en `@SpringBootApplication`
- Investiga qué es un servlet container (Tomcat)
- Explora el archivo `pom.xml` línea por línea

---

**¡Nos vemos en Clase 2 para empezar a programar!**

[← Volver a Clase 1](../README.md)
