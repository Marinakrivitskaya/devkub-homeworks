# Домашнее задание к занятию "13.1 контейнеры, поды, deployment, statefulset, services, endpoints"
Настроив кластер, подготовьте приложение к запуску в нём. Приложение стандартное: бекенд, фронтенд, база данных (пример можно найти в папке 13-kubernetes-config).

## Задание 1: подготовить тестовый конфиг для запуска приложения
Для начала следует подготовить запуск приложения в stage окружении с простыми настройками. Требования:
* под содержит в себе 3 контейнера — фронтенд, бекенд, базу;
* регулируется с помощью deployment фронтенд и бекенд;
* база данных — через statefulset.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
labels:
app: netology
name: frontend-backend
spec:
replicas: 1
selector:
matchLabels:
app: netology
template:
metadata:
labels:
app: netology
spec:
- image: marinakrivitskaya/13-1-frontend:latest
name: frontend
ports:
- containerPort: 80
- image: marinakrivitskaya/13-1-backend:latest
name: backend
ports:
- containerPort: 9000

apiVersion: apps/v1
kind: StatefulSet
metadata:
name: postgresql-db
spec:
serviceName: “postgresql-db”
selector:
matchLabels:
app: postgresql-db
replicas: 1
template:
metadata:
labels:
app: postgresql-db
spec:
containers:
- name: postgresql-db
image: postgres:13-alpine
env:
- name: POSTGRES_DB
value: news
- name: POSTGRES_PASSWORD
value: postgres
- name: POSTGRES_USER
value: postgres

apiVersion: v1
kind: Service
metadata:
name: postgresql-db
spec:
selector:
app: postgresql-db
ports:
- protocol: TCP
port: 5432
targetPort: 5432
---
```

## Задание 2: подготовить конфиг для production окружения
Следующим шагом будет запуск приложения в production окружении. Требования сложнее:
* каждый компонент (база, бекенд, фронтенд) запускаются в своем поде, регулируются отдельными deployment’ами;
* для связи используются service (у каждого компонента свой);
* в окружении фронта прописан адрес сервиса бекенда;
* в окружении бекенда прописан адрес сервиса базы данных.

<img width="735" alt="Screenshot 2021-07-30 at 17 04 29" src="https://user-images.githubusercontent.com/67638098/127769239-5de0ffd1-abcb-4ae7-a61e-9128fbfb892a.png">


## Задание 3 (*): добавить endpoint на внешний ресурс api
Приложению потребовалось внешнее api, и для его использования лучше добавить endpoint в кластер, направленный на это api. Требования:
* добавлен endpoint до внешнего api (например, геокодер).

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

В качестве решения прикрепите к ДЗ конфиг файлы для деплоя. Прикрепите скриншоты вывода команды kubectl со списком запущенных объектов каждого типа (pods, deployments, statefulset, service) или скриншот из самого Kubernetes, что сервисы подняты и работают.

---
