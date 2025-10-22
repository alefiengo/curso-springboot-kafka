# Guía de Instalación · Java 17 y Maven 3.9+

Esta guía resume los pasos para preparar el entorno requerido en el curso. Ajusta los comandos según tu sistema operativo.

---

## 1. Verifica si ya tienes Java y Maven

```bash
java -version
javac -version
mvn -version
```

Si ambos comandos muestran versiones 17.x (Java) y 3.9.x (Maven), puedes saltar a la sección **5. Configurar IntelliJ IDEA**.

---

## 2. Instalación en Windows

1. Descarga el instalador de Java 17 (Temurin recomendado) desde [https://adoptium.net/temurin/releases/?version=17](https://adoptium.net/temurin/releases/?version=17).  
2. Durante la instalación, marca la opción “Set JAVA_HOME variable”.  
3. Verifica en PowerShell:
   ```powershell
   java -version
   $env:JAVA_HOME
   ```
4. Instala Maven descargando el ZIP de [https://maven.apache.org/download.cgi](https://maven.apache.org/download.cgi).  
5. Extrae el ZIP en `C:\tools\apache-maven-3.9.x` (o ruta similar).  
6. Agrega la carpeta `bin` a la variable `Path` del sistema y define `MAVEN_HOME`.  
7. Abre una nueva terminal y ejecuta:
   ```powershell
   mvn -version
   ```

---

## 3. Instalación en macOS

### Java 17
```bash
brew install openjdk@17
sudo ln -sfn $(brew --prefix openjdk@17)/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk
```

Agrega en tu `~/.zshrc` o `~/.bashrc`:
```bash
export JAVA_HOME="$(/usr/libexec/java_home -v17)"
export PATH="$JAVA_HOME/bin:$PATH"
```

### Maven 3.9
```bash
brew install maven
mvn -version
```

---

## 4. Instalación en Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
sudo apt install -y maven
```

Configura variables en `~/.bashrc` o `~/.zshrc`:
```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
```

Recarga el perfil:
```bash
source ~/.bashrc
```

Confirma las versiones:
```bash
java -version
mvn -version
```

---

## 5. Configurar IntelliJ IDEA

1. Descarga la versión Community desde [JetBrains](https://www.jetbrains.com/idea/download/).  
2. Abre IntelliJ y ve a **File → New Project Setup → Structure for New Projects**.  
3. Selecciona el JDK 17 recién instalado como SDK por defecto.  
4. Habilita las opciones de Maven si deseas que IntelliJ importe automáticamente proyectos Maven.

---

## 6. Solución de problemas comunes

- `java: command not found`: revisa que la variable `PATH` incluya el directorio `bin` del JDK.  
- `mvn: command not found`: agrega la carpeta `bin` de Maven al `PATH`.  
- IDE usa una versión distinta de Java: en IntelliJ ve a **File → Project Structure → Project SDK** y selecciona Java 17.  
- Conflictos de versiones antiguas: elimina variables `JAVA_HOME` o `M2_HOME` previas antes de configurar las nuevas.

---

## 7. Próximos pasos

- Prueba `mvn archetype:generate` para validar que Maven descarga dependencias.  
- Crea un proyecto con Spring Initializr y verifica que `mvn spring-boot:run` funcione correctamente.  
- Documenta las versiones instaladas (requisito de la tarea de la Clase 1).

