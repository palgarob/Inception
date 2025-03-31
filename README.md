# Inception
Inception 42

### Guide to Completing the **Inception** Project (42 School)

This project will introduce you to **Docker** and **Docker Compose** by setting up a small infrastructure with multiple services. Since you're new to Docker, I'll break everything down step by step with explanations and useful resources.

---

## **1. Understanding the Project Requirements**
You need to set up **three services** inside **Docker containers**:
1. **NGINX** (with TLS 1.2 or 1.3)
2. **WordPress** (with `php-fpm`)
3. **MariaDB** (database)

Each service:
- Runs in its own container.
- Uses a **custom Dockerfile**.
- Uses **Docker volumes** for persistent data.
- Connects via a **Docker network**.

> **Forbidden things:**
> - Using prebuilt images (except for Alpine/Debian).
> - Using `network: host` or `--link`.
> - Storing credentials inside Dockerfiles.

---

## **2. Setting Up Your Project Structure**
You need a **specific directory structure**. Follow this:

```
inception/
│── Makefile
│── srcs/
│   │── docker-compose.yml
│   │── .env
│   ├── requirements/
│   │   ├── mariadb/
│   │   │   ├── Dockerfile
│   │   │   ├── conf/
│   │   ├── nginx/
│   │   │   ├── Dockerfile
│   │   │   ├── conf/
│   │   ├── wordpress/
│   │   │   ├── Dockerfile
│   │   │   ├── conf/
│── secrets/
│   │── db_password.txt
│   │── db_root_password.txt
```

- `Makefile`: Automates setup.
- `docker-compose.yml`: Defines and configures your containers.
- `.env`: Stores variables (e.g., database passwords).
- `requirements/`: Contains Dockerfiles and configurations for each service.

---

## **3. Installing Docker & Docker Compose**
Before starting, install Docker and Docker Compose:

### **For Ubuntu:**
```bash
sudo apt update
sudo apt install docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
```

### **For macOS:**
Download and install **Docker Desktop** from [here](https://www.docker.com/products/docker-desktop/).

---

## **4. Creating the Services**
### **4.1 Setting Up MariaDB**
1. **MariaDB is your database for WordPress.**
2. It must:
   - Run in its own container.
   - Store data in a **Docker volume**.
   - Use a **non-root user** for security.

#### **MariaDB Dockerfile (`srcs/requirements/mariadb/Dockerfile`)**
```dockerfile
FROM debian:latest
RUN apt update && apt install -y mariadb-server
COPY ./conf/my.cnf /etc/mysql/my.cnf
COPY ./tools/init_db.sh /docker-entrypoint-initdb.d/
CMD ["mysqld"]
```

#### **Database Initialization Script (`srcs/requirements/mariadb/tools/init_db.sh`)**
```bash
#!/bin/bash
service mysql start

mysql -e "CREATE DATABASE wordpress;"
mysql -e "CREATE USER 'wp_user'@'%' IDENTIFIED BY '${MYSQL_PASSWORD}';"
mysql -e "GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'%';"
mysql -e "FLUSH PRIVILEGES;"
```
> Replace `${MYSQL_PASSWORD}` with an environment variable.

---

### **4.2 Setting Up WordPress**
1. **WordPress needs PHP-FPM** (FastCGI for NGINX).
2. Must connect to MariaDB.
3. Use **Docker volumes** for persistent files.

#### **WordPress Dockerfile (`srcs/requirements/wordpress/Dockerfile`)**
```dockerfile
FROM debian:latest
RUN apt update && apt install -y php php-fpm php-mysql curl
COPY ./conf/www.conf /etc/php/7.4/fpm/pool.d/www.conf
WORKDIR /var/www/html
COPY ./tools/setup.sh .
CMD ["php-fpm", "-F"]
```

#### **Setup Script (`srcs/requirements/wordpress/tools/setup.sh`)**
```bash
#!/bin/bash
cd /var/www/html
curl -O https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
mv wordpress/* .
rm -rf wordpress latest.tar.gz
chown -R www-data:www-data /var/www/html
php-fpm
```

---

### **4.3 Setting Up NGINX**
1. **NGINX is the only public entry point** (port 443 with TLS).
2. **Serves WordPress** through FastCGI (PHP-FPM).
3. Uses **TLS encryption**.

#### **NGINX Dockerfile (`srcs/requirements/nginx/Dockerfile`)**
```dockerfile
FROM alpine:latest
RUN apk update && apk add nginx openssl
COPY ./conf/nginx.conf /etc/nginx/nginx.conf
CMD ["nginx", "-g", "daemon off;"]
```

#### **NGINX Configuration (`srcs/requirements/nginx/conf/nginx.conf`)**
```nginx
server {
    listen 443 ssl;
    server_name ${DOMAIN_NAME};
    
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    location / {
        root /var/www/html;
        index index.php index.html;
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---

## **5. Creating the `docker-compose.yml`**
Now, let's define our services in `docker-compose.yml`:

```yaml
version: '3.8'

services:
  mariadb:
    build: ./requirements/mariadb
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - inception

  wordpress:
    build: ./requirements/wordpress
    environment:
      - WORDPRESS_DB_HOST=mariadb
      - WORDPRESS_DB_USER=${MYSQL_USER}
      - WORDPRESS_DB_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - wp-data:/var/www/html
    depends_on:
      - mariadb
    networks:
      - inception

  nginx:
    build: ./requirements/nginx
    ports:
      - "443:443"
    volumes:
      - wp-data:/var/www/html
    depends_on:
      - wordpress
    networks:
      - inception

volumes:
  db-data:
  wp-data:

networks:
  inception:
```

---

## **6. Automate with Makefile**
Create a `Makefile` to build everything easily:

```make
all:
	docker-compose up --build -d

clean:
	docker-compose down --volumes
	docker system prune -af
```

Now, run:
```bash
make all
```

---

## **7. Accessing Your Site**
1. Open **https://your_login.42.fr** in a browser.
2. It should show the WordPress setup page.

---

## **8. Bonus Ideas**
Once your project works, try:
- **Redis caching** for WordPress.
- **An FTP server** for file uploads.
- **A static website** as an extra service.
- **Adminer** to manage the database.

---

## **9. Useful Resources**
- [Docker Docs](https://docs.docker.com/)
- [NGINX + PHP-FPM](https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/)
- [MariaDB Setup](https://mariadb.com/kb/en/docker-and-mariadb/)

---

## **Conclusion**
This project helps you understand **Docker, Docker Compose, and networking**. If you follow this step-by-step, you'll get a functional setup. Let me know if you need clarifications! 🚀
