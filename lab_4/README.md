# Лабораторная работа 4

## Плохой CI/CD файл

```yaml
name: Bad CI/CD Workflow

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Build application
      run: |
        echo "Building application..."
        docker build -t my-app .

    - name: Push to Docker Hub
      run: |
        echo "Pushing image to Docker Hub..."
        docker login -u user -p password
        docker push my-app

    - name: Deploy to production
      run: |
        echo "Deploying to production..."
        ssh user@production-server "docker pull my-app && docker run -d my-app"
```

---

## Хороший CI/CD файл

```yaml
name: Good CI/CD Workflow

on:
  push:
    branches:
      - "*"
  pull_request:
    branches:
      - "*"

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Install dependencies
      run: |
        echo "Installing dependencies..."
        pip install -r requirements.txt

    - name: Run tests
      run: |
        echo "Running tests..."
        pytest

    - name: Build application
      run: |
        echo "Building application..."
        docker build -t my-app:${{ github.sha }} .

    - name: Push to Docker Hub
      env:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      run: |
        echo "Logging in to Docker Hub..."
        echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
        docker push my-app:${{ github.sha }}

    - name: Deploy to staging
      env:
        SSH_PRIVATE_KEY: ${{ secrets.STAGING_SSH_KEY }}
      run: |
        echo "Deploying to staging..."
        echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
        ssh user@staging-server "docker pull my-app:${{ github.sha }} && docker run -d my-app:${{ github.sha }}"

    - name: Manual approval for production
      uses: hmarr/auto-approve-action@v2

    - name: Deploy to production
      env:
        SSH_PRIVATE_KEY: ${{ secrets.PRODUCTION_SSH_KEY }}
      run: |
        echo "Deploying to production..."
        echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
        ssh user@production-server "docker pull my-app:${{ github.sha }} && docker run -d my-app:${{ github.sha }}"
```

---

## Пояснения к плохим практикам и их исправлениям

### 1. **Жёстко заданная ветка**
- **Почему это плохо?** Триггер настроен только на `main`, что игнорирует PR и другие ветки.
- **Исправление:** Добавлены триггеры на все ветки и pull request.

### 2. **Отсутствие тестирования**
- **Почему это плохо?** Процесс сборки и деплоя проходит без проверки качества кода, что может привести к сбою в продакшене.
- **Исправление:** Добавлены этапы тестирования с использованием `pytest`.

### 3. **Хранение логина в явном виде**
- **Почему это плохо?** Логин и пароль в открытом виде уязвимы для атак и утечек.
- **Исправление:** Использование `GitHub Secrets` для хранения учётных данных.

### 4. **Отсутствие тегирования Docker-образов**
- **Почему это плохо?** При каждом деплое используется один и тот же тег (`latest`), что усложняет откат.
- **Исправление:** Добавлено автоматическое тегирование с использованием SHA коммита.

### 5. **Прямое выполнение деплоя на продакшн**
- **Почему это плохо?** Прямой деплой на продакшн без этапов проверки может привести к выводу из строя всей системы.
- **Исправление:** Добавлен промежуточный этап деплоя в staging и ручное подтверждение перед деплоем в продакшн.
