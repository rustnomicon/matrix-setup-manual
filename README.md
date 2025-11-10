# Инструкция по развёртыванию Matrix Synapse с доменом, Let's Encrypt SSL и EPT-MX-ADM

## Фаза 1: Подготовка домена и DNS

### Шаг 1.1 - Получить домен

**Вариант A: Купить домен**

- Namecheap, Porkbun, GoDaddy: домены `.xyz`, `.top` от \$1/год
- Пример: `matrix.exampleserver.com`

**Вариант B: Бесплатный DuckDNS**

1. Зайти на <https://www.duckdns.org>
2. Войти через Google/GitHub
3. Создать поддомен: `mymatrix.duckdns.org`
4. Указать IP вашего сервера: `YOUR.IP.ADDRESS`
5. Нажать **Update IP**

### Шаг 1.2 - Настроить DNS записи

**Для купленного домена:**

В панели регистратора добавить A-записи:

```
Тип    Имя              Значение           TTL
A      matrix           YOUR.IP.ADDRESS      3600
A      @                YOUR.IP.ADDRESS      3600
```

**Для DuckDNS:** DNS уже настроен автоматически.

### Шаг 1.3 - Проверить DNS

```bash
# Проверить A-запись
dig matrix.exampleserver.com
dig matrix.exampleserver.com +short и dig AAAA matrix.exampleserver.com +short

# или для DuckDNS
dig mymatrix.duckdns.org

# Должно вернуть IP вашего сервера
```

Подождите 5-10 минут для распространения DNS.

***

## Фаза 2: Подготовка системы

### Шаг 2.1 - Обновить систему

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install curl wget vim pwgen ufw git -y
```

### Шаг 2.2 - Установить Docker и Docker Compose

```bash
# Установить Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Добавить пользователя в группу
sudo usermod -aG docker $USER

# Перелогиниться
newgrp docker

# Установить Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Проверить
docker --version
docker-compose --version
```

### Шаг 2.3 - Настроить firewall

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 3478/tcp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp
sudo ufw allow 5349/udp
sudo ufw allow 49152:49252/udp

sudo ufw enable
sudo ufw status
```

***

## Фаза 3: Создание структуры проекта

```bash
sudo mkdir -p /opt/matrix-synapse/{data,coturn,logs,ept-mx-adm}
cd /opt/matrix-synapse
sudo chown -R $USER:$USER /opt/matrix-synapse
```

***

## Фаза 4: Генерация конфигурации Synapse

### Шаг 4.1 - Сгенерировать homeserver.yaml

**Замените `matrix.exampleserver.com` на ваш домен:**

```bash
cd /opt/matrix-synapse

docker run -it --rm \
  -v /opt/matrix-synapse/data:/data \
  -e SYNAPSE_SERVER_NAME=matrix.exampleserver.com \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate
```

### Шаг 4.2 - Настроить homeserver.yaml

```bash
vim /opt/matrix-synapse/data/homeserver.yaml
```

**Изменить следующие секции:**

#### 1. Отключить регистрацию

```yaml
enable_registration: false
```

#### 2. Политика ретенции (7 дней)

Добавить в конец:

```yaml
retention:
  enabled: true
  default_policy:
    min_lifetime: 86400000
    max_lifetime: 604800000
  allowed_lifetime_min: 86400000
  allowed_lifetime_max: 2592000000
```

#### 3. Listener

```yaml
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    bind_addresses: ['0.0.0.0']
    
    resources:
      - names: [client, federation]
        compress: false
```

#### 4. Отключить федерацию (опционально)

```yaml
federation_domain_whitelist: []
allow_public_rooms_over_federation: false
```

#### 5. База данных PostgreSQL

```yaml
database:
  name: psycopg2
  args:
    user: synapse
    password: SECURE_DB_PASSWORD_HERE
    database: synapse
    host: postgres
    port: 5432
    cp_min: 5
    cp_max: 10
```

