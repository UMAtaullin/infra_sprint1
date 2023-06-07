# Деплой проекта Kittygram на удалённый сервер
(Проект создан в рамках учебного курса Яндекс.Практикум)

### Описание проекта

Kittygram — социальная сеть для обмена фотографиями любимых питомцев, где можно добавить нового котика на сайт или изменить существующего, а также просмотреть записи других пользователей.
Проект состоит из бэкенд-приложения на Django и фронтенд-приложения на React.
Цель: вручную развернуть проект на удаленном сервере.

### Стэк технологии
- Python 3.10.7
- Django 3.2.3
- Django REST Framework 3.12.4
- Gunicorn 20.1.0
- Nginx 1.18.0
- Сertbot 2.6.0


### Деплой проекта на удаленный сервер

1. Клонировать репозиторий на сервер
    ```
    git clone git@github.com:UMAtaullin/infra_sprint1.git
    ```
2. Запустите бэкенд
    - Установите на сервер интерпретатор Python, менеджер пакетов pip, утилиту для создания виртуального окружения venv
    ```
    sudo apt install python3-pip python3-venv -y
    ```
    - Создайте виртуальное окружение в директории
    ```
    python3 -m venv venv
    ```
    - Активируйте виртуальное окружение
    ```
    source venv/bin/activate
    ```
    - Устанавите зависимости в директории infra_sprint1
    ```
    pip install -r requirements.txt
    ```
    - Выполните миграции в директории backend
    ```
    python3 manage.py migrate
    ```
    - Создайте суперпользователя
    ```
    python3 manage.py createsuperuser
    ```
    - Установите в файле settings.py для константы DEBUG значение False
     ```
     sudo nano settings.py
     ```
    - Добавьте в список ALLOWED_HOSTS внешний IP-адрес , адреса 127.0.0.1 и localhost
    ```
    ALLOWED_HOSTS = ['xxx.xxx.xxx.xxx', '127.0.0.1', 'localhost']
    ```
3. Запустите фронтенд
    - Установите на сервер пакетный менеджер npm
    ```
    curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - &&\
    sudo apt-get install -y nodejs
    ```
    - установите зависимости для фронтенд-приложения. Перейдите в директорию infra_sprint1/frontend/ и выполните команду:
    ```
    npm i
    ```
4. Установите и запустите Gunicorn
    - Установите пакет gunicorn:
    ```
    pip install gunicorn==20.1.0
    ```
    - Запустите gunicorn в директорию с файлом manage.py:
    ```
    gunicorn --bind 0.0.0.0:8080 kittygram_backend.wsgi
    ```

5. Создайте юнит для Gunicorn
    - Создайте конфигурационный файл Gunicorn для проекта Kittygram
    ```
    sudo nano /etc/systemd/system/gunicorn_kittygram.service
    ```
    - В файле gunicorn_kittygram.service опишите конфигурацию процесса
    ```
    [Unit]

    Description=gunicorn_kittygram daemon
    After=network.target

    [Service]

    User=yc-user
    WorkingDirectory=/home/yc-user/infra_sprint1/backend/
    ExecStart=/home/yc-user/infra_sprint1/backend/venv/bin/gunicorn --bind 0.0.0.0:8080 kittygram_backend.wsgi


    [Install]

    WantedBy=multi-user.target
    ```
6. Запустите процесс gunicorn_kittygram.service
    ```
    sudo systemctl start gunicorn_kittygram
    ```
7. Установите Nginx
    - Из любой директории выполните команду:
    ```
    sudo apt install nginx -y
    ```
    - Укажите файрволу, какие порты должны остаться открытыми:
    ```
    sudo ufw allow 'Nginx Full'
    sudo ufw allow OpenSSH
    ```
    - Включите файрвол:
    ```
    sudo ufw enable
    ```
7. Соберите статику фронтенд-приложения
    - Перейдите в директорию infra_sprint1/frontend и выполните команду:
    ```
    npm run build
    ```
    - Скопируйте в системную директорию /var/www/ содержимое папки .../frontend/build/:
    ```
    sudo cp -r <имя_пользователя>/infra_sprint1/frontend/build/. /var/www/kittygram/
    ```
    - Создайте директорию media в директории /var/www/kittygram/
    ```
    sudo mkdir media
    ```
    - В настройках бэкенда для константы MEDIA_ROOT укажите путь до созданной директории media:
    ```
    MEDIA_ROOT = '/var/www/kittygram/media'
    ```
    - Назначьте текущего пользователя владельцем директории media:
    ```
    sudo chown -R <имя_пользователя> /var/www/kittygram/media/
    ```
8. Получите доменное имя, по которому будет доступно приложение
    - Используйте любой сервис, выдающий доменные имена, например No-IP
    - Добавьте доменное имя в настройки Django
    ```
    ALLOWED_HOSTS = ['xxx.xxx.xxx.xxx', '127.0.0.1', 'localhost', 'ваш-домен']
    ```
    - Перезапустите Gunicorn
    ```
    sudo systemctl restart gunicorn_kittygram
    ```
    - Пропишите в конфигурационном файле Nginx выданное вам доменное имя:
    ```
        server {
    ...
        server_name <ваш-ip> <ваш-домен>;
    ...
    }
    ```
    - Проверьте корректность конфигурации и перезагрузите её:
    ```
    sudo nginx -t
    sudo systemctl reload nginx
    ```
9. Настройте файл конфигурации веб-сервера
    - Через редактор Nano откройте файл конфигурации веб-сервера:
    ```
    sudo nano /etc/nginx/sites-enabled/default
    ```
    - Удалите все настройки из файла и запишите новые:
    ```
    server {

        server_name 158.160.7.223 <ваш-домен>;

        location /api/ {
            proxy_pass http://127.0.0.1:8080;
        }

        location /admin/ {
            proxy_pass http://127.0.0.1:8080;
        }

        location /media/ {
            alias /var/www/kittygram/media/;
        }

        location / {
            root   /var/www/kittygram;
            index  index.html index.htm;
            try_files $uri /index.html;
        }

    }
    ```
    - Проверьте корректность конфигурации и перезагрузите её:
    ```
    sudo nginx -t
    sudo systemctl reload nginx
    ```
10. Настройте шифрование запросов по протоколу HTTPS
    - Установите certbot на ваш удалённый сервер
    ```
    sudo apt install snapd
    sudo snap install core; sudo snap refresh core
    sudo snap install --classic certbot
    sudo ln -s /snap/bin/certbot /usr/bin/certbot
    ```
    - Чтобы начать процесс получения сертификата, введите команду:
    ```
    sudo certbot --nginx
    ```
    - Перезагрузите конфигурацию Nginx:
    ```
    sudo systemctl reload nginx
    ```
    - Настройте автоматическое обновление SSL-сертификата
    ```
    sudo certbot renew --dry-run
    ```

11. Проверти работу сайта в браузере по адресу https://<ваш-домен>/

### Автор
Атауллин Урал, студент Яндекс практикумa.

