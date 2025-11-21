# RemnaWave Ansible Automation

Автоматическое развертывание инфраструктуры RemnaWave прокси-сервера.

> **Примечание:** Этот плейбук не претендует на звание самого лучшего или простого и сделан исключительно по вкусу и пожеланию автора. Если кто-то хочет что-то изменить или улучшить - без проблем, делайте форк и адаптируйте под свои нужды.

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
git clone https://github.com/AndreyTimoschuk/remnawave-ansible.git
cd remnawave-ansible
```

### 2. Подготовка DNS записей

**ВАЖНО:** Перед началом развертывания убедитесь, что все DNS записи для ваших доменов уже добавлены и пропагатированы:

- `panel.yourdomain.com` → IP панели
- `monitoring.yourdomain.com` → IP панели (если мониторинг на панели)
- Домены для нод (если используете) → IP соответствующих нод

Без правильных DNS записей Caddy не сможет получить SSL сертификаты и развертывание завершится с ошибкой.

### 3. Конфигурация

#### Inventory (список серверов)

Отредактируйте `inventory.ini`.

**При первой настройке ноды** (когда еще не настроен SSH по ключам), нужно указать пароль для подключения:

```ini
[nodes]
node-test ansible_host=193.68.89.137 ansible_user=root ansible_ssh_pass=pA2TCnXHsJ37CK
node1 ansible_host=YOUR_NODE_IP ansible_user=root ansible_ssh_pass=YOUR_PASSWORD

[panel]
panel ansible_host=YOUR_PANEL_IP ansible_user=root ansible_ssh_pass=YOUR_PASSWORD
```

**После настройки** (когда SSH настроен на ключи и парольная авторизация отключена), уберите `ansible_ssh_pass`:

```ini
[nodes]
node-test ansible_host=193.68.89.137 ansible_user=root
node1 ansible_host=YOUR_NODE_IP ansible_user=root

[panel]
panel ansible_host=YOUR_PANEL_IP ansible_user=root
```

> **Важно:** После первого запуска плейбука парольная авторизация SSH будет отключена в целях безопасности. Убедитесь, что ваш SSH ключ добавлен в `group_vars/nodes.yml` и `group_vars/panel.yml` перед вторым запуском.

#### Создание файлов для каждого хоста

**Для каждого хоста в inventory нужно создать два файла:**

1. **`host_vars/HOSTNAME.yml`** - переменные хоста (SECRET_KEY для нод, домены для панели)
2. **`templates/caddyfiles/HOSTNAME.j2`** - Caddyfile для ноды (только для нод, не для панели)

**Пример для ноды `node-test`:**

```bash
# Создать файл переменных
cat > host_vars/node-test.yml << EOF
---
SECRET_KEY: "YOUR_SECRET_KEY_FROM_PANEL"
EOF

