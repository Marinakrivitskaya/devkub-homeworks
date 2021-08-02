# Домашнее задание к занятию "13.2 разделы и монтирование"
Приложение запущено и работает, но время от времени появляется необходимость передавать между бекендами данные. А сам бекенд генерирует статику для фронта. Нужно оптимизировать это.
Для настройки NFS сервера можно воспользоваться следующей инструкцией (производить под пользователем на сервере, у которого есть доступ до kubectl):
* установить helm: curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
* добавить репозиторий чартов: helm repo add stable https://charts.helm.sh/stable && helm repo update
* установить nfs-server через helm: helm install nfs-server stable/nfs-server-provisioner

В конце установки будет выдан пример создания PVC для этого сервера.

## Задание 1: подключить для тестового конфига общую папку
В stage окружении часто возникает необходимость отдавать статику бекенда сразу фронтом. Проще всего сделать это через общую папку. Требования:
* в поде подключена общая папка между контейнерами (например, /static);
* после записи чего-либо в контейнере с беком файлы можно получить из контейнера с фронтом.

``` yml
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
        containers:
        - image: marinakrivitskaya/13-1-frontend:latest 
          name: frontend
          ports:
          - containerPort: 80
          volumeMounts:
          - mountPath: /static
            name: test

        - image: marinakrivitskaya/13-1-backend:latest 
          name: backend
          ports:
          - containerPort: 9000
          volumeMounts:
          - mountPath: /static
            name: test

        volumes:
        - name: test
          emptyDir: {}
```
<img width="765" alt="Screenshot 2021-08-02 at 18 06 02" src="https://user-images.githubusercontent.com/67638098/127891465-f22f5d9d-5790-40df-b0fe-4af011571900.png">

## Задание 2: подключить общую папку для прода
Поработав на stage, доработки нужно отправить на прод. В продуктиве у нас контейнеры крутятся в разных подах, поэтому потребуется PV и связь через PVC. Сам PV должен быть связан с NFS сервером. Требования:
* все бекенды подключаются к одному PV в режиме ReadWriteMany;
* фронтенды тоже подключаются к этому же PV с таким же режимом;
* файлы, созданные бекендом, должны быть доступны фронту.


pvc-volume.yml
``` yml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: static-pvc
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
```
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - image: marinakrivitskaya/13-1-frontend:latest
        name: frontend
        ports:
        - containerPort: 80
        env:
          - name: BASE_URL
            value: http://backend:9000
        volumeMounts:
        - mountPath: /static
          name: pv

      volumes:
      - name: pv
        persistentVolumeClaim:
          claimName: static-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: backend
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - image: marinakrivitskaya/13-1-backend:latest
        name: frontend
        ports:
        - containerPort: 9000
        env:
          - name: DATABASE_URL
            value: postgres://postgres:postgres@db:5432/news
                    volumeMounts:
        - mountPath: /static
          name: pv

      volumes:
      - name: pv
        persistentVolumeClaim:
          claimName: static-pvc
---
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
        ports:
        - containerPort: 5432
        env:
          - name: POSTGRES_DB
            value: news
          - name: POSTGRES_PASSWORD
            value: postgres
          - name: POSTGRES_USER
            value: postgres
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 9000
      targetPort: 9000
---
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
```      

      


