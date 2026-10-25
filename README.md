# whmphp2shell
Solve PHP2Shell in WHM for PHP7.3
Integrated with PHP MultiPHP 
# Using Ondřej Surý work 
SO MANY THANKS TO YOU!!!!! https://launchpad.net/~ondrej/+archive/ubuntu/php

(If you want to do it for PHP5.6 you have to prepare a dockerfile for PHP5.6)  

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
