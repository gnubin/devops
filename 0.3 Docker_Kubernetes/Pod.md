## Работа с Pods
***

**Pod** — это минимальная и самая простая единица в объектной модели Kubernetes. Это абстракция над одним или несколькими контейнерами, которые:

- Разделяют одно сетевое пространство (один IP-адрес)
- Имеют общие тома хранилища (volumes)
- 1 под запускается на 1 ноде (контейнеры в поде не могут быть распределены между разными узлами)

Чаще всего используются single pods (на 1 контейнер)

apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80

Также бывает использование мультиконтейнерных подов с сайдкар (sidecar) -контейнером. Это может быть какой-то агент для сбора логов.

apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  - name: log-agent
    image: fluentd
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  volumes:
  - name: shared-logs
    emptyDir: {}

Существуют также так называемые Init-контейнеры, они запускаются до запуска основных (например, выполняют какую-то команду)

apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  containers:
  - name: app
    image: my-app:latest
  initContainers:
  - name: db-migrate
    image: postgres-client
    command: ['sh', '-c', 'until pg_isready -h db; do sleep 2; done; psql -h
     db -U user -d app -f /migrations/init.sql']

[[Жизненный цикл Pod]]

**Pending**

Pod создан, но хотя бы 1 контейнер еще не запущены

**Running**

Pod привязан к ноде, все контейнеры созданы и работают

**Succeeded**

Все контейнеры завершились успешно (для Jobs)

**Failed**

Хотя бы один контейнер завершился с ошибкой

**Unknown**

Состояние Pod'a невозможно определить

**Terminating**

Запущено удаление Pod'а

Подробнее - [https://habr.com/ru/companies/T1Holding/articles/909372/](https://habr.com/ru/companies/T1Holding/articles/909372/)

***
#devops #k8s 