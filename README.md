# RemnaWave Ansible Automation

Автоматическое развертывание инфраструктуры RemnaWave.

> **Примечание:** Этот плейбук не претендует на звание самого лучшего или простого и сделан исключительно по вкусу и пожеланию автора. Если кто-то хочет что-то изменить или улучшить - без проблем, делайте форк и адаптируйте под свои нужды.

## Что это?

Три Ansible плейбука для полной автоматизации развертывания RemnaWave:

| Плейбук | Назначение |
|---------|-----------|
| **panel.yml** | Remnawave panel (PostgreSQL, Redis, Backend, Caddy)
| **nodes.yml** | Прокси ноды (RemnaNode, Caddy, Consul, мониторинг, селфстил)
| **monitoring.yml** | Мониторинг (Prometheus, Grafana)

## Ключевые особенности

### Автоматическое обнаружение нод через Consul

Не нужно вручную добавлять каждую ноду в конфигурацию мониторинга. Consul автоматически регистрирует все ноды, а Prometheus их находит сам. Добавили новую ноду? Она сразу появится в мониторинге без дополнительных настроек.

**Как это работает:**
- На панели запускается Consul Server
- На каждой ноде запускается Consul Agent, который подключается к серверу
- Нода регистрируется в Consul со своим именем и IP
- Prometheus автоматически находит все ноды через Consul и начинает собирать метрики

### Self-hosted страницы подписок (селфстил)

**Как это работает:**
- На каждой ноде запускается Caddy веб-сервер
- Вы настраиваете Caddyfile для ноды (например, `templates/caddyfiles/usa.j2`)
- Caddy раздает статические файлы из указанной директории (например, `/srv/games-site`)
- Автоматический HTTPS через Let's Encrypt
- Можно использовать свой домен для каждой ноды

### Полноценный мониторинг из коробки

Сразу после развертывания у вас будет рабочий мониторинг с красивыми графиками.

**Что включено:**
- **Prometheus** - собирает метрики со всех нод и панели
- **Grafana** - визуализация с готовыми дашбордами
- **Node Exporter** - системные метрики (CPU, RAM, диск, сеть) с каждой ноды
- **Метрики RemnaWave** - статистика пользователей, трафика, нод прямо из панели
- Автоматическое обнаружение нод - не нужно вручную добавлять каждую ноду

**Готовые дашборды:**
- Мониторинг всех нод в одном месте
- Детальная статистика по каждой ноде
- Метрики RemnaWave (пользователи, трафик, подписки)

### Автоматическая безопасность

Плейбук настраивает базовую безопасность без лишних телодвижений.

**Что делается:**
- **UFW Firewall** - открываются только нужные порты (443 для HTTPS, 22 для SSH. На remnanode доступ до ноды по порту 2222 (или какой укажете) будет только с хоста remnawave)
- **Fail2Ban** - автоматически блокирует IP при попытках брутфорса
- **SSH Hardening** - отключается вход по паролю, только по ключам
- **Rate Limiting** - защита от DDoS на порту 443 для панели
- **UFW Blacklist** - автоматическая блокировка сканеров РКН
- Автоматические обновления безопасности

### Оптимизация производительности

Все настройки для максимальной производительности прокси уже включены.

**Что оптимизируется:**
- **BBR** - алгоритм для лучшей пропускной способности сети
- **sysctl** - настройки ядра Linux для работы с большим количеством соединений
- **Docker** - оптимизация для контейнеров
- **Swap** - автоматический расчет размера подкачки
- Увеличенные лимиты на открытые файлы и соединения

### Управление логами

Логи не засоряют диск - все автоматически ротируется и очищается.

**Что настроено:**
- **Logrotate** - ротация всех логов (Docker, системные, приложения)
- **Journald лимиты** - ограничение размера системных логов до 500MB
- Автоматическая очистка старых логов
- Сжатие старых логов для экономии места

### Гибкая настройка

Все важные параметры вынесены в переменные - легко настроить под себя.

**Что можно настроить:**
- Домены для панели и нод
- Порты приложений
- Пароли и секреты (с командами для генерации)
- Caddyfile для каждой ноды отдельно
- Параметры мониторинга

### Автоматические обновления

