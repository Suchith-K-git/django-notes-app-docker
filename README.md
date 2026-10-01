# Django Notes Application – Dockerized Deployment

## Project Overview

This project demonstrates the deployment of a **Django Notes Application** using Docker and Docker Compose.

The application consists of three main services:

* **Django** – Application server
* **MySQL** – Database server
* **Nginx** – Reverse proxy server

Docker Compose is used to build, configure, network, and manage all the services.

The project also implements **MySQL healthchecks and service dependencies** so that Django starts only after MySQL is healthy, and Nginx starts after Django is available.

---

## Architecture

```text
                         User
                           |
                           | HTTP :80
                           ↓
                    +-------------+
                    |    Nginx    |
                    | Reverse     |
                    | Proxy       |
                    +-------------+
                           |
                           | :8000
                           ↓
                    +-------------+
                    |   Django    |
                    |  Gunicorn   |
                    +-------------+
                           |
                           | :3306
                           ↓
                    +-------------+
                    |    MySQL    |
                    |   test_db   |
                    +-------------+
                           |
                           ↓
                    Docker Volume
                    mysql-data
```

---

## Technologies Used

* Python
* Django
* Gunicorn
* MySQL
* Nginx
* Docker
* Docker Compose
* Linux
* Docker Volumes
* Docker Networking
* Healthchecks
* Environment Variables

---

## Project Features

* Django application containerization
* MySQL database containerization
* Nginx reverse proxy configuration
* Gunicorn application server
* Docker Compose orchestration
* Persistent MySQL data using Docker volumes
* Custom Docker network
* MySQL healthcheck
* Service dependency management
* Automatic Django database migrations
* Environment-based database configuration

---

# Project Structure

```text
django-notes-app/
│
├── backend/
│   ├── manage.py
│   ├── notesapp/
│   │   ├── settings.py
│   │   ├── urls.py
│   │   ├── wsgi.py
│   │   └── ...
│   │
│   └── ...
│
├── nginx/
│   ├── Dockerfile
│   └── nginx.conf
│
├── Dockerfile
├── docker-compose.yml
├── .env
├── .gitignore
└── README.md
```

---

# Prerequisites

Make sure the system has:

* Docker
* Docker Compose
* Git

Check Docker:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Check Git:

```bash
git --version
```

---

# Step 1 – Clone the Repository

Clone the GitHub repository:

```bash
git clone <YOUR-GITHUB-REPOSITORY-URL>
```

Move into the project directory:

```bash
cd django-notes-app
```

---

# Step 2 – Create the Environment File

Create the `.env` file:

```bash
nano .env
```

Add:

```env
DB_NAME=test_db
DB_USER=root
DB_PASSWORD=root
DB_PORT=3306
DB_HOST=mysql
```

### Important

The database host must be:

```text
mysql
```

because `mysql` is the Docker Compose service name.

Do not use:

```text
localhost
```

or:

```text
db_cont
```

---

# Step 3 – Dockerfile for Django

The Django application uses a Dockerfile to create the application image.

Example:

```dockerfile
FROM python:3.9

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

WORKDIR /app/backend

EXPOSE 8000

CMD ["gunicorn", "notesapp.wsgi", "--bind", "0.0.0.0:8000"]
```

---

# Step 4 – Nginx Dockerfile

Create:

```text
nginx/Dockerfile
```

Example:

```dockerfile
FROM nginx:latest

COPY nginx.conf /etc/nginx/conf.d/default.conf
```

---

# Step 5 – Nginx Configuration

Create:

```text
nginx/nginx.conf
```

Use:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://django:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Here:

```text
django:8000
```

refers to the Django service inside the Docker network.

---

# Step 6 – Docker Compose Configuration

The complete `docker-compose.yml`:

```yaml
services:

  mysql:
    image: mysql
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: test_db
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - notes-app
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: always

  django:
    build:
      context: .
    container_name: django_cont
    command: sh -c "python manage.py migrate --no-input && gunicorn notesapp.wsgi --bind 0.0.0.0:8000"
    ports:
      - "8000:8000"
    env_file:
      - ".env"
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - notes-app
    restart: always

  nginx:
    build:
      context: ./nginx
    container_name: nginx
    ports:
      - "80:80"
    depends_on:
      django:
        condition: service_started
    networks:
      - notes-app
    restart: always

volumes:
  mysql-data:

networks:
  notes-app:
```

---

# Step 7 – Build the Docker Images

Run:

```bash
docker compose build
```

This builds:

```text
Django image
Nginx image
```

MySQL uses the official MySQL image, so it does not need to be built.

---

# Step 8 – Start the Application

Run:

```bash
docker compose up -d
```

Docker Compose will start the services according to their dependencies.

The startup flow is:

```text
MySQL
   ↓
Healthcheck
   ↓
MySQL Healthy
   ↓
Django
   ↓
Nginx
```

---

# Step 9 – Check Running Containers

Run:

```bash
docker ps
```

Expected output should show:

```text
mysql          Up (healthy)
django_cont    Up
nginx          Up
```

---

# Step 10 – Check MySQL Health

Run:

```bash
docker inspect mysql --format='{{json .State.Health}}'
```

