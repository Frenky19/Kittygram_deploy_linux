# Деплой проекта Kittygram на удалённый сервер

### Описание проекта

Kittygram — социальная сеть для обмена фотографиями любимых питомцев. Проект
состоит из бэкенд-приложения на Django и фронтенд-приложения на React.

Возможности социальной сети:

* Регистрация.
* Добавление фото.
* Удаление фото.
* Выбор цвета питомца.
* Описание достижений питомца.
* Указание года рождения питомца.

### Деплой
Подключитесь к удалённому серверу
```
ssh -i путь_до_SSH_ключа/название_файла_с_SSH_ключом_без_расширения login@ip
```

Клонируйте проект на сервер
```
git clone git@github.com:Frenky19/infra_sprint1.git
```

Переходим в директорию backend-приложения проекта
```
cd infra_sprint1/backend/
```
Создаём виртуальное окружение
```
python -m venv venv
```
Активируем виртуальное окружение
```
source venv/bin/activate
```
Обновляем pip в виртуальном окружении
```
pip install --upgrade pip
```
Устанавливаем зависимости
```
pip install -r requirements.txt
```
Из директории, в которой находится файл manage.py применяем миграции
```
python manage.py migrate
```
Создаём суперпользователя
```
python manage.py createsuperuser
```
Собираем статику бэкенда
```
python manage.py collectstatic
```
Из корня проекта скопируем статику бэкенда в системную директорию
```
sudo cp -r /home/yc-user/infra_sprint1/backend/static_backend/ /var/www/kittygram/
```
Запускаем веб-сервер разработки Django
```
python manage.py runserver
```
В файле settings.py xxx.xxx.xxx.xxx укажите IP вашего сервера
```
ALLOWED_HOSTS = ['xxx.xxx.xxx.xxx', '127.0.0.1', 'localhost']
```
В другом окне терминала установите зависимости для фронтенд-приложения. Перейдите в директорию infra_sprint1/frontend/ и выполните команду
```
npm i
```
Запустите приложение командой
```
npm run start
```
Проверте тестовый запуск в браузере по адресу
http://внешний_ip_адрес_сервера:8000

### Установка и запуск Gunicorn
На удалённом сервере при активированном виртуальном окружении проекта
```
pip install gunicorn==20.1.0
```
Из директории с файлом manage.py
```
gunicorn --bind 0.0.0.0:8000 backend.wsgi
```
Страница администратора - должна работать без статики
```
http://ваш_публичный_IP:8000/admin/
```
Остановим и запустим для непрерывной работы.