Некоторые вещи обновляются сами без вашего участия.

**Что обновляется автоматически:**
- **GEO файлы для RemnaNode** - каждое воскресенье
- **UFW Blacklist** - ежедневно обновляется список блокируемых IP
- **Обновления безопасности** - через unattended-upgrades

## Быстрый старт

### 1. Установка

> **Примечание для Windows:** Ansible не работает на Windows напрямую. Если вы используете Windows, запустите виртуальную машину с Linux или используйте WSL2 (Windows Subsystem for Linux).

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

**ВАЖНО:** Перед началом развертывания убедитесь, что все DNS записи для ваших доменов уже добавлены:

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

## Подробный мануал на примере usa и panel-usa

Вот пошаговая инструкция как развернуть RemnaWave с нуля на примере ноды `usa` и панели `panel-usa`. Все команды можно копировать и выполнять.

### Шаг 1: Установка Ansible

Если у вас еще нет Ansible, установите его:

> **Примечание для Windows:** Ansible не работает на Windows напрямую. Если вы используете Windows, запустите виртуальную машину с Linux или используйте WSL2.

```bash
# На macOS
brew install ansible

# На Linux (Ubuntu/Debian)
sudo apt update
sudo apt install python3-pip
pip3 install ansible

# Установить коллекцию для работы с Docker
ansible-galaxy collection install community.docker
```

### Шаг 2: Создание SSH ключа

Если у вас еще нет SSH ключа, создайте его:

```bash
# Создать новый SSH ключ (если его нет)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Или если ed25519 не поддерживается, используйте RSA
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Когда спросит где сохранить, просто нажмите Enter (сохранится в ~/.ssh/id_ed25519 или ~/.ssh/id_rsa)
# Когда спросит пароль, можно оставить пустым или задать (рекомендуется задать)
```

Теперь скопируйте публичный ключ:

```bash
# Для ed25519
cat ~/.ssh/id_ed25519.pub

# Или для RSA
cat ~/.ssh/id_rsa.pub
```

Скопируйте весь вывод (начинается с `ssh-ed25519` или `ssh-rsa`) - он понадобится дальше.

### Шаг 3: Подготовка DNS записей

Перед началом развертывания нужно добавить DNS записи. Предположим, что:
- IP панели: `192.0.2.100`
- IP ноды usa: `192.0.2.1`

Добавьте в вашем DNS провайдере (Cloudflare, Namecheap и т.д.):

```
panel-usa.example.com    A    192.0.2.100
monitoring-usa.example.com    A    192.0.2.100
cdn-usa.example.com     A    192.0.2.1
```

Замените `example.com` на ваш домен. Подождите пока DNS записи распространятся (обычно 5-15 минут, иногда до часа). Проверить можно командой:

```bash
dig panel-usa.example.com +short
# Должен вернуть: 192.0.2.100
```

### Шаг 4: Настройка inventory.ini

Откройте файл `inventory.ini` и настройте его:

