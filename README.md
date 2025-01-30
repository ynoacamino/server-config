# 🚀🔧 Configuración de servidor escalable con Docker

Este proyecto utiliza **contenedores Docker** para gestionar varios servicios, con un **proxy inverso** administrado por **Nginx** para redirigir el tráfico entre los contenedores. Todo esto está diseñado para ser escalable, con **Let's Encrypt** para la gestión automática de certificados SSL 🔒.

## 🏗️ Arquitectura del Proyecto

![Arquitectura del Proyecto](https://ynoa-uploader.ynoacamino.site/uploads/1738270847_Untitled-2024-11-30-1525%20%284%29.png)

La arquitectura está compuesta por varios contenedores Docker organizados según su función. Los principales servicios incluyen:

- **alert-server**: 🛎️ Maneja las alertas del sistema.
- **bot**: 🤖 Un bot automatizado.
- **bridge-code**: 🌉 Servicio de puente entre contenedores.
- **certificates-validator**: 🔑 Valida y renueva certificados SSL.
- **ente**: 🏛️ Gestión de imagenes.
- **inveztiga-pb**: 📊 Servicio de análisis de datos.
- **minio**: ☁️ Almacenamiento compatible con S3.
- **proxy**: 🌐 Nginx como proxy inverso para la redirección de tráfico.
- **secrets**: 🗝️ Gestión de credenciales y datos sensibles.
- **uploader**: 📤 Servicio para la carga de archivos.

## 🔄 Proxy Inverso con Nginx

El servicio de **proxy** es esencial para gestionar las solicitudes HTTP/HTTPS, redirigiéndolas a los contenedores correspondientes. **Nginx** actúa como proxy inverso, manejando el tráfico entrante y redirigiéndolo según el dominio y puerto configurados.

### 🔧 Funcionamiento de Nginx

1. **Redirección de tráfico**: Nginx escucha en los puertos **80 (HTTP)** y **443 (HTTPS)**. Cuando llega una solicitud, Nginx redirige el tráfico al contenedor adecuado según el dominio configurado (ej. `alert-server.ynoacamino.site`).
   
2. **Gestión de Certificados SSL**: Nginx utiliza **Let's Encrypt** para obtener y renovar certificados SSL automáticamente. Las solicitudes HTTP son redirigidas a HTTPS para asegurar las comunicaciones.

3. **Generación dinámica de configuración**: El archivo de configuración de Nginx se genera automáticamente mediante **docker-gen**. Esta herramienta detecta cambios en los contenedores y ajusta la configuración de Nginx sin intervención manual.

### ⚙️ Configuración de Nginx

El contenedor de **proxy** usa el archivo `nginx.tmpl` para generar dinámicamente la configuración de Nginx. Este archivo contiene reglas para cada servicio, utilizando variables de entorno como `VIRTUAL_HOST` y `LETSENCRYPT_HOST`.

Ejemplo de configuración generada:

```nginx
server {
    listen 80;
    server_name {{ .Env.VIRTUAL_HOST }};
    location / {
        proxy_pass http://{{ .Name }}:{{ .Env.VIRTUAL_PORT }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 🔒 Configuración del Proxy

El archivo `docker-compose.yaml` para el servicio **proxy** configura **Nginx** y **docker-gen** para gestionar los contenedores y exponerlos adecuadamente a través del proxy inverso.

Ejemplo de configuración en el archivo `docker-compose.yml` para **proxy**:

```yaml
version: '3'
services:
  proxy:
    image: jwilder/nginx-proxy
    environment:
      - DEFAULT_HOST=default.local
      - VIRTUAL_HOST=*.ynoacamino.site        # 🌐 Los dominios que serán gestionados por Nginx.
      - LETSENCRYPT_HOST=*.ynoacamino.site    # 🔒 Los dominios para los que se generarán los certificados SSL.
      - LETSENCRYPT_EMAIL=admin@ynoacamino.site # 📧 Correo para la gestión de certificados Let's Encrypt.
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro  # 🐳 Permite la comunicación con Docker para la generación dinámica de configuración.
      - ./nginx.tmpl:/etc/nginx/nginx.tmpl:ro  # 📜 Plantilla para la configuración dinámica de Nginx.
    ports:
      - "80:80"  # 🚪 Puerto 80 para tráfico HTTP.
      - "443:443" # 🔒 Puerto 443 para tráfico HTTPS.
    restart: always  # 🔁 Reinicia el contenedor automáticamente si se detiene.
```

### 📡 Variables de Entorno para Exponer los Contenedores

Para que cada contenedor sea accesible a través del **proxy inverso** de Nginx, debes definir ciertas variables de entorno en su configuración `docker-compose.yml`. Las variables más comunes incluyen:

- **VIRTUAL_HOST**: El dominio al que el contenedor debe responder (ej. `alert-server.ynoacamino.site`).
- **VIRTUAL_PORT**: El puerto interno del contenedor al que Nginx redirigirá el tráfico (por ejemplo, `8080`).
- **LETSENCRYPT_HOST**: El dominio para el cual se generará el certificado SSL (por ejemplo, `alert-server.ynoacamino.site`).
- **LETSENCRYPT_EMAIL**: El correo electrónico asociado con el dominio para Let's Encrypt.

Ejemplo de cómo se configuran estas variables en un contenedor, por ejemplo, para el **alert-server**:

```yaml
version: '3'
services:
  alert-server:
    image: alert-server-image
    environment:
      - VIRTUAL_HOST=alert-server.ynoacamino.site  # 🌍 Dominio del contenedor.
      - VIRTUAL_PORT=8080  # 🚪 Puerto interno del contenedor.
      - LETSENCRYPT_HOST=alert-server.ynoacamino.site  # 🔒 Certificado SSL para el dominio.
      - LETSENCRYPT_EMAIL=admin@ynoacamino.site  # 📧 Correo para el certificado SSL.
    expose:
      - "8080"  # 🚪 Expone el puerto interno al contenedor de Nginx.
```

### 💻 Uso Actual en el Servidor

Actualmente, el servidor está desplegado con varios servicios operativos y gestionados por Docker. Para iniciar los servicios:

1. Clonar el repositorio:
   ```bash
   git clone <URL del repositorio> # 🧑‍💻 Clona el proyecto.
   cd <directorio del repositorio> # 📂 Accede al directorio del proyecto.
   ```

2. Levantar los servicios con Docker Compose:
   ```bash
   docker-compose up -d # 🚀 Levanta los servicios en segundo plano.
   ```

3. Verificar el estado de los contenedores:
   ```bash
   docker ps # 📊 Muestra el estado de los contenedores en ejecución.
   ```

Todos los servicios se gestionan a través del **proxy inverso Nginx**, que configura automáticamente **SSL** mediante **Let's Encrypt** y redirige el tráfico de forma eficiente a cada contenedor según su dominio correspondiente. 🌐🔒