# Создать Caddyfile
cat > templates/caddyfiles/node-test.j2 << EOF
https://cdn.example.com {
    root * /srv/games-site
    file_server
}
EOF
```

**Пример для панели `panel`:**

```bash
# Создать файл переменных
cat > host_vars/panel.yml << EOF
---
remnawave_panel_domain: "panel.yourdomain.com"
monitoring_domain: "monitoring.yourdomain.com"
grafana_port: 3302
# ... остальные переменные из examples/host_vars-panel-example.yml
EOF
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
prometheus_remnawave_username: "admin"
prometheus_remnawave_password: "SHA512_HASH"  # echo -n 'pass' | sha512sum | cut -d' ' -f1
```

#### Переменные панели

Создайте `host_vars/panel.yml` (см. пример выше в разделе "Создание файлов для каждого хоста"):

```yaml
remnawave_panel_domain: "panel.yourdomain.com"
monitoring_domain: "monitoring.yourdomain.com"
grafana_port: 3302
# ... остальные переменные (JWT secrets, PostgreSQL, Caddy Auth)
```

#### Переменные нод

Создайте `host_vars/node1.yml` (после добавления ноды в панели):

```yaml
SECRET_KEY: "GET_FROM_PANEL_AFTER_ADDING_NODE"
```

#### Caddyfile для нод

Создайте `templates/caddyfiles/node1.j2` (см. пример выше в разделе "Создание файлов для каждого хоста"):

```caddyfile
https://cdn.example.com {
    root * /srv/games-site
    file_server
}
```

### 4. Развертывание

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

#### Безопасность

- **UFW Firewall** - настраивает правила firewall, открывает только необходимые порты (443 для HTTPS, 22 для SSH). Для панели также добавляется rate limiting на порт 443 для защиты от DDoS атак.

- **Fail2Ban** - система защиты от брутфорс атак. Автоматически блокирует IP адреса при множественных неудачных попытках входа.

- **SSH Hardening** - настраивает безопасность SSH:
  - Отключает вход по паролю (только по SSH ключам)
  - Отключает root вход по паролю
  - Настраивает таймауты и ограничения
  - Создает системного пользователя с sudo правами

- **Автоматические обновления безопасности** - настраивает автоматическую установку обновлений безопасности через `unattended-upgrades`.

#### Оптимизация системы

- **Swap** - автоматически создает swap файл на основе размера RAM (если RAM < 2GB, создается swap = RAM * 2, иначе swap = RAM).

- **BBR TCP Congestion Control** - включает BBR (Bottleneck Bandwidth and Round-trip propagation time) алгоритм для улучшения пропускной способности сети, особенно полезно для прокси.

- **sysctl оптимизации** - настраивает параметры ядра Linux для оптимизации работы прокси:
  - Увеличивает лимиты на количество открытых файлов
  - Оптимизирует TCP параметры (keepalive, buffers)
  - Настраивает параметры для работы с большим количеством соединений

- **Docker оптимизация** - настраивает Docker daemon для оптимальной работы:
  - Логирование в json-file с ротацией
  - Ограничение размера логов

- **Отключение IPv6** - отключает IPv6 на всех интерфейсах (по желанию, можно включить обратно).

#### Сервисы

- **PostgreSQL 17** - реляционная база данных для хранения всех данных RemnaWave (пользователи, ноды, подписки, статистика и т.д.). Запускается в Docker контейнере.

- **Valkey/Redis** - in-memory хранилище данных для кэширования и сессий. Используется для быстрого доступа к часто используемым данным.

- **RemnaWave Backend** - основной сервис панели управления. Предоставляет REST API для управления пользователями, нодами, подписками. Запускается на порту 3000.

- **Caddy с аутентификацией** - веб-сервер с автоматическим HTTPS и встроенной системой аутентификации:
  - Автоматически получает SSL сертификаты через Let's Encrypt
  - Обеспечивает доступ к панели через аутентификационный портал
  - Работает как reverse proxy для RemnaWave Backend
  - Может проксировать Grafana (если мониторинг на том же хосте)

- **Consul Server** - система service discovery и распределенного хранилища ключ-значение:
  - Используется для автоматического обнаружения нод
  - Хранит конфигурацию и состояние сервисов
  - Интегрируется с Prometheus для автоматического обнаружения целей мониторинга

#### Управление логами

- **Logrotate** - настраивает ротацию логов для всех системных и прикладных логов:
  - Логи Docker контейнеров
  - Логи системных сервисов
  - Логи приложений
  - Автоматическая очистка старых логов

- **Journald лимиты** - ограничивает размер логов systemd journal до 500MB для предотвращения переполнения диска.

### nodes.yml

#### Всё из panel.yml плюс:

- **RemnaNode** - Xray прокси сервер, который обрабатывает весь трафик пользователей:
  - Поддерживает различные протоколы (VLESS, VMESS, Trojan, Shadowsocks и т.д.)
  - Автоматически получает конфигурацию с панели через API
  - Работает на порту, указанном в переменной `app_port` (по умолчанию 2222)

- **Caddy (селфстил)** - веб-сервер для раздачи статического контента:
  - Используется для self-hosted страниц подписок
  - Автоматический HTTPS
  - Настраивается через `templates/caddyfiles/HOSTNAME.j2`

- **Consul Agent** - клиент Consul, который подключается к Consul Server на панели:
  - Регистрирует ноду в системе service discovery
  - Отправляет health checks
  - Позволяет Prometheus автоматически обнаруживать ноды для мониторинга

- **Node Exporter** - Prometheus экспортер для сбора системных метрик:
  - CPU, RAM, Disk usage
  - Сетевая статистика
  - Статистика процессов
  - Метрики доступны на порту 9100

- **Автообновление GEO файлов** - cron задача, которая каждое воскресенье обновляет GeoIP базы данных для RemnaNode:
  - Скачивает актуальные файлы GeoIP
  - Обновляет конфигурацию RemnaNode
  - Перезапускает RemnaNode для применения изменений

- **UFW Blacklist** - автоматическая блокировка подозрительных IP адресов:
  - Ежедневно обновляет список IP адресов сканеров РКН
  - Блокирует их через UFW
  - Защищает от сканирования и атак

### monitoring.yml

- **Prometheus** - система сбора и хранения метрик:
  - Собирает метрики со всех нод через Node Exporter
  - Собирает метрики RemnaWave через API с базовой аутентификацией
  - Автоматически обнаруживает ноды через Consul
  - Хранит метрики в течение 15 дней
  - Доступен на порту 9090

- **Grafana** - система визуализации и дашбордов:
  - Создает красивые графики и дашборды
  - Интегрируется с Prometheus как data source
  - Предустановленные дашборды для мониторинга нод и RemnaWave
  - Доступна на порту 3302 (или указанном в `grafana_port`)
  - Может быть проксирована через Caddy на панели (если мониторинг на том же хосте)

- **Node Exporter** - если мониторинг на отдельном хосте, устанавливается Node Exporter для сбора метрик самого хоста мониторинга

- **Интеграция с Consul** - Prometheus автоматически находит все ноды через Consul service discovery, не нужно вручную добавлять каждую ноду в конфигурацию

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
prometheus_remnawave_username: "admin"  # Prometheus auth
prometheus_remnawave_password: "hash"   # SHA-512 hash
```

