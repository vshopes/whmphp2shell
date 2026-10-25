# whmphp2shell
Solve PHP2Shell in WHM for PHP7.3
# Guide: Secure Isolation of Legacy PHP 7.3 (Anti-PHP2Shell) Using Rootless Podman on WHM / AlmaLinux 9

This guide explains how to safely isolate a legacy version of PHP 7.3 (patched against security vulnerabilities like PHP2Shell based on Ondřej Surý's historical repositories) inside a **Rootless Podman (unprivileged)** container for a specific cPanel account.

The solution integrates seamlessly with WHM's native **MultiPHP Manager**: if the user switches the account to PHP 8.2 (or any other modern version), it will use the server's native environment. If they switch back to PHP 7.3, Apache will automatically route the traffic into the secure isolated container without affecting any other hosted accounts.

---

## PREREQUISITES
* A server running **AlmaLinux 9** with **WHM/cPanel** installed.
* **Podman** and **docker-compose** installed, configured, and running.
* The username of the targeted cPanel account (referred to as `ACCOUNT_NAME` throughout this guide).

---

## STEP 1: Enable the Rootless Environment for the cPanel User

Because cPanel accounts are heavily jailed, the initial activation of the user's Podman socket and sub-ID mapping ranges must be configured from the **`root`** shell of the server.

Run the following commands as **`root`**:

```bash
# 1. Enable user persistence (Linger) so containers keep running after the SSH session closes
loginctl enable-linger ACCOUNT_NAME

# 2. Assign secondary UID and GID ranges required to unpack the Debian base filesystem inside the user namespace
echo "ACCOUNT_NAME:200000:65536" >> /etc/subuid
echo "ACCOUNT_NAME:200000:65536" >> /etc/subgid

# 3. Force Podman to migrate and pick up the newly defined ID mappings
machinectl shell ACCOUNT_NAME@.host /usr/bin/podman system migrate

# 4. Enable and start the private Podman systemd user socket
machinectl shell ACCOUNT_NAME@.host /usr/bin/systemctl --user enable --now podman.socket
```

To verify that the socket was created successfully with the correct user ownership, run:
```bash
ls -la /run/user/\$(id -u ACCOUNT_NAME)/podman/podman.sock
# It should return something like: srw-rw---- 1 ACCOUNT_NAME ACCOUNT_NAME ... podman.sock
```

---

## STEP 2: Configure the Docker Compose Environment for the User

1. Switch to the cPanel user session (or log in directly via SSH using their credentials):
   ```bash
   su - ACCOUNT_NAME
   ```

2. Create a project directory **outside** of the public web area (`public_html`) to safeguard configuration blueprints:
   ```bash
   mkdir -p ~/docker-php73
   cd ~/docker-php73
   ```

3. Permanently point the `docker-compose` CLI client to talk to the Rootless Podman user socket instead of the global system socket. Append this to the end of the user's `~/.bashrc` file:
   ```bash
   echo 'export DOCKER_HOST="unix:///run/user/'\$(id -u)'/podman/podman.sock"' >> ~/.bashrc
   source ~/.bashrc
   ```

---

## STEP 3: Create Container Configuration Files

Inside the `~/docker-php73` directory, create the following two files using the cPanel user account:

### 1. `Dockerfile`
This compiles a Debian 12 Slim environment injecting Ondřej Surý's archive repositories. It forces the PHP-FPM master and worker pools to run as `root` internally. This is fully secure because the container's `root` map resolves directly to the unprivileged `ACCOUNT_NAME` UID on the host, preventing all standard filesystem permission issues.

```dockerfile
FROM debian:12-slim

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    lsb-release \
    && curl -sSLo /usr/share/keyrings/deb.sury.org-php.gpg https://sury.org \
    && echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://sury.org \$(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list \
    && apt-get update

RUN apt-get install -y --no-install-recommends \
    php7.3-cli php7.3-fpm php7.3-common php7.3-dev php-pear \
    php7.3-bcmath php7.3-bz2 php7.3-calendar php7.3-curl php7.3-dba \
    php7.3-enchant php7.3-gd php7.3-gmp php7.3-imap php7.3-intl \
    php7.3-mbstring php7.3-mysql php7.3-odbc php7.3-opcache php7.3-pgsql \
    php7.3-soap php7.3-xml php7.3-zip php7.3-tidy php7.3-imagick \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Configure PHP-FPM to listen to open TCP network connections on port 9000
RUN sed -i 's|listen = /run/php/php7.3-fpm.sock|listen = 0.0.0.0:9000|' /etc/php/7.3/fpm/pool.d/www.conf

# Force the 'www' pool worker to run as root to match host file ownership under Rootless mapping
RUN sed -i 's/user = www-data/user = root/' /etc/php/7.3/fpm/pool.d/www.conf \
    && sed -i 's/group = www-data/group = root/' /etc/php/7.3/fpm/pool.d/www.conf

RUN echo "pm.max_requests = 500" >> /etc/php/7.3/fpm/pool.d/www.conf
RUN mkdir -p /run/php
EXPOSE 9000

WORKDIR /var/www/html

# Launch PHP-FPM with the allow-root flag (-R) mandatory for modified pools
CMD ["/usr/sbin/php-fpm7.3", "-F", "-R"]
```

### 2. `docker-compose.yml`
Handles the local networking mapping and mounts the true live cPanel web path directly into the container workspace:

```yaml
services:
  php-app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app_php_pruebas
    restart: always
    ports:
      # Maps FastCGI to host localhost on an exclusive custom port (e.g., 9073)
      - "127.0.0.1:9073:9000"
    volumes:
      # Mounts live cPanel public files directly to the container working directory
      - /home/ACCOUNT_NAME/public_html:/var/www/html:rw
```

---

## STEP 4: Deploy the Container

While in the `~/docker-php73` directory as the cPanel user, spin up the container infrastructure in detached mode:

```bash
docker-compose up --build -d
```

Verify that the micro-environment is up and running:
```bash
docker ps
# You should see the container running with an "Up" status and port 127.0.0.1:9073 mapped.
```

---

## STEP 5: Dynamic Integration with WHM Server Apache

To make Apache dynamically switch between environments depending on the MultiPHP interface choices, we inject an absolute fallback rule into cPanel's local account overrides. The directive reads the active `.htaccess` file on the fly and proxies traffic to Podman *only* if the legacy marker is present.

Log back in as **`root`** on the server terminal and complete the hooks:

1. Create cPanel's local directory path framework for custom Apache user includes (the secure SSL section):
   ```bash
   mkdir -p /etc/apache2/conf.d/userdata/ssl/2_4/ACCOUNT_NAME/
   ```

2. Open the user configuration file:
   ```bash
   nano /etc/apache2/conf.d/userdata/ssl/2_4/ACCOUNT_NAME/container_php73.conf
   ```

3. Paste the following conditional block:
   ```apache
   <Directory "/home/ACCOUNT_NAME/public_html">
       <IfModule mod_rewrite.c>
           RewriteEngine On

           # Condition 1: Only intercept if the requested resource is a script ending in .php
           RewriteCond %{REQUEST_FILENAME} \.php\$

           # Condition 2: Examine the raw disk file. If cPanel has set the handler string to ea-php73, trigger proxy.
           # If changed on the panel to a modern version (e.g., ea-php82), the match fails and Apache defaults back.
           RewriteCond expr "file('/home/ACCOUNT_NAME/public_html/.htaccess') =~ /ea-php73/"

           # Route traffic into the custom Podman port translation, feeding the container's short internal path
           RewriteRule ^(.*)\$ fcgi://127.0.0.1:9073/var/www/html/\$1 [P,L]
       </IfModule>
   </Directory>
   ```

4. Tell cPanel to safely compile and combine this new rule block into the main web-server configuration tree (`httpd.conf`), and gracefully restart Apache:
   ```bash
   /usr/local/cpanel/scripts/rebuildhttpdconf
   systemctl restart httpd
   ```

---

## STEP 6: Testing the Integration

1. Drop a standard test file containing the `phpinfo();` function into `/home/ACCOUNT_NAME/public_html/info.php`.
2. Go to **MultiPHP Manager** inside WHM or the account's cPanel interface, and assign **PHP 8.2**. Refresh the site in your browser; it will seamlessly load the server's native modern environment.
3. Switch the profile back to **PHP 7.3**. Refresh the site again. The conditional include will trap the request before the default handlers execute, and the `phpinfo()` layout will display system signatures referencing **Debian** and the unique alfanumeric hash of your isolated container.

1. Cambia a la sesión del usuario de cPanel (o accede directamente por SSH con sus credenciales):
   ```bash
   su - ACCOUNT_NAME
   ```

2. Crea una carpeta para el proyecto **fuera** de la zona pública web para resguardar las configuraciones:
   ```bash
   mkdir -p ~/docker-php73
   cd ~/docker-php73
   ```

3. Redirige permanentemente el cliente `docker-compose` para que hable con el socket Rootless de Podman en lugar del sistema global. Añade al final de tu archivo `~/.bashrc`:
   ```bash
   echo 'export DOCKER_HOST="unix:///run/user/'\$(id -u)'/podman/podman.sock"' >> ~/.bashrc
   source ~/.bashrc
   ```

---

## PASO 3: Creación de los archivos de configuración del Contenedor

Dentro de la carpeta `~/docker-php73`, crea los siguientes archivos utilizando el usuario de la cuenta:

### 1. `Dockerfile`
Este archivo compila un entorno Debian 12 Slim inyectando los repositorios históricos de Ondřej Surý. Forza a los procesos trabajadores (*workers*) de PHP-FPM a ejecutarse como `root` internamente (lo cual es seguro ya que el `root` del contenedor equivale al UID del usuario real en el host debido a Podman Rootless), solucionando los problemas de lectura de archivos.

```dockerfile
FROM debian:12-slim

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    lsb-release \
    && curl -sSLo /usr/share/keyrings/deb.sury.org-php.gpg https://sury.org \
    && echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://sury.org \$(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list \
    && apt-get update

RUN apt-get install -y --no-install-recommends \
    php7.3-cli php7.3-fpm php7.3-common php7.3-dev php-pear \
    php7.3-bcmath php7.3-bz2 php7.3-calendar php7.3-curl php7.3-dba \
    php7.3-enchant php7.3-gd php7.3-gmp php7.3-imap php7.3-intl \
    php7.3-mbstring php7.3-mysql php7.3-odbc php7.3-opcache php7.3-pgsql \
    php7.3-soap php7.3-xml php7.3-zip php7.3-tidy php7.3-imagick \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Configurar PHP-FPM para responder en red interna/externa (Puerto 9000)
RUN sed -i 's|listen = /run/php/php7.3-fpm.sock|listen = 0.0.0.0:9000|' /etc/php/7.3/fpm/pool.d/www.conf

# Forzar al pool 'www' a ejecutarse como root para mitigar colisión de UIDs en el montaje Rootless
RUN sed -i 's/user = www-data/user = root/' /etc/php/7.3/fpm/pool.d/www.conf \
    && sed -i 's/group = www-data/group = root/' /etc/php/7.3/fpm/pool.d/www.conf

RUN echo "pm.max_requests = 500" >> /etc/php/7.3/fpm/pool.d/www.conf
RUN mkdir -p /run/php
EXPOSE 9000

WORKDIR /var/www/html

# Se arranca permitiendo la bandera de root (-R) obligatoria para el pool modificado
CMD ["/usr/sbin/php-fpm7.3", "-F", "-R"]
```

### 2. `docker-compose.yml`
Define el mapeo de red local y monta la carpeta pública real de cPanel en la ruta esperada por el contenedor:

```yaml
services:
  php-app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app_php_pruebas
    restart: always
    ports:
      # Mapea FastCGI al localhost del host en un puerto exclusivo (ej: 9073)
      - "127.0.0.1:9073:9000"
    volumes:
      # Monta la ruta real de cPanel hacia el entorno de trabajo del contenedor
      - /home/ACCOUNT_NAME/public_html:/var/www/html:rw
```

---

## PASO 4: Desplegar el contenedor

Estando dentro de la carpeta `~/docker-php73` con el usuario de la cuenta, levanta el contenedor en segundo plano:

```bash
docker-compose up --build -d
```

Verifica que el contenedor esté corriendo con éxito ejecutando:
```bash
docker ps
# Debería mostrar el contenedor con STATUS "Up" y el mapeo al puerto 127.0.0.1:9073
```

---

## PASO 5: Integración Dinámica con el Servidor Apache (WHM)

Para que los cambios de versión realizados en el panel MultiPHP conmuten de forma automática, crearemos una directiva condicional en los Includes locales de la cuenta. Esta regla lee en tiempo real el archivo `.htaccess` físico modificado por cPanel y desvía la petición al contenedor únicamente si está seleccionada la versión vieja.

Regresa a la terminal de **`root`** y realiza lo siguiente:

1. Crea la ruta de la estructura de configuración personalizada de Apache de cPanel para esa cuenta (sección SSL cifrada):
   ```bash
   mkdir -p /etc/apache2/conf.d/userdata/ssl/2_4/ACCOUNT_NAME/
   ```

2. Crea el archivo de configuración:
   ```bash
   nano /etc/apache2/conf.d/userdata/ssl/2_4/ACCOUNT_NAME/container_php73.conf
   ```

3. Pega exactamente las siguientes directivas de reescritura:
   ```apache
   <Directory "/home/ACCOUNT_NAME/public_html">
       <IfModule mod_rewrite.c>
           RewriteEngine On

           # Condición 1: Solo intercepta si el archivo solicitado es un script .php
           RewriteCond %{REQUEST_FILENAME} \.php\$

           # Condición 2: Examina el .htaccess físico. Si contiene la cadena del handler ea-php73, activa el proxy.
           # Si se cambia en el panel a otra versión (ej: ea-php82), la condición falla y Apache usa el backend nativo.
           RewriteCond expr "file('/home/ACCOUNT_NAME/public_html/.htaccess') =~ /ea-php73/"

           # Redirige el flujo al puerto local de Podman inyectando la ruta interna del contenedor
           RewriteRule ^(.*)\$ fcgi://127.0.0.1:9073/var/www/html/\$1 [P,L]
       </IfModule>
   </Directory>
   ```

4. Ejecuta los scripts de cPanel para reconstruir el archivo maestro de Apache (`httpd.conf`) con la nueva directiva integrada y reinicia el servicio web de forma segura:
   ```bash
   /usr/local/cpanel/scripts/rebuildhttpdconf
   systemctl restart httpd
   ```

---

## PASO 6: Prueba Final

1. Crea un archivo `info.php` básico dentro de `/home/ACCOUNT_NAME/public_html/`.
2. Dirígete al panel de **MultiPHP Manager** en cPanel/WHM y selecciona **PHP 8.2**. Al visitar la web en el navegador, verás que carga la versión nativa normal.
3. Cambia la versión en el panel a **PHP 7.3**. Al refrescar la URL en el navegador, el sistema procesará la condicional y verás en el diseño del `phpinfo()` que el "System" de Linux ahora hace referencia al ID alfanumérico único del contenedor Debian aislado.

