# Lab 03 · PostgreSQL con Docker Compose

## 1. Objetivo

Levantar PostgreSQL 15 mediante Docker Compose, validar la conexión y dejar preparado el entorno de base de datos para los siguientes laboratorios.

---

## 2. Comandos a ejecutar

```bash
# 1. Ubicarse en la raíz del proyecto (ajusta la ruta si tu carpeta es distinta)
cd ~/workspace/product-service

# 2. Crear el archivo docker-compose.yml
cat <<'EOF' > docker-compose.yml
services:
  postgres:
    image: postgres:15-alpine
    container_name: product-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: product_db
      POSTGRES_USER: product_user
      POSTGRES_PASSWORD: product_password
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
EOF

# 3. Levantar el contenedor
docker compose up -d

# 4. Verificar estado
docker compose ps

# 5. Inspeccionar logs iniciales (opcional)
docker compose logs -f postgres
```

---

## 3. Desglose del comando

| Comando | Descripción |
|---------|-------------|
| `docker-compose.yml` | Define el servicio `postgres` usando la imagen oficial 15-alpine con credenciales y volumen persistente. |
| `docker compose up -d` | Descarga la imagen (si es necesario) y arranca el contenedor en segundo plano. |
| `docker compose ps` | Muestra el estado de los servicios definidos en el archivo Compose. |
| `docker compose logs -f postgres` | Sigue los logs en tiempo real para confirmar que PostgreSQL quedó listo para aceptar conexiones. |

---

## 4. Explicación detallada

1. **Imagen `postgres:15-alpine`**  
   Versión ligera basada en Alpine Linux, suficiente para entornos de laboratorio.
2. **Variables de entorno**  
   `POSTGRES_DB`, `POSTGRES_USER` y `POSTGRES_PASSWORD` inicializan la base de datos, usuario y contraseña. Los mismos valores se utilizarán en `application.yml`.
3. **Volumen persistente**  
   `postgres-data` almacena los datos fuera del contenedor, evitando perder información al reiniciar.
4. **Exposición de puertos**  
   `5432:5432` permite conectarse desde la máquina host con clientes como DBeaver, psql o IntelliJ Database.

---

## 5. Conceptos aprendidos

- Definición de servicios con Docker Compose sin necesidad de instalar PostgreSQL localmente.
- Importancia de usar volúmenes para preservar los datos entre reinicios.
- Lectura y validación de logs para diagnosticar problemas de arranque.
- Variables de entorno como mecanismo estándar de configuración.

---

## 6. Troubleshooting

- **`port is already allocated`**  
  Puerto 5432 ocupado. Ajusta el mapeo a `"5433:5432"` y usa ese puerto al conectarte.
- **El contenedor se detiene inmediatamente**  
  Revisa los logs. Errores comunes: contraseña con caracteres especiales sin comillas o falta de memoria.
- **No puedo conectar desde DBeaver**  
  Asegúrate de usar `localhost`, puerto `5432`, usuario `product_user`, contraseña `product_password` y base de datos `product_db`. Verifica que el contenedor esté en estado `running`.
- **`docker compose` no existe**  
  Versiones antiguas de Docker usan `docker-compose`. Actualiza Docker o instala el plugin oficial.

---

## 7. Desafío adicional/final

Agrega un servicio opcional `pgadmin` o `adminer` al `docker-compose.yml` para administrar la base de datos desde el navegador. Asegúrate de protegerlo con credenciales y documenta la URL de acceso.

---

## 8. Recursos adicionales

- [Documentación oficial de PostgreSQL](https://www.postgresql.org/docs/)
- [Referencia de Docker Compose](https://docs.docker.com/compose/)
- [Postgres Docker Official Images](https://hub.docker.com/_/postgres)
