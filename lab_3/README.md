# Лабораторная работа 3

---

## Шаги выполнения

### 1. Установка Minikube и запуск кластера
1. Я установил Minikube на macOS:
   - Скачал и установил Minikube:
     ```bash
     brew install minikube
     ```
   - Убедился, что установка прошла успешно:
     ```bash
     minikube version
     ```
![screenshot](/img/screen_5.png)

2. Запустил Minikube с помощью команды:
   ```bash
   minikube start
   ```
3. Убедился, что кластер работает:
   ```bash
   kubectl get nodes
   ```
![screenshot](/img/screen_6.png)
---

### 2. Создание Docker-образа для сервиса
1. Написал простой сервис на Python FastAPI:
   ```python
   from fastapi import FastAPI

   app = FastAPI()

   @app.get("/")
   def read_root():
       return {"message": "Hello, World!"}
   ```
2. Создал Dockerfile:
   ```dockerfile
   FROM python:3.11-slim
   WORKDIR /app
   COPY . /app
   RUN pip install fastapi uvicorn
   CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
   ```
3. Собрал Docker-образ:
   ```bash
   eval $(minikube docker-env)  # Переключение на локальный Minikube Docker
   docker build -t hello-world-app .
   ```
![screenshot](/img/screen_7.png)
---

### 3. Написание YAML-файлов для Kubernetes
Я создал следующие YAML-файлы:

#### Deployment (развёртывание приложения)
`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world-app
  template:
    metadata:
      labels:
        app: hello-world-app
    spec:
      containers:
      - name: hello-world-container
        image: hello-world-app
        ports:
        - containerPort: 80
```

#### Service (сервис для доступа к приложению)
`service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
spec:
  type: NodePort
  selector:
    app: hello-world-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

---

### 4. Развёртывание сервиса
1. Я применил YAML-файлы одной командой:
   ```bash
   kubectl apply -f deployment.yaml -f service.yaml
   ```
2. Убедился, что ресурсы созданы:
   ```bash
   kubectl get all
   ```
![screenshot](/img/screen_8.png)
---

### 5. Доступ к сервису
1. Получил URL для доступа к сервису:
   ```bash
   minikube service hello-world-service --url
   ```
2. Открыл полученный URL в браузере и увидел сообщение: `{"message": "Hello, World!"}`.

---

### 6. Проверка работоспособности
Для проверки работоспособности я использовал следующие команды:
- Список всех подов:
  ```bash
  kubectl get pods
  ```
- Логи пода:
  ```bash
  kubectl logs <pod-name>
  ```

---

### Заключение
Я успешно развернул локальный Kubernetes-кластер и запустил в нём свой сервис. YAML-файлы позволяют автоматизировать процесс развёртывания и повторять его на других окружениях.

