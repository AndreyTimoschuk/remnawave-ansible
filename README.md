# RemnaWave Ansible Automation

Автоматическое развертывание инфраструктуры RemnaWave прокси-сервера.

## Что это?

Три Ansible плейбука для полной автоматизации развертывания RemnaWave:

| Плейбук | Назначение |
|---------|-----------|
| **panel.yml** | Remnawave panel (PostgreSQL, Redis, Backend, Caddy)
| **nodes.yml** | Прокси ноды (RemnaNode, Caddy, Consul, мониторинг, селфстил)
| **monitoring.yml** | Мониторинг (Prometheus, Grafana)

## Быстрый старт

### 1. Установка

```bash
# Установить Ansible
pip install ansible

# Установить коллекции
ansible-galaxy collection install community.docker

# Клонировать репозиторий
git clone https://github.com:AndreyTimoschuk/remnawave-ansible.git
cd remnawave-ansible
```

### 2. Конфигурация

#### Inventory (список серверов)

Отредактируйте `inventory.ini`:

```ini
[nodes]
node1 ansible_host=YOUR_NODE_IP ansible_user=root
node2 ansible_host=YOUR_NODE_IP ansible_user=root

panel ansible_host=YOUR_PANEL_IP ansible_user=root
```

#### Общие переменные

Отредактируйте `group_vars/nodes.yml`:

```yaml
system_user: "admin"                    # Имя пользователя на серверах
system_user_password: "SecurePass123"   # Пароль
monitoring_ip: "YOUR_PANEL_IP"          # IP панели
consul_server_ip: "YOUR_PANEL_IP"       # IP Consul сервера (обычно = панели)
panel_hostname: "panel"                 # Имя панели из inventory
ssh_public_key: "ssh-rsa AAAAB3..."     # Ваш публичный SSH ключ
cloudflare_api_token: "YOUR_TOKEN"      # Cloudflare API токен
prometheus_remnawave_username: "admin"
prometheus_remnawave_password: "SHA512_HASH"  # echo -n 'pass' | sha512sum
```

#### Переменные панели

Отредактируйте `host_vars/panel.yml`:

```yaml
remnawave_panel_domain: "panel.yourdomain.com"
monitoring_domain: "monitoring.yourdomain.com"
```

#### Переменные нод

Создайте `host_vars/node1.yml` (после добавления ноды в панели):

```yaml
SECRET_KEY: "GET_FROM_PANEL_AFTER_ADDING_NODE"
```

#### Caddyfile для нод

Создайте `templates/caddyfiles/node1.j2`:

```caddyfile
https://cdn.example.com {
    tls {
        dns cloudflare {
            api_token {{ cloudflare_api_token }}
        }
    }
    root * /srv/static-site
    file_server
}
```

### 3. Развертывание

```bash
# Шаг 1: Развернуть панель
ansible-playbook -i inventory.ini panel.yml --limit panel

# Шаг 2: Открыть https://panel.yourdomain.com
# Зарегистрировать первого пользователя (станет админом)
# Добавить ноды и скопировать SECRET_KEY для каждой

# Шаг 3: Добавить SECRET_KEY в host_vars/node1.yml и т.д.

# Шаг 4: Развернуть ноды
ansible-playbook -i inventory.ini nodes.yml

# Шаг 5: Развернуть мониторинг
ansible-playbook -i inventory.ini monitoring.yml --limit panel

# Готово! Откройте https://monitoring.yourdomain.com
```

## Что устанавливается

### panel.yml

**Безопасность:**
- UFW firewall (только 443, SSH)
- Fail2Ban (защита от брутфорса)
- SSH hardening (только ключи, без паролей)
- Автоматические обновления безопасности

**Оптимизация:**
- Swap (автоматический расчет по RAM)
- BBR TCP congestion control
- sysctl оптимизации для прокси
- Docker оптимизация
- Отключение IPv6

**Сервисы:**
- PostgreSQL 17 (база данных)
- Valkey/Redis (кэш)
- RemnaWave Backend
- Caddy с аутентификацией
- Consul Server (service discovery)

**Управление логами:**
- Logrotate для всех логов
- Journald лимиты (500MB max)

### nodes.yml

**Всё то же + RemnaNode:**
- RemnaNode (Xray прокси)
- Caddy (селфстил)
- Consul Agent
- Node Exporter (метрики)
- Автообновление GEO файлов (каждое воскресенье)
- UFW Blacklist (блокировка сканеров РКН, ежедневно)

