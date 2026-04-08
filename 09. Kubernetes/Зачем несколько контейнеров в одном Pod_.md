***
Для вспомогательных задач. Классический пример: Pod с основным контейнером `web-server` и боккар-контейнером (sidecar) `log-shipper`, который отправляет логи основного контейнера в централизованное хранилище.
  
  
  **Пример `pod.yaml`:**
```yaml
    apiVersion: v1
    kind: Pod # Тип объекта - Pod
    metadata:
      name: my-simple-pod # Уникальное имя Pod'а
    spec:
      containers:
      - name: nginx-container # Имя контейнера внутри Pod'а
        image: nginx:latest # Образ для запуска
        ports:
        - containerPort: 80 # Порт, который слушает контейнер
    ```
***
#devops #k8s 