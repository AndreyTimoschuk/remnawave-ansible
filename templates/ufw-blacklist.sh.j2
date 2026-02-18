#!/bin/bash

# =============================================================================
# UFW + IPSET Blacklist - Многоуровневая блокировка сетей
# =============================================================================
# Два уровня блокировки:
# 1. blacklist_dangerous - опасные сети (ботнеты, малварь, Tor)
#    → блокируем ВСЁ: входящие (src) + исходящие (dst) + ESTABLISHED
# 2. blacklist_ru - российские госструктуры
#    → блокируем только входящие, разрешаем исходящие (API, CDN)
# =============================================================================

# === БЫСТРАЯ ДИАГНОСТИКА ===
# ipset list -t
# echo "Whitelist: $(ipset list whitelist 2>/dev/null | grep -c '^[0-9]')"
# echo "Dangerous: $(ipset list blacklist_dangerous 2>/dev/null | grep -c '^[0-9]')"
# echo "RU: $(ipset list blacklist_ru 2>/dev/null | grep -c '^[0-9]')"
# grep -A15 "IPSET BLACKLIST" /etc/ufw/before.rules
# iptables -L ufw-before-input -v -n --line-numbers | head -15
# iptables -L ufw-before-output -v -n --line-numbers | head -10

# =============================================================================
# КОНФИГУРАЦИЯ
# =============================================================================

LOG_FILE="/var/log/ufw_blacklist.log"
IPSET_SAVE_FILE="/etc/ipset.rules"
IPSET_LOAD_SCRIPT="/usr/local/bin/load-ipset-blacklist.sh"

# Имена ipset наборов
IPSET_WHITELIST="whitelist"
IPSET_DANGEROUS="blacklist_dangerous"  # Опасные - блокируем ВСЁ: INPUT + OUTPUT + ESTABLISHED
IPSET_RU="blacklist_ru"                # Российские - блокируем только INPUT

# =============================================================================
# ИСТОЧНИКИ СПИСКОВ
# =============================================================================

# Российские госструктуры (блокируем входящие, разрешаем исходящие)
URL_RU="https://raw.githubusercontent.com/C24Be/AS_Network_List/main/blacklists/blacklist.txt"

# Опасные сети - блокируем ПОЛНОСТЬЮ (даже ESTABLISHED)
# Источник: https://github.com/firehol/blocklist-ipsets
URLS_DANGEROUS=(
    # === HIJACKED NETWORKS & SPAM ===
    # Spamhaus DROP - hijacked networks, professional spam/cybercrime
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/spamhaus_drop.netset"
    # Spamhaus EDROP - extended DROP list
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/spamhaus_edrop.netset"
    
    # === MALWARE & C&C SERVERS (критично для VPN!) ===
    # Feodo BadIPs - Zeus/Feodo C&C серверы
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/feodo_badips.ipset"
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/blocklist_net_ua.ipset"
    # Zeus троян - C&C серверы
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/iblocklist_abuse_zeus.netset"
    # SpyEye троян - C&C серверы
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/iblocklist_abuse_spyeye.netset"
    # Palevo ботнет (Rimecud, Pilleuz)
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/iblocklist_abuse_palevo.netset"
    # CyberCrime tracker - C&C серверы (botnets, trojans)
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/cybercrime.ipset"
    # CryptoWall ransomware C&C
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/cta_cryptowall.ipset"
    # EmergingThreats compromised hosts (заражённые машины)
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/et_compromised.ipset"
    # Binary Defense - Artillery honeypot intelligence
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/bds_atif.ipset"

    # === BRUTE-FORCE & SCANNERS ===
    # Blocklist.de - brute force attacks (SSH, FTP, etc)
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/blocklist_de.ipset"
    # DShield - top 20 attacking subnets
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/dshield.netset"
    # GreenSnow - port scanners, FTP/SSH brute force
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/greensnow.ipset"

    # === REPUTATION LISTS ===
    # CIArmy - IPs with poor reputation
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/ciarmy.ipset"
    # Darklist.de - SSH fail2ban aggregated reports
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/darklist_de.netset"
    
    # === TOR - блокируем exit nodes ===
    "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/tor_exits.ipset"
)

# =============================================================================
# WHITELIST - IP адреса которые ВСЕГДА должны быть разрешены
# =============================================================================
WHITELIST_IPS=(
{% if iptables_whitelist_ips is defined %}
{% for ip in iptables_whitelist_ips %}
    "{{ ip }}"
{% endfor %}
{% endif %}
    # Критичные IP для работы VPN/прокси
    "95.142.206.1"
)

{% raw %}
# =============================================================================
# ФУНКЦИИ
# =============================================================================

[[ $EUID -ne 0 ]] && { echo "Ошибка: скрипт должен запускаться от root" | tee -a "$LOG_FILE"; exit 1; }

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $*" | tee -a "$LOG_FILE"
}

install_packages() {
    log "Проверка пакетов..."
    if ! command -v ipset &> /dev/null; then
        log "Установка ipset..."
        apt-get update >> "$LOG_FILE" 2>&1
        apt-get install -y ipset >> "$LOG_FILE" 2>&1
    fi
}

