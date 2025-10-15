 Структура репозитория Ansible-роли для деплоя приложений в контейнерах

## Общая структура проекта

```
ansible-docker-deploy/
├── README.md
├── ansible.cfg
├── requirements.yml
├── .gitignore
├── .github/
│   └── workflows/
│       ├── deploy-staging.yml
│       └── deploy-production.yml
├── inventory/
│   ├── staging/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
├── group_vars/
│   ├── all/
│   │   ├── common.yml
│   │   ├── docker.yml
│   │   └── vault.yml
│   ├── staging/
│   │   ├── main.yml
│   │   └── secrets.yml
│   └── production/
│       ├── main.yml
│       └── secrets.yml
├── host_vars/
├── playbooks/
│   ├── site.yml
│   ├── deploy-apps.yml
│   └── maintenance.yml
├── roles/
│   ├── docker-deploy/
│   ├── nginx-proxy/
│   ├── monitoring/
│   └── common/
└── tests/
    └── integration/
```

## Структура основной роли docker-deploy

```
roles/docker-deploy/
├── README.md
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── setup.yml
│   ├── deploy-compose.yml
│   ├── deploy-stack.yml
│   ├── health-check.yml
│   └── cleanup.yml
├── handlers/
│   └── main.yml
├── templates/
│   ├── docker-compose.yml.j2
│   ├── .env.j2
│   ├── nginx.conf.j2
│   └── systemd-service.j2
├── files/
│   ├── scripts/
│   │   ├── backup.sh
│   │   └── healthcheck.sh
│   └── config/
├── meta/
│   └── main.yml
└── tests/
    ├── inventory
    └── test.yml
```

## Структура роли для конкретного приложения

```
roles/web-app/
├── README.md
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── prepare-environment.yml
│   ├── deploy-database.yml
│   ├── deploy-backend.yml
│   ├── deploy-frontend.yml
│   └── configure-proxy.yml
├── handlers/
│   └── main.yml
├── templates/
│   ├── docker-compose.web-app.yml.j2
│   ├── app-config.yml.j2
│   ├── database-init.sql.j2
│   └── nginx-vhost.conf.j2
├── files/
│   ├── ssl/
│   └── static-assets/
└── meta/
    └── main.yml
```

## Содержимое ключевых файлов

### ansible.cfg
```ini
[defaults]
host_key_checking = False
inventory = inventory/
roles_path = roles/
vault_password_file = .vault_pass
remote_user = deploy
become = True
become_method = sudo
gathering = smart
fact_caching = memory

[inventory]
enable_plugins = yaml, ini

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### requirements.yml
```yaml
---
collections:
  - name: community.docker
    version: ">=3.0.0"
  - name: community.general
    version: ">=5.0.0"
  - name: ansible.posix
    version: ">=1.4.0"

roles:
  - name: geerlingguy.docker
    version: ">=6.0.0"
  - name: geerlingguy.nginx
    version: ">=3.1.0"
```

### roles/docker-deploy/defaults/main.yml
```yaml
---
# Общие параметры деплоя
docker_deploy_app_name: "myapp"
docker_deploy_environment: "production"
docker_deploy_base_path: "/opt/{{ docker_deploy_app_name }}"
docker_deploy_compose_file: "docker-compose.yml"
docker_deploy_env_file: ".env"

# Docker настройки
docker_deploy_network_name: "{{ docker_deploy_app_name }}_network"
docker_deploy_network_driver: "bridge"
docker_deploy_restart_policy: "unless-stopped"

# Backup настройки
docker_deploy_backup_enabled: true
docker_deploy_backup_path: "/backup/{{ docker_deploy_app_name }}"
docker_deploy_backup_retention_days: 7

# Health check настройки
docker_deploy_health_check_enabled: true
docker_deploy_health_check_url: "http://localhost:8080/health"
docker_deploy_health_check_timeout: 30
docker_deploy_health_check_retries: 5

# Monitoring
docker_deploy_monitoring_enabled: false
docker_deploy_logs_driver: "json-file"
docker_deploy_logs_max_size: "10m"
docker_deploy_logs_max_file: "3"

# Security
docker_deploy_user_id: 1000
docker_deploy_group_id: 1000
docker_deploy_umask: "022"
```

### roles/docker-deploy/vars/main.yml
```yaml
---
# Внутренние переменные роли
docker_deploy_temp_dir: "/tmp/{{ docker_deploy_app_name }}_deploy"
docker_deploy_backup_timestamp: "{{ ansible_date_time.epoch }}"

