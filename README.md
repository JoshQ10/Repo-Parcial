# Repo-Parcial

Guía rápida con los comandos usados en los laboratorios de TDSE (LAB05 – LAB07) para:

1. Iniciar el servicio localmente.
2. Crear y correr los contenedores con Docker / Docker Compose.
3. Publicar la imagen en Docker Hub.
4. Crear una instancia EC2 en AWS Academy, conectarse por SSH y desplegar el contenedor.
5. (Sección aparte) Despliegue en **dos servidores** con **shutdown seguro** y **stop**.

> Reemplazar los valores entre `<...>`: `<dockerhub-user>`, `<app>`, `<ip-publica-ec2>`, etc.
> Los ejemplos asumen una app Java (Maven) que lee el puerto de la variable `PORT`.

---

## Índice

- [Requisitos](#requisitos)
- [1. Iniciar el servicio local](#1-iniciar-el-servicio-local)
- [2. Contenedores con Docker](#2-contenedores-con-docker)
- [3. Docker Compose](#3-docker-compose)
- [4. Publicar en Docker Hub](#4-publicar-en-docker-hub)
- [5. AWS Academy: crear la instancia EC2](#5-aws-academy-crear-la-instancia-ec2)
- [6. Conexión por SSH](#6-conexión-por-ssh)
- [7. Desplegar el contenedor en EC2](#7-desplegar-el-contenedor-en-ec2)
- [8. Detener y limpiar](#8-detener-y-limpiar)
- [Anexo: dos servidores, shutdown seguro y stop](#anexo-dos-servidores-shutdown-seguro-y-stop)

---

## Requisitos

- Java 21 y Maven 3.9+
- Docker Desktop (incluye Docker Compose v2)
- Cuenta en Docker Hub
- Acceso a AWS Academy (Learner Lab)

```bash
java -version
mvn -version
docker --version
docker compose version
```

---

## 1. Iniciar el servicio local

```bash
mvn clean package              # compila, corre tests y genera target/*.jar
java -jar target/<app>.jar     # inicia el servicio (puerto por defecto de la app)
```

Con otro puerto:

```bash
# Linux / macOS / Git Bash
PORT=9000 java -jar target/<app>.jar
```

```powershell
# PowerShell
$env:PORT=9000; java -jar target/<app>.jar
```

Probar:

```bash
curl "http://localhost:9000/greeting?name=Pedro"
# Hello, Pedro!
```

Detener: `Ctrl + C`.

---

## 2. Contenedores con Docker

### Dockerfile

```dockerfile
FROM amazoncorretto:21

WORKDIR /app

COPY target/*.jar app.jar

ENV PORT=9000

EXPOSE 9000

# Forma exec: la JVM es PID 1 y recibe SIGTERM en `docker stop`
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Construir la imagen y correr el contenedor

```bash
mvn clean package
docker build -t <dockerhub-user>/<app>:1.0 .
docker images

docker run -d --name <app>-1 -e PORT=9000 -p 34000:9000 <dockerhub-user>/<app>:1.0
docker ps
docker logs <app>-1
```

Probar: `http://localhost:34000/greeting?name=Container`

### Varias instancias (aislamiento)

```bash
docker run -d --name <app>-2 -p 34001:9000 <dockerhub-user>/<app>:1.0
docker run -d --name <app>-3 -p 34002:9000 <dockerhub-user>/<app>:1.0
```

### Comandos útiles

```bash
docker stop <app>-1        # detiene (envía SIGTERM)
docker start <app>-1       # vuelve a iniciar
docker rm -f <app>-1       # elimina el contenedor
docker rmi <dockerhub-user>/<app>:1.0   # elimina la imagen
```

---

## 3. Docker Compose

`compose.yaml`:

```yaml
services:
  web:
    build: .
    container_name: <app>-web
    environment:
      PORT: 9000
    ports:
      - "8087:9000"
    stop_grace_period: 15s
```

```bash
docker compose up -d --build   # construye y levanta
docker compose ps
docker compose logs -f web
docker compose down            # detiene y elimina los contenedores
```

Probar: `http://localhost:8087/greeting?name=Compose`

---

## 4. Publicar en Docker Hub

> El nombre de usuario en el tag debe ir en **minúsculas**.

```bash
docker login
docker tag <dockerhub-user>/<app>:1.0 <dockerhub-user>/<app>:latest
docker push <dockerhub-user>/<app>:1.0
docker push <dockerhub-user>/<app>:latest
```

---

## 5. AWS Academy: crear la instancia EC2

1. Entrar a AWS Academy → **Learner Lab** → **Start Lab**. Esperar a que el círculo quede en verde y dar clic en **AWS**.
2. Descargar la llave: **AWS Details** → **Download PEM** (`labsuser.pem`).
   (Alternativa: crear un *Key pair* propio en EC2 → Key Pairs → Create key pair → formato `.pem`.)
3. Ir a **EC2 → Instances → Launch instances**:
   - **Name:** `<app>-server`
   - **AMI:** Amazon Linux 2023
   - **Instance type:** `t2.micro` / `t3.micro`
   - **Key pair:** `vockey` (corresponde a `labsuser.pem`) o la llave creada
   - **Network settings → Edit → Security group:**

     | Tipo | Puerto | Origen |
     |---|---|---|
     | SSH | 22 | My IP |
     | Custom TCP | 8080 (puerto de la app) | 0.0.0.0/0 (o el rango permitido) |

4. **Launch instance** y esperar el estado `Running` con `2/2 checks passed`.
5. Copiar la **Public IPv4 address** o **Public IPv4 DNS**.

---

## 6. Conexión por SSH

Dar permisos restringidos a la llave (SSH rechaza llaves con permisos abiertos).

**PowerShell (Windows):**

```powershell
icacls .\labsuser.pem /inheritance:r
icacls .\labsuser.pem /grant:r "$($env:USERNAME):(R)"
ssh -i .\labsuser.pem ec2-user@<ip-publica-ec2>
```

**Linux / macOS / Git Bash:**

```bash
chmod 400 labsuser.pem
ssh -i labsuser.pem ec2-user@<ip-publica-ec2>
```

La primera vez responder `yes` para aceptar el fingerprint.

Copiar un archivo (ej. el jar) a la instancia, si se necesita:

```bash
scp -i labsuser.pem target/<app>.jar ec2-user@<ip-publica-ec2>:/home/ec2-user/
```

---

## 7. Desplegar el contenedor en EC2

Dentro de la instancia:

```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user
exit          # salir y volver a conectarse para que aplique el grupo docker
```

Reconectar por SSH y correr la imagen:

```bash
docker pull <dockerhub-user>/<app>:1.0

docker run -d \
  --name <app> \
  --restart unless-stopped \
  --stop-timeout 15 \
  -e PORT=9000 \
  -p 8080:9000 \
  <dockerhub-user>/<app>:1.0

docker ps
docker logs -f <app>
curl "http://localhost:8080/greeting?name=EC2"
```

Probar desde el navegador del PC local:

```
http://<ip-publica-ec2>:8080/greeting?name=AWS
```

Si no responde: revisar que el Security Group tenga el puerto 8080 abierto y que `docker ps` muestre el contenedor `Up`.

---

## 8. Detener y limpiar

```bash
# En la instancia
docker stop <app>          # shutdown ordenado (SIGTERM, espera --stop-timeout)
docker rm <app>
```

En la consola de AWS:

- **EC2 → Instances → Instance state → Stop instance** (se puede volver a iniciar; la IP pública cambia).
- **Terminate instance** para eliminarla definitivamente.
- Al terminar la sesión: **End Lab** en AWS Academy.

---

## Anexo: dos servidores, shutdown seguro y stop

Escenario: **Servidor A** (frontend / gateway, público) llama a **Servidor B** (backend) en otra instancia EC2.

```
Cliente ──HTTP──► EC2-A (puerto 8080, público) ──HTTP IP privada──► EC2-B (puerto 9000, solo desde A)
```

### A.1 Prueba local con Compose

```yaml
services:
  backend:
    image: <dockerhub-user>/<app-backend>:1.0
    environment:
      PORT: 9000
    stop_grace_period: 15s

  frontend:
    image: <dockerhub-user>/<app-frontend>:1.0
    environment:
      PORT: 8080
      BACKEND_URL: http://backend:9000   # nombre del servicio = hostname en la red de compose
    ports:
      - "8080:8080"
    depends_on:
      - backend
    stop_grace_period: 15s
```

```bash
docker compose up -d
curl http://localhost:8080/
docker compose stop frontend backend   # detiene primero el que recibe tráfico
docker compose down
```

### A.2 Dos instancias EC2

1. Lanzar **dos** instancias (sección 5): `server-a` y `server-b`, con la misma llave.
2. Security groups:

   | Instancia | Tipo | Puerto | Origen |
   |---|---|---|---|
   | A | SSH | 22 | My IP |
   | A | Custom TCP | 8080 | 0.0.0.0/0 |
   | B | SSH | 22 | My IP |
   | B | Custom TCP | 9000 | Security group de A (o IP privada de A `/32`) |

3. Anotar la **Private IPv4** de B (ej. `172.31.x.x`).
4. Instalar Docker en ambas (sección 7).

**En B (backend):**

```bash
docker run -d --name backend --restart unless-stopped --stop-timeout 15 \
  -e PORT=9000 -p 9000:9000 <dockerhub-user>/<app-backend>:1.0
```

**En A (frontend):**

```bash
curl http://<ip-privada-b>:9000/     # verificar conectividad A → B

docker run -d --name frontend --restart unless-stopped --stop-timeout 15 \
  -e PORT=8080 -e BACKEND_URL=http://<ip-privada-b>:9000 \
  -p 8080:8080 <dockerhub-user>/<app-frontend>:1.0
```

Probar: `http://<ip-publica-a>:8080/`

### A.3 Shutdown seguro (graceful)

La idea: al recibir `SIGTERM` el servidor **deja de aceptar conexiones nuevas**, **termina las peticiones en curso** y luego cierra.

- **Docker:** usar `ENTRYPOINT` en forma exec (`["java","-jar","app.jar"]`) para que la JVM sea PID 1 y reciba la señal. `docker stop -t 15` / `--stop-timeout 15` da 15 s antes de forzar `SIGKILL`.
- **Spring Boot** (`application.properties`):

  ```properties
  server.shutdown=graceful
  spring.lifecycle.timeout-per-shutdown-phase=10s
  ```

- **Servidor Java propio** (como el framework de LAB07):

  ```java
  Runtime.getRuntime().addShutdownHook(new Thread(() -> {
      server.stop();                              // cierra el ServerSocket: no más conexiones
      pool.shutdown();                            // no acepta tareas nuevas
      try {
          if (!pool.awaitTermination(10, TimeUnit.SECONDS)) {
              pool.shutdownNow();                 // fuerza si se pasa del tiempo
          }
      } catch (InterruptedException e) {
          pool.shutdownNow();
          Thread.currentThread().interrupt();
      }
      System.out.println("Shutdown complete.");
  }));
  ```

Verificar que funciona:

```bash
curl "http://localhost:8080/slow?seconds=5" &   # petición en curso
docker stop frontend                             # debe responder igual antes de cerrar
docker logs frontend                             # buscar "Shutdown complete."
```

### A.4 Stop en orden

Detener primero el servidor que recibe tráfico y luego el que depende de él:

```bash
# En A
docker stop frontend
# En B
docker stop backend
```

Detener las instancias desde la consola (**Instance state → Stop instance**) o con AWS CLI usando las credenciales de **AWS Details → AWS CLI**:

```bash
aws ec2 stop-instances  --instance-ids <id-instancia-a> <id-instancia-b>
aws ec2 start-instances --instance-ids <id-instancia-a> <id-instancia-b>   # para volver a levantarlas
```

> Al volver a iniciar las instancias la IP pública cambia; la IP privada de B se mantiene, así que `BACKEND_URL` sigue siendo válido. Gracias a `--restart unless-stopped`, los contenedores que estaban corriendo arrancan solos con Docker (los detenidos con `docker stop` no, y hay que usar `docker start <nombre>`).
