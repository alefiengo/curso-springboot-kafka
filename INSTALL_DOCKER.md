# Guía de Instalación · Docker y Docker Compose

Docker es necesario para las clases donde levantaremos PostgreSQL, Kafka y servicios auxiliares. Sigue las instrucciones según tu sistema operativo.

---

## 1. Verificar si Docker ya está instalado

```bash
docker --version
docker compose version
```

Si ambos comandos muestran versiones, asegúrate de que Docker Engine esté en ejecución; de lo contrario, continúa con la instalación.

---

## 2. Windows (Docker Desktop)

1. Descarga Docker Desktop desde [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).  
2. Ejecuta el instalador y habilita la opción “Use WSL 2 instead of Hyper-V” si tienes Windows 10/11.  
3. Reinicia la PC si el instalador lo solicita.  
4. Abre Docker Desktop y espera a que aparezca el mensaje “Docker Desktop is running”.  
5. Verifica en PowerShell:
   ```powershell
   docker --version
   docker compose version
   ```
6. Si trabajas con WSL, asegúrate de integrar la distribución desde **Settings → Resources → WSL Integration**.

---

## 3. macOS (Docker Desktop)

1. Descarga el paquete `.dmg` desde [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).  
2. Arrastra Docker Desktop a la carpeta **Aplicaciones**.  
3. Abre Docker y concédele permisos si macOS lo solicita.  
4. Confirma funcionamiento en la terminal:
   ```bash
   docker --version
   docker compose version
   ```

---

## 4. Linux (Ubuntu/Debian)

> Requiere permisos de superusuario. Revisa la documentación oficial si usas otra distribución: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/).

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Agrega tu usuario al grupo `docker` (opcional pero recomendado):
```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verifica:
```bash
docker run hello-world
docker compose version
```

---

## 5. Configurar permisos y recursos

- **Windows/macOS**: ajusta CPU/RAM compartidos desde Docker Desktop → Settings → Resources. Para Kafka se recomiendan al menos 4 GB de RAM.  
- **Linux**: asegúrate de que el servicio Docker esté habilitado al iniciar (`sudo systemctl enable docker`).  
- Habilita la integración con tu editor/terminal si usas WSL 2 o contenedores desde VS Code.

---

## 6. Troubleshooting rápido

- `docker: command not found`: verifica que el binario esté en el `PATH`; reinstala si fue una instalación parcial.
- `permission denied while trying to connect to the Docker daemon`: agrega tu usuario al grupo `docker` (Linux) o ejecuta la terminal como administrador (Windows).
- Docker Desktop no arranca: reinicia el servicio desde la interfaz o revisa los requisitos de virtualización (BIOS/UEFI).
- `docker compose` no existe: actualiza Docker Desktop o Docker Engine; el plugin Compose v2 viene incluido en versiones recientes. Si tienes una versión muy antigua, actualiza antes de continuar con el curso.

---

## 7. Próximos pasos

- Prueba `docker run hello-world` para confirmar que los contenedores se ejecutan correctamente.  
- Ejecuta `docker compose up -d` en `bloque-springboot/clase2-rest-jpa/labs/03-postgresql-docker` cuando avances en la Clase 2.  
- Consulta las secciones FAQ de cada clase para errores específicos (por ejemplo, puertos ocupados o credenciales).