# Пути к файлам
docker_deploy_compose_path: "{{ docker_deploy_base_path }}/{{ docker_deploy_compose_file }}"
docker_deploy_env_path: "{{ docker_deploy_base_path }}/{{ docker_deploy_env_file }}"
docker_deploy_config_path: "{{ docker_deploy_base_path }}/config"
docker_deploy_data_path: "{{ docker_deploy_base_path }}/data"
docker_deploy_logs_path: "{{ docker_deploy_base_path }}/logs"

# Системные настройки
docker_deploy_required_packages:
  - docker.io
  - docker-compose
  - curl
  - jq
```

### roles/docker-deploy/tasks/main.yml
```yaml
---
- name: Include setup tasks
  include_tasks: setup.yml
  tags: [setup, docker-deploy]

- name: Deploy using Docker Compose
  include_tasks: deploy-compose.yml
  when: docker_deploy_method == "compose"
  tags: [deploy, docker-deploy]

- name: Deploy using Docker Stack
  include_tasks: deploy-stack.yml
  when: docker_deploy_method == "stack"
  tags: [deploy, docker-deploy]

- name: Run health check
  include_tasks: health-check.yml
  when: docker_deploy_health_check_enabled
  tags: [health-check, docker-deploy]

- name: Cleanup old resources
  include_tasks: cleanup.yml
  when: docker_deploy_cleanup_enabled | default(false)
  tags: [cleanup, docker-deploy]
```

### roles/docker-deploy/tasks/setup.yml
```yaml
---
- name: Create application directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ docker_deploy_user_id }}"
    group: "{{ docker_deploy_group_id }}"
    mode: '0755'
  loop:
    - "{{ docker_deploy_base_path }}"
    - "{{ docker_deploy_config_path }}"
    - "{{ docker_deploy_data_path }}"
    - "{{ docker_deploy_logs_path }}"
  become: yes

- name: Create Docker network
  community.docker.docker_network:
    name: "{{ docker_deploy_network_name }}"
    driver: "{{ docker_deploy_network_driver }}"
    state: present

- name: Install required packages
  package:
    name: "{{ docker_deploy_required_packages }}"
    state: present
  become: yes

- name: Create backup directory
  file:
    path: "{{ docker_deploy_backup_path }}"
    state: directory
    mode: '0755'
  when: docker_deploy_backup_enabled
```

### roles/docker-deploy/tasks/deploy-compose.yml
```yaml
---
- name: Generate docker-compose file
  template:
    src: docker-compose.yml.j2
    dest: "{{ docker_deploy_compose_path }}"
    owner: "{{ docker_deploy_user_id }}"
    group: "{{ docker_deploy_group_id }}"
    mode: '0644'
    backup: yes
  notify: restart docker compose services

- name: Generate environment file
  template:
    src: .env.j2
    dest: "{{ docker_deploy_env_path }}"
    owner: "{{ docker_deploy_user_id }}"
    group: "{{ docker_deploy_group_id }}"
    mode: '0600'
    backup: yes
  notify: restart docker compose services

- name: Pull Docker images
  community.docker.docker_compose:
    project_src: "{{ docker_deploy_base_path }}"
    pull: yes
  when: docker_deploy_pull_images | default(true)

- name: Deploy application with Docker Compose
  community.docker.docker_compose:
    project_src: "{{ docker_deploy_base_path }}"
    state: present
    remove_orphans: yes
    timeout: "{{ docker_deploy_timeout | default(300) }}"
  register: docker_deploy_result

- name: Display deployment result
  debug:
    var: docker_deploy_result
    verbosity: 1
```

### roles/docker-deploy/templates/docker-compose.yml.j2
```yaml
version: '3.8'

networks:
  {{ docker_deploy_network_name }}:
    external: true

services:
{% for service in docker_deploy_services %}
  {{ service.name }}:
    image: {{ service.image }}
    container_name: {{ docker_deploy_app_name }}_{{ service.name }}
    restart: {{ docker_deploy_restart_policy }}
    networks:
      - {{ docker_deploy_network_name }}
{% if service.ports is defined %}
    ports:
{% for port in service.ports %}
      - "{{ port }}"
{% endfor %}
{% endif %}
{% if service.volumes is defined %}
    volumes:
{% for volume in service.volumes %}
      - {{ volume }}
{% endfor %}
{% endif %}
{% if service.environment is defined %}
    environment:
{% for env_var in service.environment %}
      {{ env_var.name }}: {{ env_var.value }}
{% endfor %}
{% endif %}
{% if service.depends_on is defined %}
    depends_on:
{% for dep in service.depends_on %}
      - {{ dep }}
{% endfor %}
{% endif %}
    logging:
      driver: {{ docker_deploy_logs_driver }}
      options:
        max-size: {{ docker_deploy_logs_max_size }}
        max-file: "{{ docker_deploy_logs_max_file }}"
{% if service.healthcheck is defined %}
    healthcheck:
      test: {{ service.healthcheck.test }}
      interval: {{ service.healthcheck.interval | default('30s') }}
      timeout: {{ service.healthcheck.timeout | default('10s') }}
      retries: {{ service.healthcheck.retries | default(3) }}
      start_period: {{ service.healthcheck.start_period | default('60s') }}
{% endif %}