### monitoring.yml

- Prometheus (сбор метрик)
- Grafana (визуализация)
- Node Exporter
- Интеграция с Consul для автообнаружения нод

## Примеры использования

### Добавить новую ноду

```bash
# 1. Добавить в inventory.ini
echo "node3 ansible_host=NEW_IP ansible_user=root" >> inventory.ini

# 2. Создать Caddyfile
cp templates/caddyfiles/node1.j2 templates/caddyfiles/node3.j2
nano templates/caddyfiles/node3.j2

# 3. Добавить ноду в панели, получить SECRET_KEY

# 4. Создать host_vars
echo "---
SECRET_KEY: \"YOUR_SECRET_KEY\"" > host_vars/node3.yml

# 5. Развернуть
ansible-playbook -i inventory.ini nodes.yml --limit node3
```

### Обновить конфигурацию

```bash
# Только Caddy на ноде
ansible-playbook -i inventory.ini nodes.yml --limit node1 --tags caddy

# Только RemnaNode на всех нодах
ansible-playbook -i inventory.ini nodes.yml --tags remnanode

# Обновить панель
ansible-playbook -i inventory.ini panel.yml --limit panel --tags remnawave

# Обновить мониторинг
ansible-playbook -i inventory.ini monitoring.yml --limit panel
```

### Полезные команды

```bash
# Проверить подключение
ansible -i inventory.ini all -m ping

# Dry-run (без изменений)
ansible-playbook -i inventory.ini nodes.yml --check

# Посмотреть что будет выполнено
ansible-playbook -i inventory.ini nodes.yml --list-tasks

# Все доступные теги
ansible-playbook -i inventory.ini nodes.yml --list-tags
```

## Доступные теги

### Общие теги:

- `firewall` - Настройка UFW
- `security` - Безопасность (Fail2Ban, SSH)
- `optimization` - Оптимизация системы
- `docker` - Docker установка
- `networking` - Сетевые настройки

### nodes.yml теги:

- `remnanode` - Настройка RemnaNode
- `caddy` - Настройка Caddy
- `consul` - Consul Agent
- `monitoring` - Node Exporter
- `geo-update` - Скрипты обновления GEO

### panel.yml теги:

- `remnawave` - RemnaWave панель
- `consul` - Consul Server
- `panel-prep` - Вся подготовка панели

### monitoring.yml теги:

- `prometheus` - Prometheus
- `grafana` - Grafana
- `verification` - Проверки

## Переменные

### group_vars/nodes.yml (общие для всех)

```yaml
system_user: "admin"                    # Пользователь на серверах
system_user_password: "password"        # Пароль
app_port: 2222                          # Порт RemnaNode
monitoring_ip: "PANEL_IP"               # IP мониторинга
consul_server_ip: "PANEL_IP"            # IP Consul
panel_hostname: "panel"                 # Имя панели из inventory
ssh_public_key: "ssh-rsa ..."           # SSH ключ
cloudflare_api_token: "token"           # Cloudflare токен
prometheus_remnawave_username: "admin"  # Prometheus auth
prometheus_remnawave_password: "hash"   # SHA-512 hash
```

### host_vars/panel.yml

```yaml
SECRET_KEY: "from_panel"                        # Если панель = нода
remnawave_panel_domain: "panel.example.com"     # Домен панели
monitoring_domain: "monitoring.example.com"     # Домен Grafana
grafana_port: 3302                              # Порт Grafana
caddy_cloudflare_token_alt: "token"             # Альтернативный токен
```

### host_vars/nodeX.yml

```yaml
SECRET_KEY: "from_panel"  # Обязательно для каждой ноды
```

### panel.yml vars (опционально переопределить)

```yaml
postgres_user: "remnawave"
postgres_password: "password"
postgres_db: "remnawave"
```

## Структура проекта

```
.
├── nodes.yml                   # Плейбук для нод
├── panel.yml                   # Плейбук для панели
├── monitoring.yml              # Плейбук для мониторинга
├── inventory.ini               # Список серверов
├── ansible.cfg                 # Конфигурация Ansible
├── group_vars/
│   └── nodes.yml              # Общие переменные
├── host_vars/                 # Переменные для каждого хоста
│   ├── node1.yml
│   ├── node2.yml
│   ├── node3.yml
│   └── panel.yml
├── templates/                 # Шаблоны конфигураций
│   ├── caddyfiles/           # Caddyfile для каждого хоста
│   ├── *-compose.yml.j2      # Docker Compose файлы
│   ├── remnawave-panel.env.j2 # .env для панели
│   ├── prometheus.yml.j2     # Prometheus конфиг
│   └── ...
└── examples/                  # Примеры конфигураций
```

