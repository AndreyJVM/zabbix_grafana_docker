# Установка и настройка zabbix grafana

### Структура проекта
```text
docker-zabbix/
├── docker-compose.yml
├── .env
├── mysql-init/
└── grafana-provisioning/
```

### Создание структуры
```shell
mkdir -p docker-zabbix/{mysql-init,grafana-provisioning}
touch docker-zabbix/docker-compose.yml docker-zabbix/.env
```

### Файл `.env`

```shell
# Общие настройки
COMPOSE_PROJECT_NAME=monitoring
TZ=Europe/Moscow

# MySQL
MYSQL_ROOT_PASSWORD=SecureRootPassword123
MYSQL_DATABASE=zabbix
MYSQL_USER=zabbix
MYSQL_PASSWORD=ZabbixDBPass123

# Zabbix
ZBX_SERVER_HOST=zabbix-server
ZBX_FRONTEND_HOST=zabbix-web
ZBX_FRONTEND_PORT=8081  # Измененный порт
ZBX_SERVER_PORT=10051
ZBX_AGENT_PORT=10050

# Grafana
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=GrafanaAdminPass123
GF_USERS_ALLOW_SIGN_UP=false
```

### docker-compose.yml
```shell
version: '3.8'

services:
  zabbix-server:
    image: zabbix/zabbix-server-mysql:7.0-latest
    container_name: zabbix-server
    restart: unless-stopped
    ports:
      - "${ZBX_SERVER_PORT}:10051"
    environment:
      - DB_SERVER_HOST=zabbix-mysql
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - TZ=${TZ}
    volumes:
      - zabbix_server_data:/var/lib/zabbix
    depends_on:
      - zabbix-mysql

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:7.0-latest
    container_name: zabbix-web
    restart: unless-stopped
    ports:
      - "${ZBX_FRONTEND_PORT}:8080"
    environment:
      - DB_SERVER_HOST=zabbix-mysql
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - ZBX_SERVER_HOST=${ZBX_SERVER_HOST}
      - PHP_TZ=${TZ}
    depends_on:
      - zabbix-mysql
      - zabbix-server

  zabbix-mysql:
    image: mysql:8.0
    container_name: zabbix-mysql
    restart: unless-stopped
    command: --default-authentication-plugin=mysql_native_password
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - TZ=${TZ}
    volumes:
      - zabbix_mysql_data:/var/lib/mysql
      - ./mysql-init:/docker-entrypoint-initdb.d

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=${GF_SECURITY_ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=${GF_USERS_ALLOW_SIGN_UP}
      - TZ=${TZ}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana-provisioning:/etc/grafana/provisioning
    depends_on:
      - zabbix-server

  zabbix-agent:
    image: zabbix/zabbix-agent:latest
    container_name: zabbix-agent
    restart: unless-stopped
    environment:
      - ZBX_HOSTNAME=zabbix-agent
      - ZBX_SERVER_HOST=${ZBX_SERVER_HOST}
      - TZ=${TZ}
    ports:
      - "${ZBX_AGENT_PORT}:10050"

volumes:
  zabbix_mysql_data:
  zabbix_server_data:
  grafana_data:
```
