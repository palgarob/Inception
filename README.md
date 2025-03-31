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

Got it! Let’s break everything down **step by step** in a way that makes sense, even if you're completely new to web servers, proxies, and encryption. I'll explain **each concept from scratch** with real-world examples so you can fully grasp them.

---

# **What is a Web Server?**
### Think of it as a Restaurant 📡  
A **web server** is a special computer that **stores websites** and **sends them** to people who want to visit them.

#### **Analogy** 🍽️:
Imagine you go to a restaurant:
1. You **order** food from the waiter.
2. The **kitchen** prepares your meal.
3. The **waiter brings it back** to your table.

Now, replace the restaurant with a web server:
1. You **enter a website address** (e.g., `www.example.com`).
2. The web server **finds the website files** (HTML, images, CSS).
3. It **sends them back** to your browser so you can see the webpage.

#### **Example**:
- **NGINX and Apache** are popular web servers.
- They store and deliver **HTML, images, CSS, JavaScript** to visitors.

---

# **What is a Proxy?**
### Think of it as a Translator 🌍  
A **proxy** is a middleman between your computer and the internet.

#### **Analogy** 🔄:
Imagine you’re in **France**, but you only speak **English**. You hire a **translator** to communicate with French speakers:
1. You **ask your translator** to tell the waiter: "I want pizza."
2. The translator **talks to the waiter** in French.
3. The waiter gives a response in **French**, and your translator tells you in **English**.

Similarly, a **proxy server**:
1. **Receives your request** for a website.
2. **Forwards it** to the actual website.
3. **Sends the response back** to you.

> 🔹 **Regular Proxy**: Helps clients browse the internet anonymously.  
> 🔹 **Reverse Proxy**: Helps servers manage requests efficiently.

---

# **What is a Reverse Proxy?**
### Think of it as a Security Guard 🚧  
A **reverse proxy** stands in front of your web server and **filters requests before they reach the actual server**.

#### **Analogy** 🏢:
Imagine you go to a **big office building**:
- The **security guard** at the entrance checks visitors.
- He **directs them** to the correct office inside the building.
- If someone dangerous arrives, he **blocks them**.

Similarly, a **reverse proxy** like NGINX:
- Receives **all website requests** first.
- **Filters out bad traffic** (hackers, bots).
- **Decides where to send requests** (to a web server or another service).

#### **Why Use a Reverse Proxy?**
✅ **Security** – Protects the real server from attacks.  
✅ **Load Balancing** – Distributes traffic across multiple servers.  
✅ **Caching** – Stores pages to serve them faster.  

---

# **What is a Load Balancer?**
### Think of it as a Traffic Cop 🚦  
A **load balancer** is a system that **splits incoming traffic** between multiple web servers.

#### **Analogy** 🛣️:
Imagine a **highway** with heavy traffic:
- A **traffic cop** directs cars to different lanes.
- This prevents **one lane** from getting **too crowded**.
- Cars reach their destination **faster**.

Similarly, a **load balancer**:
- **Distributes requests** to multiple servers.
- Prevents **one server from overloading**.
- Improves **website speed and reliability**.

> 🔹 Example: If `www.example.com` is too busy, the load balancer sends some visitors to `server1`, others to `server2`, etc.

---

# **What is Caching?**
### Think of it as a Fridge for Web Pages ❄️  
A **cache** is a storage that **saves frequently used data** so it can be accessed faster.

#### **Analogy** 🍕:
- Imagine you love pizza, but making it from scratch **takes an hour**.
- Instead, you **make a big batch** and store some in the fridge.
- Next time, you **just reheat it** (faster than making it from scratch).

Similarly, **HTTP caching**:
- Saves **copies of web pages**.
- Serves them instantly **without reloading from the server**.
- Makes websites **load faster**.

> 🔹 **NGINX can cache pages** so it doesn’t have to fetch them from WordPress every time.

---

# **What is TLS? (Transport Layer Security)**
### Think of it as a Lock on Your Mailbox 🔒  
TLS is a **security protocol** that encrypts data between you and a website.

#### **Analogy** 📬:
- Imagine sending **a love letter** 💌.
- If you use **an open envelope**, **anyone can read it**.
- If you **lock it in a box** and only the receiver has the key, **only they can open it**.

Similarly, **TLS encrypts data** so:
- Hackers **can’t see your passwords**.
- Personal data (credit cards, emails) **stay private**.
- Websites use **HTTPS** instead of HTTP (secure version).

> 🔹 **TLS 1.2 and 1.3** are modern versions used by secure websites.

---

# **How Does Everything Fit Together in Your Project?**
You will **build a system where NGINX acts as a reverse proxy**:
1. **NGINX (Reverse Proxy)**  
   - **Receives all requests** from users.
   - **Encrypts data with TLS** (HTTPS).
   - **Forwards requests** to WordPress.

2. **WordPress (PHP-FPM)**  
   - Runs inside **a container**.
   - Retrieves website content **from MariaDB**.
   - Sends the **generated webpage back to NGINX**.

3. **MariaDB (Database)**  
   - Stores **WordPress posts, users, and settings**.
   - Responds **only to WordPress** (not directly to users).

> **Result**: Visitors connect to `https://yourwebsite.com`, and NGINX securely handles everything.

---

# **Final Recap**
| Term | Real-World Example | Purpose |
|------|--------------------|---------|
| **Web Server** | A restaurant kitchen | Serves websites to users |
| **Proxy** | A translator | Passes requests between users and servers |
| **Reverse Proxy** | A security guard | Protects and forwards traffic to backend services |
| **Load Balancer** | A traffic cop | Spreads traffic across multiple servers |
| **Caching** | Storing pizza in a fridge | Saves web pages for faster access |
| **TLS** | A locked mailbox | Encrypts data for security |

---

# **What’s Next?**
Now that you understand these concepts, the next step is to **apply them in your Docker setup**:
- Set up **NGINX as a reverse proxy**.
- **Enable TLS** for secure HTTPS.
- **Configure caching** to speed up WordPress.

Would you like **hands-on examples** of how to set up each part? 🚀