create_ipsets() {
    log "Создание ipset наборов..."
    
    # Whitelist
    ipset create "$IPSET_WHITELIST" hash:net family inet hashsize 1024 maxelem 65536 2>/dev/null || true
    
    # Dangerous - опасные сети (большой набор, ~100k записей)
    ipset create "$IPSET_DANGEROUS" hash:net family inet hashsize 131072 maxelem 1000000 2>/dev/null || true
    
    # Russian - российские госструктуры
    ipset create "$IPSET_RU" hash:net family inet hashsize 8192 maxelem 100000 2>/dev/null || true
    
    log "ipset наборы созданы"
}

apply_whitelist() {
    log "Загрузка whitelist..."
    
    ipset flush "$IPSET_WHITELIST" 2>/dev/null || true
    
    local count=0
    for ip in "${WHITELIST_IPS[@]}"; do
        [[ -z "$ip" ]] && continue
        ipset add "$IPSET_WHITELIST" "$ip" 2>/dev/null && ((count++)) || true
    done
    
    log "Whitelist: $count IP"
}

# Загрузка списка в ipset
load_list_to_ipset() {
    local url="$1"
    local ipset_name="$2"
    local temp_file="/tmp/blacklist_$(basename "$url")"
    
    if ! curl -sf --connect-timeout 10 --max-time 60 --retry 2 "$url" -o "$temp_file" 2>/dev/null; then
        log "WARN: Не удалось скачать $url"
        return 1
    fi
    
    # Фильтруем валидные IPv4 подсети
    grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+(/[0-9]+)?$' "$temp_file" 2>/dev/null | while read -r subnet; do
        echo "add $ipset_name $subnet"
    done
    
    rm -f "$temp_file"
}

