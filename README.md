# docker-wp-woo 
Docker Stack for wp and woo 

При стандартной установке на локальной машине адреса будут такими:

Сам сайт: http://localhost
Админка (Dashboard): http://localhost/wp-admin
Adminer : localhost:8080


=============================================================
1. Файл .env
Самое важное здесь — передать твои UID и GID, чтобы Docker не создавал файлы от имени root.

Bash

# Твой ID пользователя в Linux (узнать можно командой `id -u`)
# Для каждого нового сайта просто меняй порты в файле .env, чтобы они не повторялись.
# Пример для первого сайта:
USER_ID=1000
GROUP_ID=1000
DB_NAME=wordpress
DB_USER=wordpress
DB_PASSWORD=secret
DB_ROOT_PASSWORD=root_secret

HOST_PORT=80
ADMINER_PORT=8080


# Пример для второго сайта:
# USER_ID=1000
# GROUP_ID=1000
# DB_NAME=wordpress
# DB_USER=wordpress
# DB_PASSWORD=secret
# DB_ROOT_PASSWORD=root_secret

# HOST_PORT=8081
# ADMINER_PORT=8082
# Один на http://localhost, другой на http://localhost:8081


=============================================================
2. Файл docker-compose.yml
Я добавил сервис wp-cli. Он использует тот же том (volume) с файлами, что и основной WordPress.

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
      # Заменили именованный том на локальную папку проекта
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
      - ./wp-content:/var/www/html/wp-content

  wp-cli:
    image: wordpress:cli
    user: "${USER_ID}:${GROUP_ID}"
    volumes:
      - ./wp-content:/var/www/html/wp-content
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
    depends_on:
      - db

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      # Теперь порт берется из .env
      - "${HOST_PORT}:80"
    volumes:
      - ./wp-content:/var/www/html/wp-content
      - ./config/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - wordpress

  adminer:
    image: adminer
    restart: always
    ports:
      # Теперь порт берется из .env
      - "${ADMINER_PORT}:8080"

=============================================================
3. Файл config/nginx.conf
Минимальный конфиг, чтобы всё работало через Nginx:

Nginx

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://wordpress:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

=============================================================
Как этим пользоваться (Инструкция)

Шаг 1: Подготовка прав (ОДИН РАЗ)
Перед запуском в папке проекта выполни:

Bash

# Даем права твоему пользователю на текущую папку
sudo chown -R $USER:$USER .   (!! ТОЧКА в конце ОБЯЗАТЕЛЬНА)
# Создаем папку для контента, если её нет
mkdir -p wp-content config

Шаг 2: Запуск
Bash

docker compose up -d

Шаг 3: Как использовать WP-CLI
Теперь ты можешь выполнять команды WP-CLI без установки его на Linux Mint. Например:

Проверить состояние: docker-compose run --rm wp-cli core version

Установить WooCommerce: docker-compose run --rm wp-cli plugin install woocommerce --activate

Обновить всё: docker-compose run --rm wp-cli core update

Почему это решает твои проблемы:
Права: Благодаря user: "${USER_ID}:${GROUP_ID}", любой файл, созданный внутри контейнера (плагин, картинка, обновление), будет принадлежать тебе. Ты сможешь редактировать их в VS Code или Sublime без sudo.

Легкость: Мы используем стандартные образы.

Adminer: Просто зайди на localhost:8080, чтобы быстро поправить что-то в базе.
