# Diagramas - Clase 1

Este directorio contiene diagramas profesionales en formato Draw.io para visualizar conceptos clave de la Clase 1.

---

## Diagramas Disponibles

### 1. Arquitectura en Capas (01-arquitectura-capas.drawio)

**Descripción**: Muestra la separación de responsabilidades en una aplicación Spring Boot.

**Conceptos ilustrados:**
- Presentation Layer (@RestController)
- Business Logic Layer (@Service)
- Data Access Layer (@Repository)
- Database

**Cuándo usar**: Para entender cómo organizar el código en capas.

**Relacionado con**: Práctica 5

---

### 2. Flujo de Petición HTTP (02-flujo-peticion-http.drawio)

**Descripción**: Visualiza el recorrido completo de una petición HTTP desde el cliente hasta la respuesta.

**Conceptos ilustrados:**
- Cliente (navegador/Postman)
- Tomcat embebido
- DispatcherServlet
- Controller
- Service
- Serialización JSON

**Cuándo usar**: Para entender cómo Spring Boot procesa requests.

**Relacionado con**: Práctica 3

---

### 3. Spring Boot Startup (03-springboot-startup.drawio)

**Descripción**: Flujo de inicio de una aplicación Spring Boot paso a paso.

**Conceptos ilustrados:**
- SpringApplication.run()
- ApplicationContext
- Component Scan
- Bean creation
- Dependency Injection
- Auto-configuration
- Embedded server startup

**Cuándo usar**: Para entender qué sucede al ejecutar `mvn spring-boot:run`.

**Relacionado con**: Práctica 1, CONCEPTOS.md

---

## Cómo Abrir los Diagramas

### Opción 1: Draw.io Desktop (Recomendado)

1. Descarga e instala [Draw.io Desktop](https://github.com/jgraph/drawio-desktop/releases)
2. Abre la aplicación
3. File → Open → Selecciona el archivo `.drawio`

### Opción 2: Draw.io Web

1. Abre [https://app.diagrams.net/](https://app.diagrams.net/)
2. Click en "Open Existing Diagram"
3. Selecciona el archivo `.drawio` desde tu computadora

### Opción 3: Integración con IntelliJ IDEA

1. Instala el plugin "diagrams.net Integration"
2. File → Settings → Plugins → Search "diagrams.net"
3. Abre el archivo `.drawio` directamente en IntelliJ

### Opción 4: VS Code

1. Instala la extensión "Draw.io Integration"
2. Abre el archivo `.drawio` directamente en VS Code

---

## Editar Diagramas

Si quieres personalizar los diagramas:

1. Abre el archivo en Draw.io
2. Edita según necesites
3. File → Save

Los archivos están en formato XML legible y versionable con Git.

---

## Exportar a Imagen

Para usar en presentaciones o documentación:

1. Abre el diagrama en Draw.io
2. File → Export as → PNG (o SVG, PDF, JPEG)
3. Configura resolución (recomendado: 300 DPI)
4. Export

---

## Paleta de Colores Usada

Los diagramas usan colores consistentes para facilitar el aprendizaje:

| Color | Componente | Ejemplo |
|-------|------------|---------|
| **Azul claro** (#dae8fc) | Cliente / Presentation Layer | Controller |
| **Verde claro** (#d5e8d4) | Business Logic | Service |
| **Amarillo claro** (#fff2cc) | Data Access | Repository |
| **Rosa claro** (#f8cecc) | Database / Datos | PostgreSQL |
| **Naranja claro** (#ffe6cc) | Infraestructura | Tomcat |

---

## Uso en Clase

### Para Instructores

Estos diagramas pueden:
- Proyectarse durante explicaciones teóricas
- Imprimirse para distribución física
- Incluirse en presentaciones de Moodle

### Para Estudiantes

Puedes:
- Consultarlos mientras programas
- Usarlos como referencia visual
- Modificarlos para tus propios proyectos

---

## Crear tus Propios Diagramas

Para el proyecto de la tarea, puedes crear diagramas similares:

1. Usa Draw.io
2. Sigue la misma paleta de colores
3. Mantén el estilo simple y claro
4. Incluye leyendas cuando sea necesario

**Recursos de Draw.io:**
- [Tutoriales oficiales](https://www.diagrams.net/doc/)
- [Plantillas](https://www.diagrams.net/blog/template-diagrams)

---

## Troubleshooting

### Problema: No puedo abrir el archivo .drawio

**Solución**: Los archivos `.drawio` son archivos XML. Necesitas Draw.io para abrirlos. Usa una de las opciones anteriores.

---

### Problema: El diagrama se ve pixelado

**Solución**: Exporta como SVG (vectorial) en lugar de PNG para mantener calidad en cualquier tamaño.

---

### Problema: Quiero modificar los colores

**Solución**:
1. Selecciona el elemento
2. Panel derecho → Style
3. Cambia Fill color

---

## Recursos Adicionales

- [Draw.io Documentation](https://www.diagrams.net/doc/)
- [Keyboard Shortcuts](https://www.diagrams.net/doc/faq/shortcut-keys)
- [Shape Libraries](https://www.diagrams.net/blog/shape-libraries)

---

[← Volver a Recursos](../) | [← Volver a Clase 1](../../README.md)
