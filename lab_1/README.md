# Лабораторная работа: Настройка Nginx

## Описание

В этой лабораторной работе я настроил Nginx для работы:

- По протоколу HTTPS с сертификатами.
- С принудительным перенаправлением HTTP-запросов (порт 80) на HTTPS (порт 443).
- С использованием `alias` для создания псевдонимов путей.
- С поддержкой виртуальных хостов для обслуживания нескольких доменных имен на одном сервере.
- С двумя pet-проектами, доступными по HTTPS.

## Структура работы

1. **Установка Nginx** через Homebrew.
2. **Создание сертификатов** для HTTPS (самоподписанные).
3. **Настройка виртуальных хостов**.
4. **Настройка перенаправления с HTTP на HTTPS**.
5. **Проверка работы alias** и других конфигураций.

## Требования

- macOS.
- Homebrew.
- Nginx.
- OpenSSL (устанавливается вместе с macOS).

## Инструкция

### 1. Установка Nginx

1. Сначала я установил Homebrew:
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

2. Затем я установил Nginx через Homebrew:
   ```bash
   brew install nginx
   ```

3. Запустил Nginx:
   ```bash
   brew services start nginx
   ```

4. Убедился, что Nginx работает, открыв в браузере [http://localhost:8080/](http://localhost:8080/).

### 2. Создание структуры проектов

Я создал директории для двух pet-проектов:
```bash
mkdir -p ~/Sites/project1
mkdir -p ~/Sites/project2
```

И создал файл `index.html` в каждой папке:

**Для Project1:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Project1</title>
</head>
<body>
    <h1>Hello from Project1!</h1>
</body>
</html>
```

**Для Project2:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Project2</title>
</head>
<body>
    <h1>Hello from Project2!</h1>
</body>
</html>
```

### 3. Создание самоподписанных сертификатов

Я перешёл в рабочую директорию, например `~/Sites/certs`:
```bash
mkdir -p ~/Sites/certs
cd ~/Sites/certs
```

Создал сертификаты для каждого проекта:
```bash
openssl genrsa -out project1.local.key 2048
openssl req -new -x509 -key project1.local.key -out project1.local.crt -days 365 \
  -subj "/C=RU/ST=SomeState/L=SomeCity/O=SomeOrg/OU=IT/CN=project1.local"

openssl genrsa -out project2.local.key 2048
openssl req -new -x509 -key project2.local.key -out project2.local.crt -days 365 \
  -subj "/C=RU/ST=SomeState/L=SomeCity/O=SomeOrg/OU=IT/CN=project2.local"
```

И скопировал сертификаты в папку конфигурации Nginx:
```bash
mkdir -p /opt/homebrew/etc/nginx/ssl
cp project1.local.* /opt/homebrew/etc/nginx/ssl/
cp project2.local.* /opt/homebrew/etc/nginx/ssl/
```

### 4. Настройка виртуальных хостов

1. Открыл основной конфигурационный файл Nginx:
   ```bash
   nano /opt/homebrew/etc/nginx/nginx.conf
   ```

2. Убедился, что в блоке `http` есть директива:
   ```nginx
   include servers/*.conf;
   ```
   Если её не было, я добавил.

3. Создал директорию для серверных конфигураций:
   ```bash
   mkdir -p /opt/homebrew/etc/nginx/servers
   ```

4. Создал файл конфигурации для первого проекта:

**`/opt/homebrew/etc/nginx/servers/project1.conf`**
```nginx
server {
    listen 80;
    server_name project1.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name project1.local;

    ssl_certificate     /opt/homebrew/etc/nginx/ssl/project1.local.crt;
    ssl_certificate_key /opt/homebrew/etc/nginx/ssl/project1.local.key;

    root /Users/username/Sites/project1;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

5. Создал файл конфигурации для второго проекта:

**`/opt/homebrew/etc/nginx/servers/project2.conf`**
```nginx
server {
    listen 80;
    server_name project2.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name project2.local;

    ssl_certificate     /opt/homebrew/etc/nginx/ssl/project2.local.crt;
    ssl_certificate_key /opt/homebrew/etc/nginx/ssl/project2.local.key;

    root /Users/username/Sites/project2;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

6. Добавил записи в `/etc/hosts`:
```bash
sudo nano /etc/hosts
```

Дописываю строки:
```
127.0.0.1 project1.local
127.0.0.1 project2.local
```

7. Перезапустил Nginx:
```bash
brew services restart nginx
```

### 5. Проверка работы

1. Перешёл по адресу `http://project1.local` и убедился, что произошёл редирект на `https://project1.local`.
2. Убедился, что открывается содержимое `index.html` из папки `project1`.
![screenshot](/img/screen_1.png)
3. Повторил для `http://project2.local` и `https://project2.local`.
![screenshot](/img/screen_2.png)

### 6. Пример использования alias

Создал директорию, где будут храниться общие файлы:
```bash
mkdir -p ~/Sites/shared_assets
```

В конфигурации я добавил `alias` для указания альтернативного пути. Например, чтобы запросы к `/assets/` для `Project1` шли из директории `~/Sites/shared_assets`:

```nginx
location /assets/ {
    alias /Users/username/Sites/shared_assets/;
}
```
![screenshot](/img/screen_3.png)
---

### 7. Добавление редиректа на страницу ошибки 404
Для улучшения пользовательского опыта я добавил кастомную страницу для ошибок 404. Она будет показываться, если запрашиваемый файл не найден.

1. Создание страницы ошибки
Сначала я создал HTML-файл, который будет использоваться в качестве страницы ошибки:
```bash
echo "<h1>404 - Not Found</h1><p>Извините, страница не существует.</p>" > ~/Sites/404.html
```

2. Добавление настроек в Nginx
```nginx
server {
    ...
    error_page 404 /404.html;

    location = /404.html {
        root /Users/username/Sites;
        internal;
    }
    ...
}
```
error_page 404 /404.html; — указывает Nginx, что для ошибки 404 нужно возвращать файл /404.html.
location = /404.html { ... } — описывает обработку страницы ошибки:
root /Users/y4r1k/Sites; — путь к каталогу, где находится файл 404.html.
internal; — запрещает прямой доступ к этой странице по URL, она используется только для обработки ошибок.

3. Перезагрузка Nginx

После внесения изменений я проверил синтаксис конфигурации и перезагрузил Nginx:

```bash
brew services restart nginx
```

4. Результат

Если пользователь попытается открыть несуществующую страницу, например https://project2.local/nonexistent, он увидит кастомное сообщение:

![screenshot](/img/screen_4.png)
## Заключение

Теперь мой Nginx настроен для работы с HTTPS, виртуальными хостами, перенаправлением HTTP -> HTTPS и использованием alias для псевдонимов путей.
