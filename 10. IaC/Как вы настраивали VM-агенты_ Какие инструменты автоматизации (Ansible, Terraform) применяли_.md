Отличный практический вопрос! Он позволяет показать не только знание инструментов, но и понимание принципов инфраструктуры как кода (IaC) и управления конфигурацией.

Вот развернутый ответ, основанный на реалистичном опыте.

---

### Общая философия подхода

"Мы никогда не настраивали агенты вручную на каждой машине. Это противоречит принципам воспроизводимости, масштабируемости и надежности. Весь процесс был автоматизирован с помощью двух ключевых инструментов, которые решают разные задачи:

1.  **Terraform** — для ** provisioning** (подготовки инфраструктуры и "базовой" настройки).
2.  **Ansible** — для **configuration management** (непосредственной установки и тонкой настройки агентов).

Это классическая и очень эффективная связка."

---

### 1. Provisioning с помощью Terraform

**Задача Terraform:** Убедиться, что на создаваемой виртуальной машине есть всё необходимое для дальнейшей настройки Ansible. Заложить "фундамент".

**Что мы делали в Terraform-конфигурации (на примере Yandex Cloud; для AWS/Azure логика аналогична):**

```hcl
# 1. Создаем ВМ
resource "yandex_compute_instance" "app_server" {
  name        = "app-server-${count.index}"
  count       = 3
  platform_id = "standard-v2"
  # ... (остальные параметры ВМ)

  # 2. Важно: Заранее добавляем SSH-ключ для доступа
  metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_rsa.pub")}"
  }

  # 3. Создаем и добавляем теги / метки
  labels = {
    role      = "application"
    env       = "production"
    component = "backend"
  }

  # 4. (Опционально) Заранее добавляем скрипт для бутстрапа
  metadata = {
    user-data = <<-EOF
      #!/bin/bash
      apt-get update
      apt-get install -y python3 # Критично: Ansible требует Python на целевой машине
    EOF
  }
}

# 5. Генерируем динамический инвентарный файл для Ansible
resource "local_file" "ansible_inventory" {
  content = templatefile("inventory.tmpl", {
    app_servers = yandex_compute_instance.app_server[*].network_interface[0].nat_ip_address
  })
  filename = "../ansible/inventory/production.yml"
}
```

**Шаблон `inventory.tmpl`**:
```yaml
all:
  hosts:
    %{ for ip in app_servers ~}
    ${ip}:
    %{ endfor ~}
  children:
    app_servers:
      hosts:
        %{ for ip in app_servers ~}
        ${ip}:
        %{ endfor ~}
```

**Итог работы Terraform:** У нас есть: 1) созданные ВМ, 2) доступ по SSH, 3) установленный Python, 4) актуальный Ansible-инвентарь с IP-адресами новых машин.

---

### 2. Установка и настройка агентов с помощью Ansible

**Задача Ansible:** Установить нужный агент (например, Filebeat) и настроить его конфигурацию в зависимости от роли сервера.

**Структура Ansible-проекта:**
```
ansible/
├── inventory/
│   └── production.yml (сгенерирован Terraform)
├── roles/
│   ├── filebeat/          # Роль для установки Filebeat
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── filebeat.yml.j2  # Шаблон конфига
│   │   └── vars/
│   │       └── main.yml
│   └── node_exporter/     # Роль для установки агента для мониторинга
├── site.yml               # Главный плейбук
└── group_vars/
    └── all.yml            # Общие переменные (например, адрес Logstash)
```

**а) Роль для Filebeat (`roles/filebeat/tasks/main.yml`):**