**Сгенерировать пароль БД:**

```bash
pwgen -s 32 1
```

#### 6. Метрики

```yaml
enable_metrics: true
metrics_port: 9000
```

**Сохранить** :wq!.

***

## Фаза 5: Настройка Coturn

### Шаг 5.1 - Сгенерировать TURN secret

```bash
TURN_SECRET=$(pwgen -s 64 1)
echo "TURN_SECRET: $TURN_SECRET" | tee /opt/matrix-synapse/coturn/turnserver.secret
echo $TURN_SECRET
```

### Шаг 5.2 - Создать turnserver.conf

```bash
vim /opt/matrix-synapse/coturn/turnserver.conf
```

**Вставить (замените домен и secret):**

```ini
realm=matrix.exampleserver.com
server-name=matrix.exampleserver.com

listening-ip=0.0.0.0
external-ip=YOUR.IP.ADDRESS
relay-ip=YOUR.IP.ADDRESS

listening-port=3478
tls-listening-port=5349
min-port=49152
max-port=49252

fingerprint
log-file=/var/log/coturn/turnserver.log
verbose

static-auth-secret=YOUR_TURN_SECRET_HERE
lt-cred-mech

denied-peer-ip=0.0.0.0-0.255.255.255
denied-peer-ip=127.0.0.0-127.255.255.255
denied-peer-ip=::1
```

### Шаг 5.3 - Добавить TURN в homeserver.yaml

```bash
vim /opt/matrix-synapse/data/homeserver.yaml
```

Добавить в конец:

```yaml
turn_uris:
  - "turn:matrix.exampleserver.com:3478?transport=udp"
  - "turn:matrix.exampleserver.com:3478?transport=tcp"
  - "turns:matrix.exampleserver.com:5349?transport=tcp"

turn_shared_secret: "YOUR_TURN_SECRET_HERE"
turn_allow_guests: true
```

***

## Фаза 6: Установка EPT-MX-ADM

### Шаг 6.1 - Клонировать репозиторий

```bash
cd /opt/matrix-synapse
git clone https://github.com/EPTLLC/EPT-MX-ADM.git ept-mx-adm
cd ept-mx-adm
```

### Шаг 6.2 - Настроить config.json

```bash
vim config.json
```

**Изменить:**

```json
{
  "matrix_server": "https://matrix.exampleserver.com",
  "app": {
    "host": "0.0.0.0",
    "port": 5000,
    "debug": false
  },
  "language": "ru"
}
```

### Шаг 6.3 - Сгенерировать SECRET_KEY

```bash
python3 -c 'import secrets; print(secrets.token_hex(32))'
```

Скопируйте результат для следующего шага.

***

## Фаза 7: Создание docker-compose.yml

```bash
cd /opt/matrix-synapse
vim docker-compose.yml
```

**Полная конфигурация:**

