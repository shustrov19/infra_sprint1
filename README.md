# Kittygram
### Описание 
Проект c backend на Django c установкой на удаленный сервер.Благодаря этому проекту, можно на сайт [Kittygram](https://kitty-gramm.ddns.net/) добавлять карточки котиков, с их описанием (цвет, год рождения, фотография) и указывать, чем они отличились.
### Технологии 

- Python 3
- Django
- Nginx
- Gunicorn
- React
- Django REST Framework
- Certbot

## Инструкция по запуску
### Подключение к удалённому серверу по SSH-ключу 
В терминале Git Bash введите:
```
ssh -i путь_до_SSH_ключа/название_файла_с_SSH_ключом_без_расширения login@ip
```

### Клонирование проекта с GitHub на сервер

```
git@github.com:ваш_аккаунт/infra_sprint1.git
```
### Настройка бэкенд-приложения
1. Установите зависимости из файла requirements.txt:

- Перейдите в директорию бэкенд-приложения проекта
```
cd название_проекта/backend/
```
- Создайте виртуальное окружение.
```
python3 -m venv venv
```
- Активируйте виртуальное окружение.
```
source venv/bin/activate
```
- Установите зависимости.
```
pip install -r requirements.txt
```
2. Выполните миграции и создайте суперюзера из директории с файлом manage.py:
- Примените миграции.
```
python3 manage.py migrate
```
- Создайте суперпользователя.
```
python3 manage.py createsuperuser
```
3. Добавьте в список ALLOWED_HOSTS внешний IP сервера, 127.0.0.1, localhost и домен:
Вместо xxx.xxx.xxx.xxx укажите IP сервера, а вместо <ваш_домен> – доменное имя.
```
ALLOWED_HOSTS = ['xxx.xxx.xxx.xxx', '127.0.0.1', 'localhost', 'ваш_домен']
```
4. Откройте файл settings.py и отключите режим дебага для бэкенд-приложения:
```
from pathlib import Path
...
# Вот тут нужно поменять значение с True на False.
DEBUG = False
...
```
5. Подготовьте бэкенд-приложение для сбора статики. В файле settings.py укажите директорию, куда эту статику нужно сложить. Через редактор Nano откройте файл settings.py, укажите новое значение для константы STATIC_URL, добавьте константу STATIC_ROOT и сохраните изменения в файле:
- Замените стандартное значение 'static' на 'static_backend'.
```
STATIC_URL = 'static_backend'
```
- Укажите директорию, куда бэкенд-приложение должно сложить статику.
```
STATIC_ROOT = BASE_DIR / 'static_backend'
```
6. Соберите статику бэкенд-приложения:
```
python3 manage.py collectstatic
```
7. Скопируйте директорию static_backend/ в директорию /var/www/kittygram/:
```
sudo cp -r путь_к_директории_с_бэкендом/static_backend /var/www/kittygram
```
8. В директории проекта создайте файл .env и запишите в него переменную SECRET_KEY из infra_sprint1/backend/kittygram_backend/settings.py в формате ключ=значение:
```
SECRET_KEY=<secret_key из settings.py>
```

Чтобы поместить переменные из этого файла в пространство переменных окружения — понадобится библиотека python-dotenv; установите её через pip:
```
(venv) ... pip install python-dotenv 
```
Из этой библиотеки импортируйте в код в settings.py и выполните функцию load_dotenv():
```
import os
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

SECRET_KEY = os.getenv('SECRET_KEY')
...
```
Файл .env должен лежать в той же директории, что и исполняемый файл manage.py. 
9. Чтобы фотографии котиков отображались на сайте, создайте директорию media в директории /var/www/kittygram/. Django-приложение будет использовать эту директорию для хранения картинок.
10. В settings.py для константы MEDIA_ROOT укажите путь до созданной директории media.
11. Назначьте текущего пользователя владельцем директории media, чтобы Django-приложение могло сохранять картинки. Для этого используйте команду chown:
```
 # Подставьте в команду имя своего пользователя.
 sudo chown -R <имя_пользователя> /var/www/kittygram/media/
 
```
### Настройка фронтенд-приложения
1. Находясь в директории с фронтенд-приложением, установите зависимости для него:
```
npm i
```
2. Из директории с фронтенд-приложением выполните команду:
```
npm run build
```
3. Скопируйте статику фронтенд-приложения в директорию по умолчанию:
Точка после build важна — будет скопировано содержимое директории.
```
sudo cp -r путь_к_директории_с_фронтенд-приложением/build/. /var/www/kittygram/
```
### Установка и настройка WSGI-сервера Gunicorn
1. Подключитесь к удалённому серверу, активируйте виртуальное окружение
бэкенд-приложения и установите пакет gunicorn: 
```
pip install gunicorn==20.1.0
```
2. Перейдите в директорию с файлом manage.py, и запустите Gunicorn:
```
gunicorn --bind 0.0.0.0:8000 kittygram_backend.wsgi
```
3. Создайте файл конфигурации юнита systemd для Gunicorn в директории
/etc/systemd/system/.
```
sudo nano /etc/systemd/system/gunicorn_kittygram.service
```
4. Подставьте в код из листинга свои данные, добавьте этот код без комментариев в файл конфигурации Gunicorn и сохраните изменения:
```
[Unit]
# Это текстовое описание юнита, пояснение для разработчика.
Description=gunicorn daemon
# Условие: при старте операционной системы запускать процесс только после того,
# как операционная система загрузится и настроит подключение к сети.
After=network.target
[Service]
# От чьего имени будет происходить запуск.
User=имя_пользователя_в_системе
# Путь к директории проекта.
WorkingDirectory=/home/имя_пользователя/папка_с_проектом/папка_с_файлом_manage.py/
# Команду, которую вы запускали руками, теперь будет запускать systemd.
# Чтобы узнать путь до Gunicorn, воспользуйтесь командой which gunicorn.
ExecStart=/.../venv/bin/gunicorn --bind 0.0.0.0:8000 backend.wsgi:application
[Install]
# В этом параметре указывается вариант запуска процесса.
# Значение <multi-user.target> указывает, чтобы systemd запустил процесс
# доступный всем пользователям и без графического интерфейса.
WantedBy=multi-user.target
```
Команда sudo systemctl с параметрами start, stop или restart запустит, остановит
или перезапустит Gunicorn. Например, вот команда запуска:
```
sudo systemctl start gunicorn_kittygram
```
Чтобы systemd следил за работой демона Gunicorn, запускал его при старте системы
и при необходимости перезапускал, используйте команду:
```
sudo systemctl enable gunicorn_kittygram
```
### Установка и настройка веб- и прокси-сервера Nginx
1. Установите Nginx:
```
sudo apt install nginx -y
```
2. Запустите Nginx командой:
```
sudo systemctl start nginx
```
3. Обновите настройки Nginx. Для этого откройте файл конфигурации веб-сервера…
```
sudo nano /etc/nginx/sites-enabled/default
```
…очистите содержимое файла и запишите новые настройки:
```
server { 
	listen 80;
	server_name ваш_домен; 
	
	location /api/ { 
		proxy_pass http://127.0.0.1:8000; 
		client_max_body_size 20M; 
	}
	
	location /admin/ { 
		proxy_pass http://127.0.0.1:8000; 
		client_max_body_size 20M; 
	}
		 
	location /media/ { 
		root /var/www/kittygram; 
	} 

	location / { 
		root /var/www/kittygram; 
		index index.html index.htm; 
		try_files $uri /index.html; 
	} 
}
```
Сохраните изменения в файле, закройте его и проверьте на корректность:
```
sudo nano /etc/nginx/sites-enabled/default
```
Перезагрузите конфигурацию Nginx:
```
sudo systemctl reload nginx
```
### Настройка файрвола ufw
Файрвол установит правило, по которому будут закрыты все порты, кроме тех, которые
вы явно укажете.
1. Активируйте разрешение принимать запросы только на порты 80, 443 и 22:
```
# 80, 443: с ними будут работать пользователи, делая запросы к приложению.
sudo ufw allow 'Nginx Full'
# 22: нужен, чтобы вы могли подключаться к серверу по SSH.
sudo ufw allow OpenSSH
```
2. Включите файрвол:
```
sudo ufw enable
```
В терминале выведется запрос на подтверждение операции. Введите y и нажмите Enter.
3. Проверьте работу файрвола:
```
sudo ufw status
```
### Получение и настройка SSL-сертификата

1. Установите certbot:
- Установка пакетного менеджера snap.
```
sudo apt install snapd
```
- Установка и обновление зависимостей для пакетного менеджера snap.
```
sudo snap install core; sudo snap refresh core
```
- Установка пакета certbot.
```
sudo snap install --classic certbot
```
- Обеспечение доступа к пакету для пользователя с правами администратора.
```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
2. Запустите certbot и получите SSL-сертификат. Для этого выполните команду:
```
sudo certbot --nginx
```
Далее система попросит вас указать электронную почту и ответить на несколько вопросов. Сделайте это. Следующим шагом укажите имена, для которых вы хотели бы активировать HTTPS(Введите номер нужного доменного имени и нажмите Enter).
После этого certbot отправит ваши данные на сервер Let's Encrypt и там будет выпущен
сертификат, который автоматически сохранится на вашем сервере в системной директории /etc/ssl/. Также будет автоматически изменена конфигурация Nginx: в файл
/etc/nginx/sites-enbled/default добавятся новые настройки и будут прописаны пути к сертификату.

5. Проверьте конфигурацию Nginx, и если всё в порядке, перезагрузите её.
``` 
sudo systemctl restart nginx
```

### Автор 
[Виктор Шустров](https://github.com/shustrov19/) 

### Контакты
Email: shustrov19@gmail.com
Telegram: @Shustrov19
