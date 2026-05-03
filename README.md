## Все основное по туториалу:
Настройка переменных окружения
```bash
$ export GITHUB_USERNAME=<имя_пользователя>
$ export GIST_TOKEN=<сохраненный_токен>
$ alias edit=<nano|vi|vim|subl>
```
Клонирование репозитория
```sh
$ git clone https://github.com/${GITHUB_USERNAME}/lab06 projects/lab_docker
$ cd projects/lab_docker
```
Изменение удалённого репозитория
```sh
$ git remote remove origin
$ git remote add origin https://github.com/${GITHUB_USERNAME}/lab_docker
```
Установка Docker
```sh
# Debian
$ sudo apt-get update
$ sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Создание файла main.py
```sh
$ cat >> main.py <<EOF
print("Hello, Docker!")
EOF
```
Создание файла requirements.txt
```sh
$ cat >> requirements.txt <<EOF
flask
requests
EOF
```
Создание Dockerfile
```sh
$ cat >> Dockerfile <<EOF
FROM python:3.11-slim

WORKDIR /app

COPY app/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .

EXPOSE 5000

CMD ["python", "app.py"]
```
Сборка Docker-образа
```sh
$ docker build -t lab-docker .
$ docker run --rm -it lab-docker
```
### Docker compose

```sh
$ cat >> docker-compose.yml <<EOF
services:
  app:
    build: .
    container_name: lab_docker_app
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_USER: labuser
      DB_PASS: labpass
      DB_NAME: labdb
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.0
    container_name: lab_docker_db
    restart: always
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: labdb
      MYSQL_USER: labuser
      MYSQL_PASSWORD: labpass
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-prootpass"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  db_data:
  EOF
```



## Homework
## Часть 1 — Dockerfile и контейнер
1) Создание Dockerfile
```sh
$ cat > Dockerfile <<'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY app/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

2) Собрать Docker-образ
```sh
$ docker build -t lab_docker_app .
```
3) Запустить контейнер
```sh
$ docker run -d --name lab_docker_app -p 5000:5000 lab_docker_app
```
4) Скопировать README.md в /home/ контейнера + зайти в контейнер + проверить файл внутри контейнера + выйти из контейнера
```sh
$ docker cp README.md lab_docker_app:/home/README.md
$ docker exec -it lab_docker_app sh
# ls -la /home
# exit
```
5) Остановить и удалить контейнер
```sh
docker stop lab_docker_app
docker rm lab_docker_app
```

## Часть 2 — Docker Compose + MySQL
1) Исправить HTML-шаблон
```sh
cat > app/templates/index.html <<'EOF'
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Docker Lab</title>
</head>
<body>
    <h1>Список из базы данных</h1>

    <ul>
        {% for item in items %}
            <li>{{ item.name }}</li>
        {% endfor %}
    </ul>
</body>
</html>
EOF
```
2) Исправить db/init.sql
```sh
cat > db/init.sql <<'EOF'
SET NAMES utf8mb4;

CREATE TABLE items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

INSERT INTO items (name) VALUES
('Пример 1'),
('Пример 2');
EOF
```

3) Исправить app/models.py
```sh
cat > app/models.py <<'EOF'
import os
import mysql.connector


class ItemModel:
    def __init__(self):
        self.config = {
            'host': os.getenv('DB_HOST', 'db'),
            'user': os.getenv('DB_USER', 'labuser'),
            'password': os.getenv('DB_PASS', 'labpass'),
            'database': os.getenv('DB_NAME', 'labdb'),
            'charset': 'utf8mb4',
            'collation': 'utf8mb4_unicode_ci',
            'use_unicode': True
        }

    def get_all_items(self):
        try:
            conn = mysql.connector.connect(**self.config)
            cursor = conn.cursor(dictionary=True)
            cursor.execute('SELECT name FROM items')
            items = cursor.fetchall()
            cursor.close()
            conn.close()
            return items
        except Exception as e:
            print(f"Error: {e}")
            return []
EOF
```

4) Создать docker-compose.yml
```sh
cat > docker-compose.yml <<'EOF'
services:
  app:
    build: .
    container_name: lab_docker_app
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_USER: labuser
      DB_PASS: labpass
      DB_NAME: labdb
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.0
    container_name: lab_docker_db
    restart: always
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: labdb
      MYSQL_USER: labuser
      MYSQL_PASSWORD: labpass
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-prootpass"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  db_data:
EOF
```

5) Удалить старые контейнеры
```sh
docker compose down -v
```

6) Запуск
```sh
docker compose up --build
```
<img width="1215" height="685" alt="Снимок экрана 2026-05-03 в 09 25 18" src="https://github.com/user-attachments/assets/4ade32e5-139e-414c-b0e8-3439398c267e" />