## Безопасность

### Перед развертыванием

1. **Сгенерируйте безопасные пароли:**

```bash
# PostgreSQL password
openssl rand -base64 24

# JWT secrets (по 64 символа hex)
openssl rand -hex 64

# SHA-512 hash для Prometheus
echo -n 'your_password' | sha512sum
```

2. **Создайте SSH ключи:**

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
cat ~/.ssh/id_rsa.pub  # Добавить в group_vars/nodes.yml
```

3. **Cloudflare API Token:**
   - Зайти на https://dash.cloudflare.com/profile/api-tokens
   - Create Token → Edit zone DNS
   - Скопировать токен

### После развертывания

```bash
# Проверить firewall
ssh root@YOUR_IP "ufw status verbose"

# Проверить Fail2Ban
ssh root@YOUR_IP "fail2ban-client status sshd"

# Проверить Docker контейнеры
ssh root@YOUR_IP "docker ps"
```

## Мониторинг

После развертывания `monitoring.yml`:

1. Откройте Grafana: https://monitoring.yourdomain.com
2. Логин: `admin` / `admin` (измените!)
3. Добавьте Prometheus data source:
   - Configuration → Data Sources → Add
   - Type: Prometheus
   - URL: `http://127.0.0.1:9090`
   - Save & Test
4. Импортируйте дашборд:
   - Create → Import → Dashboard ID: `1860`
   - Select Prometheus data source
   - Import

## Troubleshooting

### Проблема: "Permission denied (publickey)"

```bash
# Добавить SSH ключ на сервер
ssh-copy-id root@YOUR_SERVER_IP
```

### Проблема: "SECRET_KEY is not defined"

1. Развернуть панель
2. Зарегистрироваться
3. Добавить ноду в панели
4. Скопировать SECRET_KEY
5. Добавить в `host_vars/nodeX.yml`

### Проблема: Caddy не получает сертификаты

```bash
# Проверить DNS
dig yourdomain.com

# Проверить логи Caddy
ssh root@YOUR_IP "docker logs caddy"

# Проверить Cloudflare токен
# Токен должен иметь права: Zone:DNS:Edit, Zone:Zone:Read
```

### Проблема: "Port 443 already in use"

```bash
# Проверить что занимает порт
ssh root@YOUR_IP "netstat -tulpn | grep :443"

# Остановить конфликтующий сервис
ssh root@YOUR_IP "systemctl stop nginx"  # или apache2
```

### Проблема: Docker контейнеры не запускаются

```bash
# Проверить логи
ssh root@YOUR_IP "docker compose -f /opt/remnawave/docker-compose.yml logs"

# Проверить ресурсы
ssh root@YOUR_IP "df -h && free -h"
```

## Обновления

### Обновить RemnaNode на всех нодах

```bash
ansible-playbook -i inventory.ini nodes.yml --tags remnanode
```

### Обновить Caddy конфигурацию

```bash
# Отредактировать templates/caddyfiles/node1.j2
ansible-playbook -i inventory.ini nodes.yml --limit node1 --tags caddy
```

### Обновить панель

```bash
ansible-playbook -i inventory.ini panel.yml --limit panel --tags remnawave
```

### Обновить GEO файлы вручную

```bash
ssh root@YOUR_NODE_IP "/opt/remnanode/update-geo-files.sh"
```

## Автоматические задачи

После развертывания автоматически выполняются:

| Задача | Расписание | Описание |
|--------|-----------|----------|
| Обновление GEO файлов | Воскресенье 3:00 МСК | Обновление geosite, geoip, zapret.dat |
| UFW Blacklist | Ежедневно 3:00 | Блокировка сетей сканеров |
| WARP restart | Ежедневно 2:00 МСК | Перезапуск WARP интерфейса |
| Logrotate | Ежедневно | Ротация всех логов |
| Security updates | Автоматически | Обновления безопасности |

## Порты

### Панель (panel):

| Порт | Назначение | Доступ |
|------|-----------|--------|
| 443 | HTTPS (Caddy) | Интернет |
| 3000 | RemnaWave Backend | localhost |
| 5432 | PostgreSQL | localhost |
| 6379 | Redis | localhost |
| 8500 | Consul API | localhost + ноды |
| 9090 | Prometheus | localhost |
| 3302 | Grafana | localhost (через Caddy) |