You should see:

```text
"Status":"healthy"
```

You can also use:

```bash
docker ps
```

and check the `STATUS` column.

---

# Step 11 – Check Django Logs

Run:

```bash
docker logs django_cont
```

You should see Gunicorn starting and listening on:

```text
0.0.0.0:8000
```

---

# Step 12 – Check Nginx Logs

Run:

```bash
docker logs nginx
```

---

# Step 13 – Access the Application

Because Nginx is mapped to port 80:

```text
http://<SERVER-IP>
```

For example:

```text
http://13.234.XX.XX
```

Nginx receives the request on port `80` and forwards it to:

```text
Django → port 8000
```

---

# Step 14 – Direct Django Access

Django is also exposed on port `8000`.

You can access:

```text
http://<SERVER-IP>:8000
```

However, in the intended architecture, users access the application through:

```text
Port 80 → Nginx → Django
```

---

# Step 15 – Check Docker Network

Run:

```bash
docker network ls
```

Find the application network.

Inspect it:

```bash
docker network inspect django-notes-app_notes-app
```

The containers should be connected to the same network:

```text
mysql
django_cont
nginx
```

---

# Step 16 – Check the MySQL Volume

List volumes:

```bash
docker volume ls
```

Inspect the volume:

```bash
docker volume inspect django-notes-app_mysql-data
```

The volume stores MySQL data outside the container filesystem.

Therefore, database data can persist when the MySQL container is recreated.

---

# Step 17 – Stop the Application

To stop the application:

```bash
docker compose stop
```

---

# Step 18 – Start Again

```bash
docker compose start
```

---

# Step 19 – Stop and Remove Containers

```bash
docker compose down
```

This removes the containers and network but keeps the named volume by default.

---

# Step 20 – Remove Containers and Volumes

If you want to completely remove the containers and database volume:

```bash
docker compose down -v
```

**Warning:** This removes the MySQL volume and therefore deletes the stored database data.

---

# Troubleshooting

## 1. Check all containers

```bash
docker ps -a
```

---

## 2. Check Django logs

```bash
docker logs django_cont
```

---

## 3. Check MySQL logs

```bash
docker logs mysql
```

---

## 4. Check Nginx logs

```bash
docker logs nginx
```

---

## 5. Check Django health

```bash
docker inspect django_cont --format='{{json .State.Health}}'
```

---

## 6. Check MySQL health

```bash
docker inspect mysql --format='{{json .State.Health}}'
```

---

## 7. Rebuild everything

If you make changes to the Dockerfile or application:

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

---

# Service Communication

The containers communicate using Docker's internal network.

```text
Nginx
  |
  | django:8000
  ↓
Django
  |
  | mysql:3306
  ↓
MySQL
```

The important point is that containers should communicate using **Docker service names**, not `localhost`.

For example:

```env
DB_HOST=mysql
```

and Nginx:

```nginx
proxy_pass http://django:8000;
```

---

# Service Dependencies

The project uses Docker Compose dependency conditions.

Django waits for MySQL:

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

Nginx waits for Django to start:

```yaml
depends_on:
  django:
    condition: service_started
```

This creates the following startup sequence:

```text
        MySQL
          |
          | Healthcheck
          ↓
       Healthy
          |
          ↓
        Django
          |
          | Started
          ↓
        Nginx
```

---

# Healthcheck

The MySQL healthcheck uses:

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 30s
```

This checks whether MySQL is responding before Docker considers the MySQL service healthy.

---

# Persistent Storage

MySQL uses a Docker named volume:

```yaml
volumes:
  - mysql-data:/var/lib/mysql
```

This means MySQL data is stored in:

```text
mysql-data
```

instead of only inside the container.

---

# Reverse Proxy

Nginx acts as a reverse proxy.

The request flow is:

```text
Client
  |
  | HTTP :80
  ↓
Nginx
  |
  | Proxy
  ↓
Django :8000
  |
  ↓
MySQL :3306
```

Nginx configuration:

```nginx
proxy_pass http://django:8000;
```

---

# Key Docker Commands

### Build

```bash
docker compose build
```

### Start

```bash
docker compose up -d
```

### Stop

```bash
docker compose stop
```

### Remove containers

```bash
docker compose down
```

### View containers

```bash
docker ps
```

### View all containers

```bash
docker ps -a
```

### View logs

```bash
docker logs django_cont
docker logs mysql
docker logs nginx
```

### Follow logs

```bash
docker logs -f django_cont
```

### Rebuild

```bash
docker compose up -d --build
```

---

# Skills Demonstrated

This project demonstrates practical knowledge of:

* Docker
* Docker Compose
* Dockerfile
* Containerization
* Django deployment
* Gunicorn
* Nginx
* Reverse Proxy
* MySQL
* Docker Networking
* Docker Volumes
* Healthchecks
* Service Dependencies
* Environment Variables
* Linux
* Application Deployment
* Troubleshooting

---

# Conclusion

This project demonstrates how a Django application can be containerized and deployed using Docker Compose with a separate application server, database server, and reverse proxy. It also demonstrates persistent storage, container networking, healthchecks, service dependencies, automated migrations, and production-style application startup.