```yaml
version: "3.8"

services:
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: matrix_synapse
    restart: unless-stopped
    
    volumes:
      - ./data:/data
      - ./logs:/var/log/synapse
    
    ports:
      - "8008:8008"
      - "9000:9000"
    
    environment:
      SYNAPSE_SERVER_NAME: matrix.exampleserver.com
      SYNAPSE_REPORT_STATS: "no"
    
    networks:
      - matrix_network
    
    depends_on:
      - postgres
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8008/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:15-alpine
    container_name: matrix_postgres
    restart: unless-stopped
    
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    
    environment:
      POSTGRES_DB: synapse
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: "YOUR_DB_PASSWORD_HERE"
    
    networks:
      - matrix_network
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U synapse"]
      interval: 10s
      timeout: 5s
      retries: 5

  coturn:
    image: coturn/coturn:latest
    container_name: matrix_coturn
    restart: unless-stopped
    
    volumes:
      - ./coturn/turnserver.conf:/etc/coturn/turnserver.conf:ro
      - ./logs:/var/log/coturn
    
    ports:
      - "3478:3478/tcp"
      - "3478:3478/udp"
      - "5349:5349/tcp"
      - "5349:5349/udp"
      - "49152-49252:49152-49252/udp"
    
    networks:
      - matrix_network
    
    command: -n --log-file=stdout

  ept-mx-adm:
    image: python:3.11-slim
    container_name: matrix_admin
    restart: unless-stopped
    
    working_dir: /app
    
    volumes:
      - ./ept-mx-adm:/app
    
    ports:
      - "5000:5000"
    
    environment:
      FLASK_SECRET_KEY: "YOUR_GENERATED_SECRET_KEY_HERE"
      FLASK_APP: "app.py"
    
    networks:
      - matrix_network
    
    depends_on:
      - synapse
    
    command: >
      bash -c "
      apt-get update && 
      apt-get install -y git && 
      pip install --no-cache-dir -r requirements.txt && 
      chmod +x install_assets.sh && 
      ./install_assets.sh && 
      python app.py
      "

networks:
  matrix_network:
    driver: bridge
```

**Заменить:**

- `YOUR_DB_PASSWORD_HERE` - пароль PostgreSQL
- `YOUR_GENERATED_SECRET_KEY_HERE` - SECRET_KEY из шага 6.3

***

## Фаза 8: Запуск контейнеров

```bash
cd /opt/matrix-synapse

# Запустить
docker-compose up -d

# Проверить статус
docker-compose ps

# Посмотреть логи
docker-compose logs -f
```

### Шаг 8.1 - Хотфикс локали БД

```bash
cd /opt/matrix-synapse

docker compose stop synapse
# Запретить новые подключения к synapse
docker exec -it matrix_postgres psql -U synapse -d postgres -c "REVOKE CONNECT ON DATABASE synapse FROM PUBLIC;"

# Завершить активные подключения к synapse
docker exec -it matrix_postgres psql -U synapse -d postgres -c "SELECT pid, pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'synapse';"

docker exec -it matrix_postgres psql -U synapse -d postgres -c "DROP DATABASE IF EXISTS synapse;"

docker exec -it matrix_postgres psql -U synapse -d postgres -c "CREATE DATABASE synapse WITH OWNER=synapse TEMPLATE=template0 ENCODING='UTF8' LC_COLLATE='C' LC_CTYPE='C';"

docker compose start synapse

```

Ждите 2-3 минуты для инициализации.

***

## Фаза 9: Установка Nginx и Let's Encrypt

### Шаг 9.1 - Установить Nginx и Certbot

```bash
sudo apt-get install nginx certbot python3-certbot-nginx -y
```

### Шаг 9.2 - Получить SSL-сертификат

```bash
# Остановить Nginx
sudo systemctl stop nginx

# Получить сертификат (замените домен)
sudo certbot certonly --standalone -d matrix.exampleserver.com

# Certbot спросит email и согласие с ToS
```

Сертификаты будут в `/etc/letsencrypt/live/matrix.exampleserver.com/`.

### Шаг 9.3 - Создать конфиг Nginx

```bash
sudo vim /etc/nginx/sites-available/matrix
```

