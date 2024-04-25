# Архитектура проекта

![Screenshot](Architecture.JPG)

# Реализация проекта


## :one: Включаем и настраиваем вложенную виртуализацию

1.) Включаем `SVM` в BIOS

2.) Удаляем компоненты Windows:
- `Win` + `R` → `Выполнить` → `appwiz.cpl` → `Enter`
- Переходим в верхний левый раздел и нажмите `Включить или отключить функции Windows`
- Далее снимаем флажок с `Hyper-V`, `Подсистема Windows для Linux` и `Платформа виртуальной машины`

3.) Отключаем безопасность на основе виртуализации:
- `Win` → `cmd` (запускаем от имени администратора)
- Вводим `bcdedit /set hypervisorlaunchtype off`

4.) Включаем вложенную виртуализацию в VMware:
- `Витруальна машина` → `Изменить настройки` → `Процессор` → ☑ Виртуальный Intel VT-x/EPT или AMD-V/RVI

---

## :two: Установка minikube
```bash
$ sudo apt update              # Обновляемся
$ sudo apt upgrade             # Обновляемся
$ sudo apt install docker.io   # Устанавливаем Docker
$ sudo apt install curl        # Устанавливаем курл
$ sudo apt install virtualbox  # Устанавливаем VirtualBox
```

#### Устанавливаем kubectl:
```bash
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"  # Загружаем исполняемый файл kubectl
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl  # Устанавливаем исполняемый файл kubectl в директории /usr/local/bin с правами доступа для запуска от root-а
```

#### Устанавливаем Minikube:
```bash
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube
$ minikube start --driver=docker --force  # Запускаем Minikube
$ minikube start --driver=virtualbox --force  # Или так, если выдало ошибку
$ minikube status  # Проверяем состояние Minikube
$ minikube stop    # Остановить кластер
```

#### Очистка локального состояния:
```bash
$ minikube start   # В таком случае команда minikube start вернёт ошибку: machine does not exist
$ minikube delete  # Чтобы исправить это, нужно очистить локальное состояние
```

#### Установка kubernetes-dashboard:
```bash
$ minikube addons enable dashboard  # Включаем модуль
$ export EDITOR=nano  # Меняем редактор VIM на NANO (чтобы по умолчанию edit использовал редактор nano)
$ kubectl edit service kubernetes-dashboard -n kube-system  # Меняем порт в сервисе, чтобы можно было зайти в UI приложения
Меняем строку type: ClusterIP на type: NodePort
$ minikube ip  # Смотрим ip minikube
$ kubectl get service -A | grep dashboard  # Смотрим порт сервиса
Вставляем в браузер:  # http://<ip_minikube>:<port_service>
```

---

#### Создание namespace "Monitoring"
```bash
$ kubectl create namespace monitoring
```

#### Cоздание манифестов для развертывания Prometheus и Grafana в новом namespace
### Prometheus
- Манифест для развертывания Prometheus
```bash
$ nano prometheus.yml
```

- Конфиг:
```yml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  ports:
    - name: web
      port: 9090
      targetPort: 9090
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-pod
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus-container
          image: prom/prometheus:latest
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-storage
              mountPath: /prometheus
      volumes:
        - name: prometheus-storage
          emptyDir: {}
```

- Запускаем сборку:
```bash
kubectl apply -f prometheus.yml
```

```bash
# Делаем чтобы через edit был редактор nano
$ export EDITOR=nano

# Выбираем сервис, который хотим открыть в Web-интерфейсе
$ kubectl get services -n monitoring

# Редактируем конфигурации сервиса (Находим строку type: ClusterIP и меняем на type: NodePort)
$ kubectl edit service prometheus-service -n monitoring

# Смотрим какой порт присвоился и вставляем minikube ip и порт сервиса в браузер
$ kubectl get service -n monitoring
```

#### Определение службы (Service)
- apiVersion: Указывает на версию API, с которой работает данный манифест.
- kind: Указывает на тип объекта Kubernetes, в данном случае - служба.
- metadata: Содержит метаданные объекта, такие как имя, пространство имен (namespace) и метки (labels).
- spec: Содержит спецификацию службы, включая селектор (selector) и порты.

#### Определение развертывания (Deployment)
- apiVersion: Указывает на версию API, с которой работает данный манифест.
- kind: Указывает на тип объекта Kubernetes, в данном случае - развертывание.
- metadata: Содержит метаданные объекта, такие как имя, пространство имен (namespace) и метки (labels).
- spec: Содержит спецификацию развертывания, включая количество реплик, селектор, шаблон пода и контейнеры.

### Grafana
- Манифест для развертывания Grafana
```bash
$ nano grafana.yml
```