```yaml
---
- name: Install Filebeat apt key
  apt_key:
    url: "https://artifacts.elastic.co/GPG-KEY-elasticsearch"
    state: present

- name: Add Filebeat repository
  apt_repository:
    repo: "deb https://artifacts.elastic.co/packages/7.x/apt stable main"
    state: present
    filename: "elastic-7.x"

- name: Install Filebeat package
  apt:
    name: filebeat
    state: present
    update_cache: yes

- name: Copy configured Filebeat config
  template: # Используем шаблон Jinja2 для подстановки переменных
    src: filebeat.yml.j2
    dest: /etc/filebeat/filebeat.yml
    owner: root
    group: root
    mode: '0644'
  notify: restart filebeat

- name: Enable and start Filebeat service
  systemd:
    name: filebeat
    state: started
    enabled: yes

- name: Enable system module
  command: filebeat modules enable system
  args:
    creates: /etc/filebeat/modules.d/system.yml

```

**б) Шаблон конфига Filebeat (`roles/filebeat/templates/filebeat.yml.j2`):**
```yaml
filebeat.inputs:
- type: filestream
  id: system-logs
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/messages
    - /var/log/syslog

# Переменная 'logstash_host' задается в group_vars/all.yml
output.logstash:
  hosts: ["{{ logstash_host }}:5044"]

# Добавляем теги из меток ВМ (переданные из Terraform через инвентарь)
tags: [{% for tag in ansible_labels %}"{{ tag }}"{% if not loop.last %}, {% endif %}{% endfor %}]
fields:
  env: "{{ ansible_labels.env | default('unknown') }}"
  role: "{{ ansible_labels.role | default('unknown') }}"
```

**в) Главный плейбук (`site.yml`):**

```yaml
---
- name: Apply common configuration to all servers
  hosts: all
  become: yes
  roles:
    - role: node_exporter  # Ставим агент для мониторинга на все машины

- name: Configure application servers
  hosts: app_servers       # Группа, сгенерированная Terraform
  become: yes
  roles:
    - role: filebeat       # Ставим и настраиваем Filebeat
```

**Запуск:** После работы Terraform мы переходим в папку `ansible` и выполняем:
```bash
ansible-playbook -i inventory/production.yml site.yml
```

---

### 3. Дополнительные практики

*   **Использование Dynamic Inventory для облаков:** Для больших инфраструктур вместо генерации статического файла мы использовали **dynamic inventory скрипты** (например, `yandex-cloud.yml` для Yandex Cloud или `aws_ec2.py` для AWS). Ansible сам запрашивает API облака и получает актуальный список всех ВМ и их меток.
*   **Переменные на основе тегов:** В шаблоны конфигов (Jinja2) мы передавали метки ВМ (например, `role: application`) в качестве переменных Ansible. Это позволяло гибко настраивать поведение агента в зависимости от роли сервера.
*   **Пакетирование в образы (Packer):** Для еще большей скорости и стабильности мы использовали **HashiCorp Packer** для создания предварительно настроенных образов ВМ. В образ уже "запекались" базовые компоненты (Python, сам агент), а Ansible использовался только для финальной, изменчивой конфигурации (например, указания адреса сервера мониторинга).

### Итог для собеседования

"Мы использовали принцип **"Terraform + Ansible"**:
1.  **Terraform** создавал инфраструктуру, проставлял метки и генерировал инвентарь.
2.  **Ansible** брал этот инвентарь и выполнял идемпотентную установку и настройку агентов (Filebeat, Node Exporter) на всех целевых ВМ.

Ключевые преимущества такого подхода:
*   **Воспроизводимость:** Конфигурация агента описана в коде (Ansible role) и применяется одинаково на всех серверах.
*   **Масштабируемость:** Добавили 100 новых ВМ -> Terraform создал их и обновил инвентарь -> один запуск `ansible-playbook` настраивает всё.
*   **Гибкость:** С помощью тегов и групп в инвентаре мы могли по-разному настраивать агенты на разных типах серверов (например, на app-серверах собирать логи приложения, на db-серверах — логи СУБД).
*   **Контроль версий:** Весь процесс хранится в Git, что позволяет отслеживать изменения, проводить code review и легко откатываться."