В директории /etc/systemd/system/ создайте файл gunicorn.service и откройте его в Nano
```
sudo nano /etc/systemd/system/gunicorn.service
```
Подставьте в код из листинга свои данные, добавьте этот код без комментариев в файл gunicorn.service и сохраните изменения
```
[Unit]
# Это текстовое описание юнита, пояснение для разработчика.
Description=gunicorn daemon 

# Условие: при старте операционной системы запускать процесс только после того, 
# как операционная система загрузится и настроит подключение к сети.
# Ссылка на документацию с возможными вариантами значений 
# https://systemd.io/NETWORK_ONLINE/
After=network.target 

[Service]
# От чьего имени будет происходить запуск:
# укажите имя, под которым вы подключались к серверу.
User=<имя-пользователя-в-системе>

# Путь к директории проекта:
# /home/<имя-пользователя-в-системе>/
# <директория-с-проектом>/<директория-с-файлом-manage.py>/.
# Например:
WorkingDirectory=/home/<имя-пользователя-в-системе>/infra_sprint1/backend/

# Команду, которую вы запускали руками, теперь будет запускать systemd:
# /home/<имя-пользователя-в-системе>/
# <директория-с-проектом>/<путь-до-gunicorn-в-виртуальном-окружении> --bind 0.0.0.0:8000 backend.wsgi
ExecStart=/home/yc-user/prettypets-django-react/backend/venv/bin/gunicorn --bind 0.0.0.0:8000 backend.wsgi

[Install]
# В этом параметре указывается вариант запуска процесса.
# Значение <multi-user.target> указывают, чтобы systemd запустил процесс,
# доступный всем пользователям и без графического интерфейса.
WantedBy=multi-user.target
```
Чтобы точно узнать путь до Gunicorn, активируйте виртуальное окружение и воспользуйтесь командой
```
which gunicorn
```
Заново запустите процесс gunicorn.service
```
sudo systemctl start gunicorn 
```
Добавьте процесс Gunicorn в список автозапуска операционной системы на удалённом сервере
```
sudo systemctl enable gunicorn 
```
### Установка Nginx
Находясь на удалённом сервере, из любой директории
```
sudo apt install nginx -y
sudo systemctl start nginx
```
Укажите файрволу, какие порты должны остаться открытыми
```
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
```
Включите файрвол
```
sudo ufw enable
```
Проверьте внесённые изменения
```
sudo ufw status
```
Запустите сборку фронтенд-приложения из директории infra_sprint1/frontend/
```
npm run build
```
Скопируйте в системную директорию Nginx (которую он использует по умолчанию для доступа к статическим файлам — /var/www/) содержимое папки .../frontend/build/
```
sudo cp -r /home/yc-user/infra_sprint1/frontend/build/. /var/www/kittygram/
```
Чтобы фотографии отображались на сайте, создайте директорию media в директории /var/www/kittygram/. Django-приложение будет использовать эту директорию для хранения картинок.
В настройках бэкенда для константы MEDIA_ROOT указан путь до созданной директории media.
Назначьте текущего пользователя владельцем директории media, чтобы Django-приложение могло сохранять картинки. Для этого используйте команду chown
```
sudo chown -R <имя_пользователя> /var/www/kittygram/media/
sudo chown -R yc-user /var/www/kittygram/media/
```

Через редактор Nano откройте файл конфигурации веб-сервера
```
sudo nano /etc/nginx/sites-enabled/default
```
Удалите все настройки из файла, запишите и сохраните новые
```
server {

    listen 80;
    server_name публичный_ip_вашего_удалённого_сервера;
    
    location /api/ {
        client_max_body_size 20M;
        # Эта команда определяет, куда нужно перенаправить запрос.
        proxy_pass http://127.0.0.1:8000;
    }

    location /admin/ {
        client_max_body_size 20M;
        proxy_pass http://127.0.0.1:8000;
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
Проверьте файл конфигурации на ошибки
```
sudo nginx -t
```
Перезагрузите конфигурацию Nginx
```
sudo systemctl reload nginx
```
В адресную строку браузера введите внешний IP вашего удалённого сервера без указания порта.

Команда для просмотра лога последних запросов
```
sudo tail /var/log/nginx/access.log
```
Команда для просмотра лога последних ошибок
```
sudo tail /var/log/nginx/error.log
```

### Получение и настройка SSL-сертификата

Установка certbot
```
sudo apt install snapd 
```

Далее сервер, скорее всего, попросит вас перезагрузить операционную систему. Сделайте это, а потом последовательно выполните команды:
```
sudo snap install core; sudo snap refresh core

sudo snap install --classic certbot

sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Запускаем certbot и получаем SSL-сертификат:
```
sudo certbot --nginx 
```
Далее система оповестит вас о том, что учётная запись зарегистрирована и попросит указать имена, для которых вы хотели бы активировать HTTPS.
В файл /etc/nginx/sites-enabled/default добавятся новые настройки и будут прописаны пути к сертификату.

Перезагрузите конфигурацию Nginx:
```
sudo systemctl reload nginx 
```

Чтобы узнать актуальный статус сертификата и сколько дней осталось до его перевыпуска, используйте команду:
```
sudo certbot certificates
```

Убедитесь, что сертификат будет обновляться автоматически.
Если не выведется ошибка, значит, всё в порядке.
```
sudo certbot --nginx 
```

Вручную сертификат можно обновить командой:
```
sudo certbot renew --pre-hook "service nginx stop" --post-hook "service nginx start"  
```