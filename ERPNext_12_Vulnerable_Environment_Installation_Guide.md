# ERPNext 12 Vulnerable Environment Installation Guide

> ⚠️ Note: This guide is for ERPNext 12.18.0. All operations should be performed in an isolated test environment.

---

## 1. Docker Installation

### 1.1 Uninstall Old Versions (If Present)

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

### 1.2 Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

### 1.3 Add Docker Official GPG Key

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### 1.4 Add Docker Stable Repository

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 1.5 Install Docker Engine and Compose V2 Plugin

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 1.6 Add Current User to Docker Group

> ⚠️ Note: You need to log out and log back in for this to take effect.

```bash
sudo usermod -aG docker $USER
```

### 1.7 Verify Installation

```bash
docker --version
docker compose version
```

---

## 2. Clone Project and Create Required Directories and Files

```bash
git clone https://github.com/shridarpatil/erpnext-docker.git
cd erpnext-docker
mkdir -p data/redis data/mariadb sites
```

---

## 3. Create Environment Variable File

```bash
cat > env_file << 'EOF'
MYSQL_ROOT_PASSWORD=admin
DB_HOST=db
DB_PORT=3306
EOF
```

---

## 4. Create Custom MariaDB Configuration File

MariaDB 10.6 uses `utf8mb4` by default. We create a minimal `my.cnf` to override the defaults:

```bash
cat > my.cnf << 'EOF'
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
max_allowed_packet = 256M
innodb_buffer_pool_size = 512M
innodb_log_file_size = 128M
EOF
```

---

## 5. Modified docker-compose.yml

```yaml
services:
  # ==================== Database Service ====================
  db:
    image: mariadb:10.6
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-admin}
    volumes:
      - ./data/mariadb:/var/lib/mysql
      - ./my.cnf:/etc/mysql/conf.d/my.cnf:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD:-admin}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ==================== Redis Service ====================
  redis:
    image: redis:5.0.5-alpine
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - ./data/redis:/data

  # ==================== ERPNext Main Application ====================
  web-app:
    image: shridh0r/erpnext:v12
    restart: unless-stopped
    ports:
      - "8005:8000"
    volumes:
      - ./sites:/home/frappe/frappe-bench/sites
    depends_on:
      redis:
        condition: service_started
      db:
        condition: service_healthy
    env_file:
      - env_file
    command: sh -c "cd /home/frappe/frappe-bench && /home/frappe/.local/bin/bench start --no-dev"

  # ==================== Background Workers ====================
  default-worker:
    image: shridh0r/erpnext:v12
    restart: unless-stopped
    volumes:
      - ./sites:/home/frappe/frappe-bench/sites
    depends_on:
      - web-app
    command: sh -c "cd /home/frappe/frappe-bench && /home/frappe/.local/bin/bench worker --queue default"

  long-worker:
    image: shridh0r/erpnext:v12
    restart: unless-stopped
    volumes:
      - ./sites:/home/frappe/frappe-bench/sites
    depends_on:
      - web-app
    command: sh -c "cd /home/frappe/frappe-bench && /home/frappe/.local/bin/bench worker --queue long"

  short-worker:
    image: shridh0r/erpnext:v12
    restart: unless-stopped
    volumes:
      - ./sites:/home/frappe/frappe-bench/sites
    depends_on:
      - web-app
    command: sh -c "cd /home/frappe/frappe-bench && /home/frappe/.local/bin/bench worker --queue short"

  # ==================== Scheduler ====================
  scheduler:
    image: shridh0r/erpnext:v12
    restart: unless-stopped
    volumes:
      - ./sites:/home/frappe/frappe-bench/sites
    depends_on:
      - web-app
    command: sh -c "cd /home/frappe/frappe-bench && /home/frappe/.local/bin/bench schedule"

  # ==================== Socket.IO ====================
  socketio:
    image: shridh0r/erpnext:v12
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - ./sites:/home/frappe/frappe-bench/sites
    depends_on:
      - web-app
    command: sh -c "cd /home/frappe/frappe-bench && /usr/bin/node /home/frappe/frappe-bench/apps/frappe/socketio.js"
```

---

## 6. Deploy Docker

```bash
docker compose up -d
```

---

## 7. Create Site and Administrator Inside the Container

> ⚠️ The following operations must be executed inside the container (not on the host machine).

### 7.1 Enter the web-app Container

```bash
docker exec -it erpnext-docker-web-app-1 bash
```

> ⚠️ Note: The following operations are now inside the container and must be executed within the Python virtual environment.

### 7.2 Activate the Frappe Bench Virtual Environment

```bash
source /home/frappe/frappe-bench/env/bin/activate
```

### 7.3 Optional Commands (Execute as Needed)

```bash
bench reinstall
bench build
deactivate
```

### 7.4 Alternative Solution (If the Above Methods Do Not Work)

> ⚠️ Note: The following commands are an alternative solution. Try them if the main workflow is unavailable.

```bash
# Activate the Frappe Bench virtual environment
source /home/frappe/frappe-bench/env/bin/activate

# Create a site (database name and site name should be consistent)
bench new-site site1.local --mariadb-root-password "Str0ngP@ssw0rd!" --admin-password "Test@12345"

# Install the ERPNext app for this site
bench --site site1.local install-app erpnext

# Wait for the app installation to complete and exit the virtual environment
deactivate
```

### 7.5 Exit the Container

```bash
exit
```
