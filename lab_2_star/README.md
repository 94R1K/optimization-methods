# Лабораторная работа 2 со звездочкой

## Плохой Docker Compose

```yaml
version: "3.9"

services:
  app:
    image: myapp:latest
    ports:
      - "8080:80"
    environment:
      - DEBUG=True
    volumes:
      - ./app:/app
  database:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD=root
      MYSQL_DATABASE=app_db
    ports:
      - "3306:3306"
    volumes:
      - /var/lib/mysql:/var/lib/mysql
```

#### Ошибки
1. **Использование тега `latest`**: приводит к нестабильности при изменении образа.
2. **Открытый порт `3306`**: база данных доступна извне.
3. **Секреты указаны в явном виде**: пароль базы данных хранится в открытом виде.

## Хороший Docker Compose

```yaml
version: "3.9"

services:
  app:
    image: myapp:1.0.0
    ports:
      - "127.0.0.1:8080:80"
    environment:
      - DEBUG=False
    volumes:
      - ./app:/app:ro
  database:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
      MYSQL_DATABASE: app_db
    secrets:
      - mysql_root_password
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password
```

#### Исправления
1. **Фиксация версий образов**: добавлены конкретные теги `myapp:1.0.0` и `mysql:8.0`.
2. **Ограничение доступа к портам**: доступ к портам только с локального хоста.
3. **Использование `secrets`**: секреты вынесены в отдельные файлы.

---

### Изоляция контейнеров в сети

```yaml
version: "3.9"

services:
  app:
    image: myapp:1.0.0
    networks:
      - isolated
    ports:
      - "127.0.0.1:8080:80"
  database:
    image: mysql:8.0
    networks:
      - isolated
    secrets:
      - mysql_root_password

networks:
  isolated:
    internal: true

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password
```

#### Объяснение
1. Сеть `isolated` с параметром `internal: true` изолирует контейнеры от внешнего мира.
2. Контейнеры могут взаимодействовать только внутри сети.

---

#### Изоляция сети
**Как это работает**:  
- Внутренняя сеть `internal` изолирует контейнеры от внешнего мира.  
- Контейнеры видят только друг друга, если находятся в одной сети.
