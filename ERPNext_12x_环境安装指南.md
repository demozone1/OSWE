# ERPNext 12.x 环境安装指南

> ⚠️ 注意：本指南针对 ERPNext 12.x 版本（推荐 12.18.0），所有操作均在ubuntu虚拟机中进行。

---

## 一、Docker 安装

### 1.1 卸载旧版本（如果存在）

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

### 1.2 安装依赖

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

### 1.3 添加 Docker 官方 GPG 密钥

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### 1.4 添加 Docker 稳定版仓库

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 1.5 安装 Docker Engine 及 Compose V2 插件

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 1.6 将当前用户加入 docker 组

> ⚠️ 注意：执行后需重新登录才能生效。

```bash
sudo usermod -aG docker $USER
```

### 1.7 验证安装

```bash
docker --version
docker compose version
```

---

## 二、克隆项目并构建必要目录和文件

```bash
git clone https://github.com/shridarpatil/erpnext-docker.git
cd erpnext-docker
mkdir -p data/redis data/mariadb sites
```

---

## 三、创建环境变量文件

```bash
cat > env_file << 'EOF'
MYSQL_ROOT_PASSWORD=admin
DB_HOST=db
DB_PORT=3306
EOF
```

---

## 四、创建 MariaDB 自定义配置文件

MariaDB 10.6 默认使用 `utf8mb4`，我们创建一个最小化的 `my.cnf` 来覆盖默认值：

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

## 五、修改后的 docker-compose.yml

```yaml
services:
  # ==================== 数据库服务 ====================
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

  # ==================== Redis 服务 ====================
  redis:
    image: redis:5.0.5-alpine
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - ./data/redis:/data

  # ==================== ERPNext 主应用 ====================
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

  # ==================== 后台 Worker ====================
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

  # ==================== 调度器 ====================
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

## 六、部署 Docker

```bash
docker compose up -d
```

---

## 七、在容器内创建站点与管理员

> ⚠️ 以下操作需要在容器内部执行（非宿主机）。

### 7.1 进入 web-app 容器

```bash
docker exec -it erpnext-docker-web-app-1 bash
```

> ⚠️ 注意：以下操作已进入容器内部，且需要在 Python 虚拟环境下执行。

### 7.2 激活 Frappe Bench 的虚拟环境

```bash
source /home/frappe/frappe-bench/env/bin/activate
```

### 7.3 可选命令（按需执行）

```bash
# bench reinstall
# bench build
deactivate
```

### 7.4 备用方案（如上述方法无效）

> ⚠️ 注意：以下命令为备用方案，如主流程不可用可尝试执行。

```bash
# 激活 Frappe Bench 的虚拟环境
# source /home/frappe/frappe-bench/env/bin/activate

# 创建站点（数据库名和站点名保持一致）
# bench new-site site1.local --mariadb-root-password "Str0ngP@ssw0rd!" --admin-password "Test@12345"

# 安装 ERPNext 应用到该站点
# bench --site site1.local install-app erpnext

# 等待应用安装完成并退出虚拟环境
deactivate
```

### 7.5 退出容器

```bash
exit
```