- Конфиг:
```yml
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: monitoring
  labels:
    app: grafana
spec:
  selector:
    app: grafana
  ports:
    - name: web
      port: 3000
      targetPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-pod
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana-container
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
```

- Запускаем сборку:
```bash
kubectl apply -f grafana.yml
```

```bash
# Делаем чтобы через edit был редактор nano
$ export EDITOR=nano

# Выбираем сервис, который хотим открыть в Web-интерфейсе
$ kubectl get services -n monitoring

# Редактируем конфигурации сервиса (Находим строку type: ClusterIP и меняем на type: NodePort)
$ kubectl edit service grafana-service -n monitoring

# Смотрим какой порт присвоился и вставляем minikube ip и порт сервиса в браузер
$ kubectl get service -n monitoring
```
Логин и пароль по умолчанию: `admin/admin`



#### Создание namespace "Web"
```bash
$ kubectl create namespace web
```

### Nginx
- Манифест для развертывания NGINX
```bash
$ nano nginx.yml
```

- Конфиг:
```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: web
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pod
  namespace: web
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

- Запускаем сборку:
```bash
kubectl apply -f nginx.yml
```

```bash
# Делаем чтобы через edit был редактор nano
$ export EDITOR=nano

# Выбираем сервис, который хотим открыть в Web-интерфейсе
$ kubectl get services -n web

# Редактируем конфигурации сервиса (Находим строку type: ClusterIP и меняем на type: NodePort)
$ kubectl edit service nginx-service -n web

# Смотрим какой порт присвоился и вставляем minikube ip и порт сервиса в браузер
$ kubectl get service -n web
```











## :three: Запускаем несколько подов `nginx`, `mysql` и `wildfly`. Дополнительно связка `Prometheus Grafana` с помощью чартов



---

## :four: Работа с подами и контейнерами

```bash
# Проверяем состояние подов контейнеров
$ kubectl get po -A
```
```bash
# Смотрим утилизацию контейнеров
$ minikube addons enable metrics-server  # Включаем метрики сервера в кластере
$ kubectl top po  # Смотрим утилизацию по CPU и RAM
```
```bash
# Смотрим логи контейнеров
$ kubectl logs <name_pod>
```
```bash
# Подключение к контейнеру
$ kubectl exec -it nginx-pod -- bash
$ ls -ali  # Проверяем проходят ли команды
```
```bash
# Останавливаем контейнеры
$ minikube stop  # Можем остановить все (Остановка кластера)
$ kubectl delete po nginx-pod  # Остановить контейнеры выборочно
```

---

## :five: Настройка и подключение к сервису через UI
```bash
# Делаем чтобы через edit был редактор nano
$ export EDITOR=nano

# Выбираем сервис, который хотим открыть в Web-интерфейсе
$ kubectl get services -n <namespace>

# Редактируем конфигурации сервиса (Находим строку type: ClusterIP и меняем на type: NodePort)
$ kubectl edit service <name_service> -n <namespace>
```

---








## :six: Собираем свой контейнер используя UBI 7.7 в качестве базового образа
UBI 7.7 - Это образ ОС Linux фактически голый

```bash
# Задача:
1.) С установленным apache (httpd)
2.) Httpd должен запускаться при запуске контейнера
3.) Заменить содержимое домашней страницы HTTPD по умолчанию (/var/www/html/index.html)
4.) Запустить созданный контейнер
5.) Проверить доступность  curl http://....
6.) В результате должно отобразиться содержимое index.html
```

### Решение:
- Создаём `Dockerfile`
```bash
$ nano Dockerfile
```
- Пишем конфиг `Dockerfile`:
```yml
FROM registry.access.redhat.com/ubi7/ubi:7.7

# Устанавливаем apache (httpd)
RUN yum install -y httpd

# Замена содержимого домашней страницы HTTPD по умолчанию
COPY index.html /var/www/html/

# Указываем команду по умолчанию для запуска контейнера
CMD [ "httpd", "-D", "FOREGROUND" ]
```

- Создаём `index.html`
```bash
$ nano index.html
```
- Пишем конфиг `index.html`:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Добро пожаловать в мой сайт</title>
</head>
<body>
    <h1>Sber DevOps success</h1>
    <p>Это моя домашняя страница</p>
</body>
</html>
```

- Создаём наш образ
```bash
$ docker build -t my-apache-image /home/andrey/
```

- Запускаем контейнер на основе нашего образа
```bash
$ docker run -d -p 80:80 my-apache-image
```

Ну и идём в браузер `localhost:80`
Либо курлим: `curl http://localhost:80`

---



