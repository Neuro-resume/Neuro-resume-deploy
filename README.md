# Neuro-resume-deploy

Ansible playbooks for deploying Neuro Resume application consisting of frontend, backend, and PostgreSQL database.

## Application Components

- **Frontend**: React/Next.js application (neuroresume/neuro-resume-frontend)
- **Backend**: FastAPI/Python application (neuroresume/neuro-resume-backend)
- **Database**: PostgreSQL 15 Alpine

## Structure

- `playbooks/`: Ansible playbooks for deployment
  - `all-delploy.yml`: Deploy all components
  - `backend-delpoy.yml`: Deploy backend and database
  - `frontend-deploy.yml`: Deploy frontend
- `roles/`: Ansible roles
  - `backend-app/`: Backend and database deployment
  - `frontend-app/`: Frontend deployment
- `inventory/`: Inventory files for different environments
- `group_vars/`: Environment-specific variables

## Environment Variables

Set the following environment variables in your GitHub repository secrets/environments:

### Database Configuration
- `DATABASE_URL`: PostgreSQL connection string

### Application Configuration
- `APP_ENV`: Application environment (development/staging/production)
- `DEBUG`: Debug mode (True/False)
- `LOG_LEVEL`: Logging level

### API Configuration
- `API_V1_PREFIX`: API prefix (/api/v1)
- `CORS_ORIGINS`: Allowed CORS origins

### JWT Authentication
- `JWT_SECRET`: JWT secret key
- `JWT_ALGORITHM`: JWT algorithm (HS256)
- `JWT_EXPIRATION`: JWT expiration time in seconds

### MCP Configuration
- `ANTHROPIC_API_KEY`: Anthropic API key
- `OPENAI_API_KEY`: OpenAI API key
- `GEMINI_API_KEY`: Google Gemini API key
- `GEMINI_MODEL`: Gemini model name

## GitHub Actions Workflows

### Production Deployment (Manual)
- **Триггер**: Только ручной запуск через кнопку "Run workflow"
- **Окружение**: production
- **Вводные параметры**:
  - `confirm_deployment`: Необходимо ввести "deploy" для подтверждения
  - `target_version`: Версия приложения (по умолчанию "latest")

### Staging Deployment (Manual)
- **Триггер**: Только ручной запуск через кнопку "Run workflow"
- **Окружение**: staging
- **Вводные параметры**: Аналогично production с префиксом STAGING_

### Ansible Syntax Test
- **Триггер**: Push или PR в `main`/`develop` branches
- **Назначение**: Проверка синтаксиса Ansible плейбуков

## Ручной деплой

1. Перейдите на вкладку **Actions** в GitHub repository
2. Выберите нужный workflow (Production или Staging)
3. Нажмите **Run workflow**
4. Введите параметры:
   - **confirm_deployment**: `deploy`
   - **target_version**: желаемая версия (или оставьте `latest`)
5. Нажмите **Run workflow**

Workflow проверит подтверждение и начнет деплой.

## Быстрый старт деплоя

1. **Перейдите в Actions**: Откройте вкладку Actions в вашем GitHub репозитории
2. **Выберите workflow**: 
   - "Deploy to Production" для продакшена
   - "Deploy to Staging" для стейджинга
3. **Запустите workflow**: Нажмите "Run workflow" в правом верхнем углу
4. **Заполните параметры**:
   - `confirm_deployment`: введите `deploy`
   - `target_version`: версия образа (по умолчанию `latest`)
5. **Подтвердите**: Нажмите зеленую кнопку "Run workflow"

Деплой начнется автоматически и вы увидите прогресс в реальном времени.

## Required GitHub Secrets

### SSH Access
- `SSH_PRIVATE_KEY`: Private SSH key for production server access
- `STAGING_SSH_PRIVATE_KEY`: Private SSH key for staging server access
- `TARGET_HOST`: Production server hostname/IP
- `STAGING_TARGET_HOST`: Staging server hostname/IP

### Ansible Vault
- `ANSIBLE_VAULT_PASSWORD`: Password for Ansible vault (same for both environments)

### Application Secrets (Production)
- `DATABASE_URL`
- `APP_ENV`
- `DEBUG`
- `LOG_LEVEL`
- `API_V1_PREFIX`
- `CORS_ORIGINS`
- `JWT_SECRET`
- `JWT_ALGORITHM`
- `JWT_EXPIRATION`
- `ANTHROPIC_API_KEY`
- `OPENAI_API_KEY`
- `GEMINI_API_KEY`
- `GEMINI_MODEL`
- `API_URL`

### Application Secrets (Staging)
Все production секреты с префиксом `STAGING_`:
- `STAGING_DATABASE_URL`
- `STAGING_APP_ENV`
- etc.

**Примечание**: Теперь деплой запускается только вручную через GitHub Actions интерфейс.

## Environments Setup

Create the following environments in your GitHub repository:

1. **production**
   - Add all production secrets
   - Configure branch protection for `main`

2. **staging**
   - Add all staging secrets with `STAGING_` prefix
   - Configure for `develop` branch

### Deploy all components
```bash
ansible-playbook -i inventory/production/ playbooks/all-delploy.yml
```

### Deploy backend only
```bash
ansible-playbook -i inventory/production/ playbooks/backend-delpoy.yml
```

### Deploy frontend only
```bash
ansible-playbook -i inventory/production/ playbooks/frontend-deploy.yml
```

## Requirements

- Ansible 2.9+
- Docker
- Docker Compose
- Python 3
- Access to target servers via SSH

## Ansible Vault Setup

For sensitive configuration, use Ansible Vault:

1. Create vault password file:
```bash
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass
```

2. Encrypt sensitive files:
```bash
ansible-vault encrypt group_vars/production/secrets.yml
ansible-vault encrypt group_vars/staging/secrets.yml
```

3. Edit encrypted files:
```bash
ansible-vault edit group_vars/production/secrets.yml
```

## Manual Deployment