```ini
[nodes]
usa ansible_host=192.0.2.1 ansible_user=root ansible_ssh_pass=ВАШ_ПАРОЛЬ_ОТ_СЕРВЕРА

[panel]
panel-usa ansible_host=192.0.2.100 ansible_user=root ansible_ssh_pass=ВАШ_ПАРОЛЬ_ОТ_СЕРВЕРА

[nodes:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Замените:
- `192.0.2.1` на реальный IP вашей ноды usa
- `192.0.2.100` на реальный IP вашей панели
- `ВАШ_ПАРОЛЬ_ОТ_СЕРВЕРА` на root пароль от серверов

**Важно:** После первого запуска плейбука парольная авторизация будет отключена, и нужно будет убрать `ansible_ssh_pass` из inventory.ini.

### Шаг 5: Настройка group_vars

#### group_vars/nodes.yaml

Откройте `group_vars/nodes.yaml` и заполните:

```yaml
---
system_user: "admin"
system_user_password: "ВАШ_СЛУЧАЙНЫЙ_ПАРОЛЬ"
app_port: 2222
monitoring_ip: "192.0.2.100"
consul_server_ip: "192.0.2.100"
panel_hostname: "panel-usa"
ssh_public_key: "ВАШ_SSH_ПУБЛИЧНЫЙ_КЛЮЧ"
prometheus_remnawave_username: "admin"
prometheus_remnawave_password: "SHA512_HASH_ПАРОЛЯ"
```

Как заполнить:

1. **system_user_password** - сгенерируйте случайный пароль:
   ```bash
   openssl rand -base64 24
   ```
   Скопируйте результат и вставьте вместо `ВАШ_СЛУЧАЙНЫЙ_ПАРОЛЬ`.

2. **monitoring_ip** и **consul_server_ip** - это IP панели (192.0.2.100 в нашем примере).

3. **panel_hostname** - имя панели из inventory.ini, у нас это `panel-usa`.

4. **ssh_public_key** - тот ключ, который вы скопировали на шаге 2. Вставьте его целиком, например:
   ```
   ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGxK8vQ... your_email@example.com"
   ```

5. **prometheus_remnawave_password** - это SHA-512 hash пароля. Сначала сгенерируйте пароль:
   ```bash
   openssl rand -base64 24
   ```
   Потом создайте hash:
   ```bash
   echo -n 'ваш_пароль_здесь' | sha512sum | cut -d' ' -f1
   ```
   Скопируйте hash и вставьте вместо `SHA512_HASH_ПАРОЛЯ`.

#### group_vars/panel.yaml

Откройте `group_vars/panel.yaml` и заполните аналогично:

```yaml
---
system_user: "admin"
system_user_password: "ТОТ_ЖЕ_ПАРОЛЬ_ЧТО_В_NODES"
ssh_public_key: "ТОТ_ЖЕ_КЛЮЧ_ЧТО_В_NODES"
prometheus_remnawave_username: "admin"
prometheus_remnawave_password: "ТОТ_ЖЕ_HASH_ЧТО_В_NODES"
```

Можно использовать те же значения, что и в `nodes.yaml`.

### Шаг 6: Настройка host_vars для панели

Создайте файл `host_vars/panel-usa.yml`:

```bash
nano host_vars/panel-usa.yml
```

Вставьте следующее:

```yaml
---
remnawave_panel_domain: "panel-usa.example.com"
monitoring_domain: "monitoring-usa.example.com"
grafana_port: 3302

# JWT секреты - сгенерируйте два разных ключа
jwt_auth_secret: "ВАШ_ПЕРВЫЙ_JWT_СЕКРЕТ"
jwt_api_tokens_secret: "ВАШ_ВТОРОЙ_JWT_СЕКРЕТ"

# PostgreSQL настройки
postgres_user: "remnawave"
postgres_password: "ВАШ_ПАРОЛЬ_POSTGRES"
postgres_db: "remnawave"

# Caddy Authentication настройки
caddy_auth_admin_user: "admin"
caddy_auth_admin_email: "admin@example.com"
caddy_auth_admin_secret: "ВАШ_ПАРОЛЬ_ДЛЯ_CADDY_AUTH"
caddy_auth_token_lifetime: 2592000
```

Как заполнить:

1. **remnawave_panel_domain** и **monitoring_domain** - ваши домены из шага 3.

2. **jwt_auth_secret** и **jwt_api_tokens_secret** - сгенерируйте два разных ключа:
   ```bash
   openssl rand -hex 64
   ```
   Выполните команду дважды, получите два разных ключа и вставьте их.

3. **postgres_password** - сгенерируйте пароль:
   ```bash
   openssl rand -base64 24
   ```

4. **caddy_auth_admin_secret** - это пароль для входа в Caddy Authentication портал. Сгенерируйте:
   ```bash
   openssl rand -base64 24
   ```

Сохраните файл (Ctrl+O, Enter, Ctrl+X в nano).

### Шаг 7: Развертывание панели

Теперь можно развернуть панель:

```bash
ansible-playbook -i inventory.ini panel.yml --limit panel-usa
```

Это займет 10-15 минут. Плейбук установит все необходимое: Docker, PostgreSQL, Redis, RemnaWave Backend, Caddy и настроит безопасность.

Если все прошло успешно, вы увидите в конце сообщение с URL панели и учетными данными для Caddy Authentication.

### Шаг 8: Первый вход в панель

1. Откройте в браузере: `https://panel-usa.example.com`