### Ноды (nodes):

| Порт | Назначение | Доступ |
|------|-----------|--------|
| 443 | HTTPS (Caddy) | Интернет |
| 2222 | RemnaNode | localhost |
| 9100 | Node Exporter | Панель |
| 8500 | Consul Agent | Панель |
| SSH | SSH | Ограничено |

## Примеры Caddyfile

### Простой (один домен)

```caddyfile
https://cdn.example.com {
    tls {
        dns cloudflare {
            api_token {{ cloudflare_api_token }}
        }
    }
    root * /srv/static-site
    file_server
}
```

### С кастомным портом

```caddyfile
https://cdn.example.com:8443 {
    tls {
        dns cloudflare {
            api_token {{ cloudflare_api_token }}
        }
    }
    root * /srv/static-site
    file_server
}
```

### Несколько доменов

```caddyfile
https://cdn-1.example.com:843 {
    tls {
        dns cloudflare {
            api_token {{ cloudflare_api_token }}
        }
    }
    root * /srv/static-site
    file_server
}

https://cdn-2.example.com:843 {
    tls {
        dns cloudflare {
            api_token {{ cloudflare_api_token }}
        }
    }
    root * /srv/static-site
    file_server
}
```

## Проверка работы

### После развертывания панели

```bash
ssh root@PANEL_IP

# Проверить контейнеры
docker ps
# Должны быть: remnawave, remnawave-db, remnawave-redis, caddy-panel, consul-server

# Проверить здоровье
curl http://localhost:3000/api/health
# Должен вернуть: OK

# Проверить Consul
curl http://localhost:8500/v1/status/leader
```

### После развертывания нод

```bash
ssh root@NODE_IP

# Проверить контейнеры
docker ps
# Должны быть: remnanode, caddy, consul-agent, node_exporter

# Проверить логи RemnaNode
docker logs remnanode --tail 50

# Проверить подключение к Consul
curl http://{{ consul_server_ip }}:8500/v1/catalog/services
```

### После развертывания мониторинга

```bash
ssh root@PANEL_IP

# Проверить Prometheus
curl http://localhost:9090/-/healthy
# Должен вернуть: Prometheus is Healthy.

# Проверить targets
curl http://localhost:9090/api/v1/targets

# Проверить Grafana
curl http://localhost:3302/api/health
```

## Логи

Все логи автоматически ротируются:

```bash
# Посмотреть логи RemnaNode
ssh root@NODE_IP "tail -f /var/log/remnanode/*.log"

# Docker логи
ssh root@NODE_IP "docker logs -f remnanode"

# System logs
ssh root@NODE_IP "journalctl -f"

# Fail2Ban
ssh root@NODE_IP "journalctl -u fail2ban -f"
```

## FAQ

**Q: Можно ли развернуть панель и мониторинг на разных серверах?**

A: Да, просто укажите разные хосты в `inventory.ini`:

```ini
panel ansible_host=IP1 ansible_user=root
monitoring ansible_host=IP2 ansible_user=root
```

И запустите:
```bash
ansible-playbook -i inventory.ini panel.yml --limit panel
ansible-playbook -i inventory.ini monitoring.yml --limit monitoring
```

**Q: Как изменить пользователя?**

A: Измените `system_user` в `group_vars/nodes.yml`

**Q: Как использовать Ansible Vault для секретов?**

```bash
# Создать vault
ansible-vault create group_vars/vault.yml

# Добавить секреты в vault.yml
cloudflare_api_token: "real_token"
postgres_password: "real_password"

# Запустить с vault
ansible-playbook -i inventory.ini panel.yml --ask-vault-pass
```

**Q: Сколько RAM нужно?**

- Минимум для ноды: 1GB (будет создан swap)
- Рекомендовано для ноды: 2GB
- Минимум для панели: 2GB
- Рекомендовано для панели: 4GB

**Q: Поддерживаются ли другие ОС?**

Тестировалось на Ubuntu 22.04/24.04. Debian должен работать с минимальными изменениями.

## Лицензия

MIT License - используйте как хотите.

## Поддержка

- Issues: GitHub Issues
- Документация RemnaWave: https://remna.st/docs
- Telegram: @remnawave (если есть)

---

**Сделано с ❤️ для RemnaWave Community**