apply_dangerous_blacklist() {
    log "Загрузка ОПАСНЫХ списков (блокируем ВСЁ включая ESTABLISHED)..."
    
    if [[ ${#URLS_DANGEROUS[@]} -eq 0 ]]; then
        log "WARN: URLS_DANGEROUS пуст - dangerous блокировка отключена"
        return 0
    fi
    
    ipset flush "$IPSET_DANGEROUS" 2>/dev/null || true
    
    local restore_file="/tmp/ipset_dangerous_restore.txt"
    > "$restore_file"
    
    for url in "${URLS_DANGEROUS[@]}"; do
        log "  Скачивание: $(basename "$url")"
        load_list_to_ipset "$url" "$IPSET_DANGEROUS" >> "$restore_file"
    done
    
    # Удаляем дубликаты и загружаем
    sort -u "$restore_file" > "${restore_file}.sorted"
    local count=$(wc -l < "${restore_file}.sorted")
    
    if ipset restore -! < "${restore_file}.sorted" 2>> "$LOG_FILE"; then
        log "Dangerous blacklist: $count подсетей"
    else
        log "WARN: Ошибки при загрузке dangerous blacklist"
    fi
    
    rm -f "$restore_file" "${restore_file}.sorted"
}

apply_ru_blacklist() {
    log "Загрузка RU списка (блокируем входящие, разрешаем исходящие)..."
    
    if [[ -z "$URL_RU" ]]; then
        log "WARN: URL_RU пуст - RU блокировка отключена"
        return 0
    fi
    
    ipset flush "$IPSET_RU" 2>/dev/null || true
    
    local temp_file="/tmp/blacklist_ru.txt"
    
    if ! curl -sf --connect-timeout 10 --max-time 60 --retry 3 "$URL_RU" -o "$temp_file" 2>/dev/null; then
        log "WARN: Не удалось скачать RU список"
        return 1
    fi
    
    local restore_file="/tmp/ipset_ru_restore.txt"
    grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+(/[0-9]+)?$' "$temp_file" | \
        sort -u | \
        while read -r subnet; do echo "add $IPSET_RU $subnet"; done > "$restore_file"
    
    local count=$(wc -l < "$restore_file")
    
    if ipset restore -! < "$restore_file" 2>> "$LOG_FILE"; then
        log "RU blacklist: $count подсетей"
    else
        log "WARN: Ошибки при загрузке RU blacklist"
    fi
    
    rm -f "$temp_file" "$restore_file"
}

integrate_with_ufw() {
    log "Интеграция с UFW..."
    
    local before_rules="/etc/ufw/before.rules"
    local marker_start="# BEGIN IPSET BLACKLIST"
    local marker_end="# END IPSET BLACKLIST"
    local tmp_rules="/tmp/ipset_ufw_rules.txt"
    
    [[ ! -f "$before_rules" ]] && { log "ERROR: $before_rules не найден"; return 1; }
    
    # Удаляем старые правила
    if grep -q "$marker_start" "$before_rules"; then
        sed -i "/$marker_start/,/$marker_end/d" "$before_rules"
    fi
    
    # ВАЖНО: Порядок правил критичен!
    # 1. Whitelist - всегда пропускаем
    # 2. Dangerous - блокируем ВСЁ (ДО проверки ESTABLISHED!)
    # 3. ESTABLISHED,RELATED - пропускаем ответы на наши соединения
    # 4. RU blacklist - блокируем только новые входящие
    cat > "$tmp_rules" << EOF

$marker_start
# === WHITELIST - всегда разрешаем доверенные IP ===
-A ufw-before-input -m set --match-set $IPSET_WHITELIST src -j ACCEPT
-A ufw-before-output -m set --match-set $IPSET_WHITELIST dst -j ACCEPT

# === DANGEROUS - блокируем ВСЁ (ботнеты, малварь, Tor, спамеры) ===
# Входящие от опасных IP
-A ufw-before-input -m set --match-set $IPSET_DANGEROUS src -j DROP
# Исходящие К опасным IP (не даём серверу подключаться к ботнетам/малвари)
-A ufw-before-output -m set --match-set $IPSET_DANGEROUS dst -j DROP

# === ESTABLISHED/RELATED - разрешаем ответы на НАШИ соединения ===
# (только для IP которые НЕ в dangerous списке - они уже заблокированы выше)
-A ufw-before-input -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# === RU BLACKLIST - блокируем только НОВЫЕ входящие соединения ===
# Можно делать исходящие соединения к этим IP (API, CDN, etc)
-A ufw-before-input -m set --match-set $IPSET_RU src -j DROP
$marker_end
EOF
    
    # Вставляем правила
    if grep -q "# End required lines" "$before_rules"; then
        sed -i "/# End required lines/r $tmp_rules" "$before_rules"
    else
        sed -i "/:ufw-before-input/r $tmp_rules" "$before_rules"
    fi
    
    rm -f "$tmp_rules"
    
    if grep -q "$marker_start" "$before_rules"; then
        log "UFW интеграция успешна"
    else
        log "ERROR: UFW интеграция не удалась!"
        return 1
    fi
}

save_ipset() {
    log "Сохранение ipset..."
    
    ipset save > "$IPSET_SAVE_FILE" 2>/dev/null
    
    cat > "$IPSET_LOAD_SCRIPT" << 'LOADSCRIPT'
#!/bin/bash
IPSET_SAVE_FILE="/etc/ipset.rules"
[[ -f "$IPSET_SAVE_FILE" ]] && ipset restore < "$IPSET_SAVE_FILE"
LOADSCRIPT
    chmod +x "$IPSET_LOAD_SCRIPT"
    
    cat > /etc/systemd/system/ipset-load.service << 'SERVICEEOF'
[Unit]
Description=Load ipset rules
Before=ufw.service
After=network-pre.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/load-ipset-blacklist.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SERVICEEOF
    
    systemctl daemon-reload
    systemctl enable ipset-load.service >> "$LOG_FILE" 2>&1
    
    log "Автозагрузка настроена"
}

reload_ufw() {
    log "Перезагрузка UFW..."
    ufw reload >> "$LOG_FILE" 2>&1
}

show_stats() {
    echo ""
    echo "==========================================" | tee -a "$LOG_FILE"
    echo "              СТАТИСТИКА                  " | tee -a "$LOG_FILE"
    echo "==========================================" | tee -a "$LOG_FILE"
    echo "Whitelist:                 $(ipset list "$IPSET_WHITELIST" 2>/dev/null | grep -c '^[0-9]' || echo 0) IP" | tee -a "$LOG_FILE"
    echo "Dangerous (IN+OUT блок):   $(ipset list "$IPSET_DANGEROUS" 2>/dev/null | grep -c '^[0-9]' || echo 0) подсетей" | tee -a "$LOG_FILE"
    echo "RU (только INPUT блок):    $(ipset list "$IPSET_RU" 2>/dev/null | grep -c '^[0-9]' || echo 0) подсетей" | tee -a "$LOG_FILE"
    echo "==========================================" | tee -a "$LOG_FILE"
    echo ""
    echo "Whitelist IP:" | tee -a "$LOG_FILE"
    ipset list "$IPSET_WHITELIST" 2>/dev/null | grep '^[0-9]' | head -20 | tee -a "$LOG_FILE"
    echo ""
    echo "UFW правила (INPUT):" | tee -a "$LOG_FILE"
    iptables -L ufw-before-input -n --line-numbers 2>/dev/null | head -8 | tee -a "$LOG_FILE"
    echo ""
    echo "UFW правила (OUTPUT):" | tee -a "$LOG_FILE"
    iptables -L ufw-before-output -n --line-numbers 2>/dev/null | head -5 | tee -a "$LOG_FILE"
}

# =============================================================================
# MAIN
# =============================================================================

log "========================================"
log "UFW + IPSET Multi-Blacklist запущен"
log "========================================"

install_packages
create_ipsets
apply_whitelist
apply_dangerous_blacklist    # Сначала опасные (блок ВСЁ)
apply_ru_blacklist           # Потом RU (блок только входящие)
integrate_with_ufw
save_ipset
reload_ufw
show_stats

log "Скрипт завершён успешно"
{% endraw %}