2. Вас перенаправит на страницу Caddy Authentication. Введите:
   - Пользователь: `admin` (или тот, что указали в `caddy_auth_admin_user`)
   - Email: `admin@example.com` (или тот, что указали в `caddy_auth_admin_email`)
   - Пароль: тот, что сгенерировали для `caddy_auth_admin_secret`

3. После входа вы попадете в RemnaWave панель. Зарегистрируйте первого пользователя - он станет администратором.

### Шаг 9: Добавление ноды в панели

1. В панели RemnaWave перейдите в раздел "Nodes" (Ноды).

2. Нажмите "Add Node" (Добавить ноду).

3. Заполните:
   - **Name**: `usa` (или любое другое имя)
   - **IP**: `192.0.2.1` (IP вашей ноды)
   - **Port**: `2222` (или тот, что указали в `app_port`)

4. Сохраните. Панель покажет **SECRET_KEY** - скопируйте его, он понадобится дальше.

### Шаг 10: Настройка host_vars для ноды

Создайте файл `host_vars/usa.yml`:

```bash
nano host_vars/usa.yml
```

Вставьте:

```yaml
---
SECRET_KEY: "СКОПИРОВАННЫЙ_SECRET_KEY_ИЗ_ПАНЕЛИ"
```

Сохраните файл.

### Шаг 11: Создание Caddyfile для ноды

Создайте файл `templates/caddyfiles/usa.j2`:

```bash
nano templates/caddyfiles/usa.j2
```

Вставьте:

```caddyfile
https://cdn-usa.example.com:843 {
    root * /srv/games-site
    file_server
}
```

Замените `cdn-usa.example.com` на ваш реальный домен (который вы добавили в DNS на шаге 3).

Сохраните файл.


### Шаг 12: Развертывание ноды

Теперь разверните ноду:

```bash
ansible-playbook -i inventory.ini nodes.yml --limit usa
```

Это также займет 10-15 минут. Плейбук установит RemnaNode, Caddy, Consul Agent, Node Exporter и все необходимое.

### Шаг 13: Развертывание мониторинга

Разверните мониторинг на панели:

```bash
ansible-playbook -i inventory.ini monitoring.yml --limit panel-usa
```

Это установит Prometheus и Grafana.

### Шаг 14: Обновление inventory.ini

Теперь, когда SSH ключи настроены, уберите пароли из `inventory.ini`:

```ini
[nodes]
usa ansible_host=192.0.2.1 ansible_user=root

[panel]
panel-usa ansible_host=192.0.2.100 ansible_user=root

[nodes:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Уберите `ansible_ssh_pass=...` из обеих строк.

### Шаг 15: Проверка работы

1. **Панель**: Откройте `https://panel-usa.example.com` - должна открываться панель RemnaWave.

3. **Мониторинг**: Откройте `https://monitoring-usa.example.com` - должна открываться Grafana.

4. **Проверка ноды в панели**: В панели RemnaWave в разделе "Nodes" нода `usa` должна быть онлайн (зеленый статус).

### Готово!

Теперь у вас работает:
- ✅ Панель управления RemnaWave
- ✅ Нода usa с RemnaNode
- ✅ Мониторинг (Prometheus + Grafana)

### Что дальше?

- Добавьте больше нод - просто повторите шаги 9-13 для каждой новой ноды
- Настройте дашборды Linux node _ fleet overview-1763743007941 и Node Exporter Full-1763743021570 в Grafana - импортируйте дашборды из `templates/`

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

- **RemnaWave** - основной сервис панели управления. Предоставляет REST API для управления пользователями, нодами, подписками. 

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

- **RemnaNode** - Xray прокси сервер, который обрабатывает весь трафик пользователей.

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

## Переменные

### group_vars/nodes.yml (общие для всей группы nodes)

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


## Автоматические задачи

После развертывания автоматически выполняются:

| Задача | Расписание | Описание |
|--------|-----------|----------|
| Обновление GEO файлов | Воскресенье 3:00 МСК | Обновление geosite, geoip, zapret.dat |
| UFW Blacklist | Ежедневно 3:00 | Блокировка сетей сканеров |
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


## Логи

Все логи автоматически ротируются.

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
