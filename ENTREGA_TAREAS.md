# Guía de Entrega de Tareas

Todas las tareas del curso **Spring Boot & Apache Kafka** se entregan mediante un **repositorio público** (GitHub recomendado, GitLab opcional). A continuación encontrarás los pasos detallados para crear tu repositorio, subir la solución y compartirla con el instructor.

---

## 1. Crear tu repositorio (GitHub)

1. Ingresa a https://github.com y crea una cuenta si aún no la tienes.
2. Haz clic en **New repository**.
3. Nombre sugerido: `tareas-springboot-kafka`. Puedes usar otro nombre si lo prefieres.
4. Selecciona **Public** y marca la casilla **Add a README file** para iniciar con un README vacío.
5. Presiona **Create repository**.

> ¿Prefieres GitLab? El proceso es similar: https://gitlab.com → **New Project** → **Create new project**.

---

## 2. Clonar el repositorio en tu máquina

```bash
# En tu carpeta de trabajo (por ejemplo ~/workspace)
git clone https://github.com/<tu-usuario>/tareas-springboot-kafka.git
cd tareas-springboot-kafka
```

Si nunca has utilizado Git, puedes instalar GitHub Desktop o GitKraken para trabajar gráficamente. En todos los casos, asegúrate de conocer la carpeta local donde quedará tu repositorio.

---

## 3. Estructura recomendada por clase

```
tareas-springboot-kafka/
├── README.md                # Índice general del curso
├── clase1-introduccion/
│   ├── README.md            # Desarrollo y evidencias de la tarea
│   └── recursos/            # Capturas o archivos de apoyo
├── clase2-rest-jpa/
│   ├── README.md
│   ├── src/                 # Código de la tarea si corresponde
│   └── recursos/
├── clase3-arquitectura-capas/
│   ├── README.md
│   ├── src/
│   └── recursos/
└── ...
```

Adapta el contenido según la tarea. Lo importante es mantener un README con pasos claros y evidencias, y guardar los archivos de código en `src/` o la carpeta que se requiera.

---

## 4. Contenido mínimo de cada README por clase

- Objetivo de la tarea.
- Pasos ejecutados (comandos, configuraciones, capturas).
- Resultado obtenido (salidas de consola, evidencias).
- Notas/dificultades (opcional pero recomendado).

Ejemplo de fragmento:

```markdown
# Clase 2 - REST APIs y Persistencia

## Objetivo
Implementar un CRUD sencillo con Spring Data JPA.

## Pasos
1. Configurar PostgreSQL con Docker Compose (`docker compose up -d`).
2. Ejecutar la aplicación (`mvn spring-boot:run`).
3. Probar endpoints con Postman (capturas en recursos/postman.png).

## Resultado
- `GET /api/products` devuelve los productos creados.
- `POST /api/products` crea un nuevo registro en la base de datos.
```

Incluye capturas dentro del README usando `![texto](recursos/imagen.png)`.

---

## 5. Workflow básico con Git

Después de completar la tarea de una clase:

```bash
# Verifica el estado del repositorio
git status

# Agrega los archivos nuevos o modificados
git add .

# Registra un commit con un mensaje descriptivo
git commit -m "feat: tarea clase 2"

# Sube los cambios a tu repositorio remoto
git push origin main
```

Repite el flujo `add → commit → push` cada vez que avances. Mantener un historial de commits claro facilita la revisión.

---

## 6. Compartir la tarea

1. Asegúrate de que tu repositorio **sea público**.
2. Confirma que todos los archivos (README, código, capturas) estén subidos y visibles en GitHub/GitLab.
3. Copia el enlace al repositorio, por ejemplo: `https://github.com/tu-usuario/tareas-springboot-kafka`.
4. Ingresa a Moodle y pega ese enlace en la actividad correspondiente antes de la fecha límite.

> No se aceptarán entregas por `.zip` ni enlaces privados.

---

## 7. Checklist antes de entregar

- [ ] Repositorio público con la estructura propuesta.
- [ ] README de la clase con pasos, comandos y evidencias.
- [ ] Archivos de código y recursos organizados.
- [ ] Commits con mensajes descriptivos.
- [ ] Enlace enviado a Moodle antes del plazo.

---

## Recursos de apoyo

- [Guía oficial de Git (libro Pro Git, capítulo 1)](https://git-scm.com/book/es/v2)
- [GitHub Docs – Crear un repositorio](https://docs.github.com/es/get-started/quickstart/create-a-repo)
- [GitHub Desktop – Tutorial](https://docs.github.com/es/desktop/overview/getting-started-with-github-desktop)
- [Markdown Guide](https://www.markdownguide.org/basic-syntax/)

Si nunca has usado Git y necesitas apoyo adicional, avisa al instructor; se pueden organizar sesiones breves de acompañamiento.

---

Esta guía aplica para todas las clases. Mantén tu repositorio actualizado; será útil para tu portafolio y para preparar el proyecto integrador.
