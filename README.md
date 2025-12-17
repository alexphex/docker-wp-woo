
🚀 WordPress Docker Environment (Linux Mint optimized)
Эта сборка позволяет запускать несколько независимых сайтов на одной машине без конфликтов портов и проблем с правами (sudo).

1. Настройка окружения (.env)
Создай файл .env в корне проекта. Для каждого нового сайта (wp2, wp3) просто меняй порты и названия БД.

Bash

# Твой ID пользователя в Linux (узнай командой `id -u` и `id -g`)
USER_ID=1000
GROUP_ID=1000

# Настройки базы данных
DB_NAME=wordpress
DB_USER=wordpress
DB_PASSWORD=secret
DB_ROOT_PASSWORD=root_secret

# Порты (для второго сайта смени, например, на 8081 и 8082)
HOST_PORT=80
ADMINER_PORT=8080
2. Конфигурация Docker (docker-compose.yml)
Создай файл docker-compose.yml. Обрати внимание: мы монтируем всю папку html, чтобы ты видел ядро WordPress.

YAML

version: '3.8'

services:
  db:
    image: mariadb:10.6
    restart: always
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - ./db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    restart: always
    user: "${USER_ID}:${GROUP_ID}"
    depends_on:
      - db
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
      WORDPRESS_CONFIG_EXTRA: |
        define('FS_METHOD', 'direct');
    volumes:
      - ./html:/var/www/html

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "${HOST_PORT}:80"
    volumes:
      - ./html:/var/www/html
      - ./config/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - wordpress

  wp-cli:
    image: wordpress:cli
    user: "${USER_ID}:${GROUP_ID}"
    volumes:
      - ./html:/var/www/html
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
    depends_on:
      - db

  adminer:
    image: adminer
    restart: always
    ports:
      - "${ADMINER_PORT}:8080"
3. Конфигурация Nginx (config/nginx.conf)
Создай папку config и в ней файл nginx.conf. Этот конфиг правильно передает запросы к WordPress.

Nginx

server {
    listen 80;
    server_name localhost;

    root /var/www/html;
    index index.php;

    location / {
        proxy_pass http://wordpress:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Оптимизация: Nginx сам отдает статику, не нагружая WordPress
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
    }
}
🛠 Как пользоваться (Быстрый старт)
Шаг 1: Подготовка папок
В терминале внутри папки проекта выполни:

Bash

mkdir -p html config
Это важно: если папки создаст Docker, они будут принадлежать root. Если создашь ты — у тебя будут полные права.

Шаг 2: Запуск
Bash

docker compose up -d
Шаг 3: Работа с файлами
Теперь все файлы WordPress появятся в папке html. Ты можешь открывать их в VS Code, менять wp-config.php или файлы в wp-content без sudo.

Шаг 4: База данных (Adminer)
Зайди на http://localhost:8080

Система: MySQL

Сервер: db (не localhost!)

Логин/Пароль: из твоего файла .env

Шаг 5: Команды WP-CLI
Если нужно установить плагин или сбросить пароль:

Bash

docker compose run --rm wp-cli search-replace 'old-domain.com' 'new-domain.com'
💡 Как поднять второй сайт?
Создай новую папку для второго проекта (например, ~/wp-site-2).

Скопируй туда эти же файлы.

В файле .env измени:

HOST_PORT=8081

ADMINER_PORT=8082

(Опционально) названия БД.

Запусти docker compose up -d. Теперь первый сайт на порту 80, а второй — на 8081.
