# Clase 1: Introducción a Spring Boot y Microservicios

Bienvenido a la primera clase del curso. Construirás tu primera aplicación REST API con Spring Boot, desde cero hasta una arquitectura profesional.

---

## En Esta Clase Aprenderás

- Crear proyectos Spring Boot con Spring Initializr
- Implementar REST APIs con diferentes tipos de endpoints
- Construir un CRUD completo sin base de datos
- Organizar código con arquitectura en capas profesional
- Monitorear aplicaciones con Spring Boot Actuator
- Aplicar inyección de dependencias

---

## Prerrequisitos

### Conocimientos
- Java básico (sintaxis, clases, objetos)
- Conceptos de HTTP y REST (GET, POST, PUT, DELETE)
- Manejo básico de terminal

### Herramientas Necesarias
- **Java 17+** - JDK instalado y configurado
- **Maven 3.9+** - Gestor de dependencias
- **IntelliJ IDEA** - IDE recomendado (Community o Ultimate)
- **Postman** - Para probar APIs (opcional pero recomendado)
- **Git** - Control de versiones

**¿No tienes todo instalado?** → [Guía de instalación completa](../../INSTALL_JAVA_MAVEN.md)

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

### Paso 2: Empieza con la Primera Práctica
**[Ir a Práctica 1: Hola Mundo](practicas/01-hola-mundo/)**

---

## Contenido de la Clase

### Prácticas Progresivas

1. **[Hola Mundo en Spring Boot](practicas/01-hola-mundo/)**
   - Crear proyecto con Spring Initializr
   - Primer endpoint REST
   - Ejecutar y probar

2. **[REST APIs con Parámetros](practicas/02-parametros/)**
   - Query parameters (`?name=value`)
   - Path variables (`/api/users/{id}`)
   - Request body (JSON)

3. **[TODO API - CRUD Completo](practicas/03-todo-api/)**
   - GET, POST, PUT, DELETE
   - ArrayList como almacenamiento
   - Códigos de estado HTTP

4. **[Actuator y Configuración](practicas/04-actuator/)**
   - Health checks
   - Métricas de aplicación
   - Archivo `application.yml`

5. **[Arquitectura en Capas](practicas/05-arquitectura-capas/)**
   - Separar Controller, Service, Model
   - Inyección de dependencias
   - Buenas prácticas

---

### Teoría y Conceptos

**[Conceptos Fundamentales](CONCEPTOS.md)** (Lectura opcional)
- ¿Qué son los microservicios?
- Arquitectura de Spring Boot
- Patrones de diseño aplicados
- Comparativas con otros frameworks

---

### Recursos de Apoyo

**[Diagramas](recursos/diagramas/)**
- Arquitectura en capas
- Flujo de petición HTTP
- Spring Boot startup flow

**[Postman Collection](recursos/postman/)**
- Colección completa de la clase
- 20 requests listos para importar

---

### Desafíos

**[Desafío Rápido](desafios/rapido.md)**
- Agregar filtrado a TODO API

**[Desafío Intermedio](desafios/intermedio.md)** (Opcional)
- Búsqueda y estadísticas

**[Desafío Avanzado](desafios/avanzado.md)** (Extra)
- Paginación y ordenamiento

---

### Tarea para Casa

**[Proyecto Individual](tarea/)**
- Crear tu propia REST API (Banking, Retail o Telecom)
- Código en GitHub/GitLab
- Rúbrica de evaluación detallada
- Proyectos de ejemplo incluidos

---

## Material de Referencia

### Durante la Clase
- **[Cheatsheet](cheatsheet.md)** - Referencia rápida de comandos y anotaciones
- **[FAQ](FAQ.md)** - Preguntas frecuentes y troubleshooting

### Después de la Clase
- **[CONCEPTOS.md](CONCEPTOS.md)** - Teoría profunda
- **[Ejemplos de Proyectos](tarea/ejemplos/)** - Proyectos de referencia completos

---

## Conceptos Clave

Al finalizar esta clase dominarás:

| Concepto | Descripción |
|----------|-------------|
| **Microservicio** | Aplicación pequeña e independiente con responsabilidad única |
| **Spring Boot** | Framework que simplifica desarrollo Spring con auto-configuración |
| **REST API** | Arquitectura para diseñar APIs web usando HTTP |
| **@RestController** | Anotación que marca una clase como controlador REST |
| **@GetMapping** | Mapea peticiones HTTP GET a métodos |
| **@PostMapping** | Mapea peticiones HTTP POST a métodos |
| **Inyección de Dependencias** | Patrón para desacoplar componentes |
| **Actuator** | Herramienta de Spring Boot para monitoreo |

**Ver glosario completo →** [cheatsheet.md#glosario-técnico](cheatsheet.md#glosario-técnico)

---

## Checklist de Finalización

Antes de terminar la clase, asegúrate de haber:

- [ ] Completado las 5 prácticas
- [ ] Ejecutado con éxito una TODO API completa
- [ ] Entendido la arquitectura Controller → Service → Model
- [ ] Probado Actuator (`/actuator/health`)
- [ ] Importado la Postman collection
- [ ] Intentado el desafío rápido
- [ ] Revisado las instrucciones de la tarea

---

## ¿Necesitas Ayuda?

### Durante la Clase
1. Revisa el **[FAQ](FAQ.md)** - Errores comunes ya documentados
2. Compara tu código con la **solución** de cada práctica
3. Pregunta al instructor o compañeros

### Después de la Clase
1. Consulta **[CONCEPTOS.md](CONCEPTOS.md)** para profundizar
2. Revisa los **[ejemplos de proyectos](tarea/ejemplos/)** de la tarea
3. Usa el **[cheatsheet](cheatsheet.md)** como referencia

---

## Próxima Clase

**Clase 2: REST APIs con Spring Data JPA**
- Conectar a PostgreSQL
- Persistencia real de datos
- Relaciones entre entidades
- Queries personalizadas

**[Ver Clase 2 →](../clase2-rest-jpa/)**

---

## Estructura de Archivos

```
clase1-introduccion/
├── README.md (estás aquí)
├── CONCEPTOS.md
├── cheatsheet.md
├── FAQ.md
├── practicas/
│   ├── 01-hola-mundo/
│   ├── 02-parametros/
│   ├── 03-todo-api/
│   ├── 04-actuator/
│   └── 05-arquitectura-capas/
├── recursos/
│   ├── diagramas/
│   └── postman/
├── desafios/
│   ├── rapido.md
│   ├── intermedio.md
│   └── avanzado.md
└── tarea/
    ├── README.md
    ├── RUBRICA.md
    └── ejemplos/
```

---

**¡Comencemos!** [Práctica 1: Hola Mundo](practicas/01-hola-mundo/)