**Вставить (замените домен):**

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name matrix.exampleserver.com;

    ssl_certificate     /etc/letsencrypt/live/matrix.exampleserver.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.exampleserver.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    client_max_body_size 50M;
    proxy_read_timeout 600s;
    proxy_connect_timeout 600s;
    proxy_send_timeout 600s;

    # Matrix Admin API
    location /_synapse/admin {
        proxy_pass http://localhost:8008;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;

        if ($request_method = 'OPTIONS') { return 204; }
    }

    # Matrix Client API
    location /_matrix {
        proxy_pass http://localhost:8008;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
    }

    # Перенаправление /login → /admin/login
    location = /login {
        return 301 /admin/login;
    }

    # EPT-MX-ADM (SPA под префиксом /admin/)
    location /admin/ {
        proxy_pass http://localhost:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_request_buffering off;
        proxy_redirect off;

        try_files $uri $uri/ @ept_admin;
    }

    # В именованном location нельзя указывать proxy_pass с URI-частью — без завершающего слэша!
    location @ept_admin {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Well-known
    location /.well-known/matrix/client {
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
        return 200 '{"m.homeserver": {"base_url": "https://matrix.exampleserver.com"}}';
    }

    location /.well-known/matrix/server {
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
        return 200 '{"m.server": "matrix.exampleserver.com:443"}';
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name matrix.exampleserver.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

### Шаг 9.4 - Активировать конфиг

```bash
sudo ln -s /etc/nginx/sites-available/matrix /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl start nginx
sudo systemctl enable nginx
```

***

## Фаза 10: Создание пользователей

```bash
# Войти в контейнер
docker exec -it matrix_synapse bash

# Создать администратора
register_new_matrix_user -c /data/homeserver.yaml -u admin -p StrongPass123! --admin

# Создать обычных пользователей
register_new_matrix_user -c /data/homeserver.yaml -u user1 -p Pass456!

# Выход
exit
```

***

## Фаза 11: Доступ к EPT-MX-ADM

### Через браузер

```
https://matrix.exampleserver.com/admin/
```

**При входе:**

- **Matrix Server**: `https://matrix.exampleserver.com` (или оставить пустым)
- **Username**: `admin` (без @ и домена, автоформатируется)
- **Password**: ваш пароль

***

## Фаза 12: Подключение Element

### Desktop/Web

1. Открыть Element Desktop или <https://app.element.io>
2. При входе нажать **Edit** / **Change server**
3. Указать: `https://matrix.exampleserver.com`
4. Залогиниться: `@admin:matrix.exampleserver.com` + пароль

### Android/iOS

1. Скачать Element из Google Play / App Store
2. При запуске нажать **Change server**
3. Указать: `https://matrix.exampleserver.com`
4. Залогиниться

***

## Фаза 13: Автообновление SSL-сертификата

```bash
# 1) Переиздать сертификат через nginx-плагин с заменой (обновит renewal-профиль)
sudo certbot certonly --nginx -d chat7651.ru
# В диалоге выбери: 2) Renew & replace

# 2) Проверить автопродление без простоя
sudo certbot renew --dry-run

# 3) (Опц.) Хук на перезагрузку Nginx после реального обновления
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy
echo -e '#!/bin/sh\nsystemctl reload nginx' | sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh >/dev/null
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

***

## Быстрые команды

```bash
# Перезапуск
docker-compose restart

# Логи
docker-compose logs -f synapse
docker-compose logs -f ept-mx-adm

# Новый пользователь
docker exec -it matrix_synapse register_new_matrix_user -c /data/homeserver.yaml -u username -p password

# Статус
docker-compose ps
curl https://matrix.exampleserver.com/_matrix/client/versions
```

***

## Troubleshooting

**SSL-сертификат не получается:**

- Убедитесь, что порт 80 открыт
- Проверьте DNS: `dig matrix.exampleserver.com`
- Попробуйте через Certbot с Nginx: `sudo certbot --nginx -d matrix.exampleserver.com`

**EPT-MX-ADM не запускается:**

```bash
docker-compose logs ept-mx-adm
# Если ошибки с зависимостями - пересоздать контейнер
docker-compose up -d --force-recreate ept-mx-adm
```

**Element не подключается:**

- Проверьте well-known: `curl https://matrix.exampleserver.com/.well-known/matrix/client`
- Проверьте API: `curl https://matrix.exampleserver.com/_matrix/client/versions`

***

Готово! Полностью функциональный Matrix-сервер с доменом, автоматическим SSL и современной админ-панелью EPT-MX-ADM развёрнут.
