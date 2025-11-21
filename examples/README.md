# Примеры конфигураций

Эта директория содержит примеры конфигурационных файлов для быстрого старта.

## 📁 Файлы

### Inventory
- `inventory-example.ini` - Пример файла инвентаря с серверами

### Variables
- `group_vars-example.yml` - Общие переменные для всех нод
- `host_vars-node-example.yml` - Переменные для ноды
- `host_vars-panel-example.yml` - Переменные для панели

### Caddyfile примеры
- `caddyfile-simple.j2` - Простой Caddyfile для одного домена
- `caddyfile-custom-port.j2` - Caddyfile с кастомным портом
- `caddyfile-multiple-domains.j2` - Несколько доменов на одной ноде

## 🚀 Как использовать

### 1. Скопировать примеры

```bash
# Inventory
cp examples/inventory-example.ini inventory.ini

# Group vars
cp examples/group_vars-example.yml group_vars/nodes.yml

# Host vars для ноды
cp examples/host_vars-node-example.yml host_vars/node1.yml

# Host vars для панели
cp examples/host_vars-panel-example.yml host_vars/panel.yml

# Caddyfile
cp examples/caddyfile-simple.j2 templates/caddyfiles/node1.j2
```

### 2. Отредактировать значения

Замените все `YOUR_*` и `example.com` на свои реальные значения.

### 3. Развернуть

```bash
ansible-playbook -i inventory.ini nodes.yml
```

## 💡 Советы

- Начните с простого Caddyfile, потом добавляйте сложность
- Используйте разные Cloudflare токены для dev/prod
- Храните секреты в Ansible Vault
- Тестируйте на одной ноде перед массовым развертыванием