### host_vars/panel.yml

```yaml
SECRET_KEY: "from_panel"                        # Если панель = нода
remnawave_panel_domain: "panel.example.com"     # Домен панели
monitoring_domain: "monitoring.example.com"     # Домен Grafana
grafana_port: 3302                              # Порт Grafana
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
# Шаг 1: Выберите пароль (например: openssl rand -base64 24)
# Шаг 2: Создайте hash: echo -n 'your_password' | sha512sum | cut -d' ' -f1

# Webhook secret (64 символа, только буквы и цифры)
openssl rand -base64 48 | tr -d '=+/' | cut -c1-64
```

2. **Создайте SSH ключи:**

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
cat ~/.ssh/id_rsa.pub  # Добавить в group_vars/nodes.yml
```

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
4. Импортируйте дашборды вручную:
   - Create → Import
   - Нажмите "Upload JSON file"
   - Загрузите файлы из `templates/`:
     - `Linux node _ fleet overview-1763743007941.json`
     - `Node Exporter Full-1763743021570.json`
   - В поле "Data source" выберите ваш Prometheus data source (например, "Prometheus")
   - Нажмите "Import"
   
   **Примечание:** Дашборды используют переменную `${datasource}`, поэтому при импорте вы сможете выбрать нужный Prometheus data source из списка.

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


## Примеры Caddyfile

### Простой (один домен)

```caddyfile
https://cdn.example.com {
    root * /srv/games-site
    file_server
}
```

### С кастомным портом

```caddyfile
https://cdn.example.com:8443 {
    root * /srv/games-site
    file_server
}
```

### Несколько доменов

```caddyfile
https://cdn-1.example.com:843 {
    root * /srv/games-site
    file_server
}

https://cdn-2.example.com:843 {
    root * /srv/games-site
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

**Q: Сколько RAM нужно?**

- Минимум для ноды: 1GB (будет создан swap)
- Минимум для панели: 2GB

**Q: Поддерживаются ли другие ОС?**

Тестировалось на Ubuntu 22.04/24.04. Debian должен работать с минимальными изменениями.

## Лицензия

MIT License - используйте как хотите.

---

**Сделано с ❤️ для RemnaWave Community**

