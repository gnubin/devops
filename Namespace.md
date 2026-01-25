***
Отлично! **Namespace (пространства имён)** и **Labels (метки)** — это две фундаментальные системы организации и выбора объектов в Kubernetes. Они решают разные, но дополняющие друг друга задачи.

---

### Namespace (Пространство имён)

#### Что это?

**Namespace** — это виртуальный кластер внутри физического кластера Kubernetes. Представьте, что это **изолированные "комнаты" или "окружения"** внутри одного большого "здания" (кластера).

#### Для чего нужно?

1. **Изоляция ресурсов:** Основная цель — разделить кластер между несколькими пользователями или командами (например, `team-frontend`, `team-backend`), или разделить среды (`dev`, `staging`, `production`).
    
2. **Ограничение доступа (через RBAC):** Вы можете настроить права доступа так, чтобы пользователи из Namespace `dev` не могли даже видеть ресурсы в Namespace `production`.
    
3. **Разделение ресурсов (через ResourceQuota):** Вы можете ограничить количество CPU, RAM или Pod'ов, которые может использовать каждый Namespace. Это не позволяет одной команде исчерпать все ресурсы кластера.
    
4. **Изоляция имен:** Объекты с одинаковыми именами могут существовать в разных Namespace. Например, у вас может быть Deployment `user-service` в Namespace `dev` и другой Deployment `user-service` в Namespace `staging`.
    

#### Как и где используется?

- **Команды для работы:**
    
    bash
    
    # Показать все пространства имён
    kubectl get namespaces
    kubectl get ns # Краткая форма
    
    # Создать namespace
    kubectl create namespace my-namespace
    
    # Показать все Pod'ы в определённом namespace
    kubectl get pods -n my-namespace
    kubectl get pods --namespace=my-namespace
    
    # Применить манифест к конкретному namespace
    kubectl apply -f my-deploy.yaml -n my-namespace
    
    # Установить namespace по умолчанию для текущего контекста (очень удобно!)
    kubectl config set-context --current --namespace=my-namespace
    
- **В манифестах YAML:** Namespace указывается в метаданных объекта.
    
    yaml
    
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
      namespace: my-namespace # Указание namespace
    spec:
      containers:
      - name: nginx
        image: nginx
    
- **Сетевые взаимодействия:** Сервисы (Service) внутри одного namespace могут обращаться друг к другу по короткому имени (`my-service`). Для доступа к сервису из другого namespace нужно использовать полное доменное имя (FQDN): `my-service.my-namespace.svc.cluster.local`.
    

**Важно:** Не все объекты находятся в namespace (например, PersistentVolume, Namespace本身, Node). Это объекты уровня кластера. Проверить можно командой `kubectl api-resources --namespaced=false`.
***
#devops #k8s 