{% endfor %}

{% if docker_deploy_volumes is defined %}
volumes:
{% for volume in docker_deploy_volumes %}
  {{ volume.name }}:
{% if volume.driver is defined %}
    driver: {{ volume.driver }}
{% endif %}
{% if volume.driver_opts is defined %}
    driver_opts:
{% for key, value in volume.driver_opts.items() %}
      {{ key }}: {{ value }}
{% endfor %}
{% endif %}
{% endfor %}
{% endif %}
```

### roles/docker-deploy/handlers/main.yml
```yaml
---
- name: restart docker compose services
  community.docker.docker_compose:
    project_src: "{{ docker_deploy_base_path }}"
    state: present
    restarted: yes
  listen: "restart docker compose services"

- name: reload systemd daemon
  systemd:
    daemon_reload: yes
  become: yes

- name: restart nginx
  service:
    name: nginx
    state: restarted
  become: yes
```

### playbooks/site.yml
```yaml
---
- name: Deploy applications infrastructure
  hosts: all
  become: yes
  roles:
    - common
    - docker

- name: Deploy web applications
  hosts: web_servers
  become: yes
  vars:
    docker_deploy_method: "compose"
    docker_deploy_services:
      - name: webapp
        image: "{{ app_image }}:{{ app_version }}"
        ports:
          - "8080:8080"
        environment:
          - name: DATABASE_URL
            value: "{{ database_url }}"
          - name: API_KEY
            value: "{{ api_key }}"
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
        volumes:
          - "{{ docker_deploy_data_path }}/webapp:/app/data"
          - "{{ docker_deploy_logs_path }}/webapp:/app/logs"
  roles:
    - docker-deploy

- name: Deploy monitoring stack
  hosts: monitoring
  become: yes
  vars:
    docker_deploy_app_name: "monitoring"
    docker_deploy_services:
      - name: prometheus
        image: "prom/prometheus:latest"
        ports:
          - "9090:9090"
        volumes:
          - "{{ docker_deploy_config_path }}/prometheus.yml:/etc/prometheus/prometheus.yml:ro"
      - name: grafana
        image: "grafana/grafana:latest"
        ports:
          - "3000:3000"
        environment:
          - name: GF_SECURITY_ADMIN_PASSWORD
            value: "{{ grafana_admin_password }}"
  roles:
    - docker-deploy
```

### group_vars/all/docker.yml
```yaml
---
# Docker общие настройки
docker_edition: "ce"
docker_packages:
  - "docker-{{ docker_edition }}"
  - "docker-{{ docker_edition }}-cli"
  - "containerd.io"

# Сетевые настройки
docker_daemon_options:
  log-driver: "json-file"
  log-opts:
    max-size: "10m"
    max-file: "3"
  storage-driver: "overlay2"

# Пользователи Docker
docker_users:
  - "{{ ansible_user }}"
  - deploy

# Очистка
docker_prune_enabled: true
docker_prune_schedule: "0 2 * * 0"  # Каждое воскресенье в 2:00
```

### .github/workflows/deploy-production.yml
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ secrets.TARGET_HOST }} >> ~/.ssh/known_hosts
      
      - name: Install Ansible
        run: |
          sudo apt update
          sudo apt install -y ansible
          ansible-galaxy install -r requirements.yml
      
      - name: Deploy applications
        run: |
          ansible-playbook \
            -i inventory/production/hosts.yml \
            --private-key ~/.ssh/id_rsa \
            playbooks/site.yml \
            --limit production
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
          APP_IMAGE: ${{ secrets.APP_IMAGE }}
          APP_VERSION: ${{ secrets.APP_VERSION }}
          GRAFANA_ADMIN_PASSWORD: ${{ secrets.GRAFANA_ADMIN_PASSWORD }}
```

## Преимущества такой структуры

1. **Модульность**: Каждая роль отвечает за свою область ответственности
2. **Переиспользование**: Роли можно использовать в разных проектах
3. **Тестируемость**: Легко тестировать отдельные компоненты
4. **Масштабируемость**: Просто добавлять новые приложения и окружения
5. **Безопасность**: Секреты изолированы по окружениям
6. **CI/CD готовность**: Интеграция с GitHub Actions из коробки