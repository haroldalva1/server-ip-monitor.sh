#!/bin/bash

# ================= CONFIGURACIÓN =================
VERSION="2.3"
SERVER="uploads.mitv.plus"
PORT="2022"
USER="ips_user"
PASS="ips_user"
REMOTE_FILE="/uploads/temporal.txt"
LOCK_FILE="/tmp/instalador.lock"
LOCK_TIMEOUT=300  # 5 minutos en segundos
LOG_DIR="/var/log/instalador"
LOG_FILE="$LOG_DIR/instalador.log"
MAX_RETRIES=5
RETRY_DELAY=10
MAX_LOG_SIZE=10485760  # 10MB
HEARTBEAT_FILE="/tmp/instalador_heartbeat"
CLIENT_NAME_FILE="/root/nombre_cliente"

# ============== FUNCIONES ==============

initialize_environment() {
    sudo mkdir -p "$LOG_DIR"
    sudo chmod 755 "$LOG_DIR"
    sudo touch "$LOG_FILE"
    sudo chmod 644 "$LOG_FILE"
    
    sudo bash -c "cat > /etc/logrotate.d/instalador <<EOF
$LOG_FILE {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
    maxsize $MAX_LOG_SIZE
}
EOF"
    
    if ! systemctl list-units --all --plain | grep -q "instalador.timer"; then
        sudo bash -c "cat > /etc/systemd/system/instalador.service <<EOF
[Unit]
Description=Instalador Client Service
After=network.target

[Service]
Type=oneshot
ExecStart=/root/instalador.sh
User=root
EOF"

        sudo bash -c "cat > /etc/systemd/system/instalador.timer <<EOF
[Unit]
Description=Run Instalador every minute

[Timer]
OnCalendar=*:0/1
Persistent=true
Unit=instalador.service

[Install]
WantedBy=timers.target
EOF"

        systemctl daemon-reload
        systemctl enable --now instalador.timer
    fi
}

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [v$VERSION] $1" | sudo tee -a "$LOG_FILE"
}

safe_lock() {
    # Lock file con verificación de timeout
    if [ -f "$LOCK_FILE" ]; then
        local lock_age=$(($(date +%s) - $(stat -c %Y "$LOCK_FILE")))
        if [ "$lock_age" -lt "$LOCK_TIMEOUT" ]; then
            log "Script ya en ejecución (lock activo por $lock_age segundos). Saliendo."
            exit 0
        else
            log "Eliminando lock file antiguo ($lock_age segundos)"
            rm -f "$LOCK_FILE"
        fi
    fi
    touch "$LOCK_FILE"
}

validate_client_name() {
    if [ ! -f "$CLIENT_NAME_FILE" ] || [ -z "$(sudo cat "$CLIENT_NAME_FILE" 2>/dev/null)" ]; then
        while true; do
            read -p "INGRESE NOMBRE DEL SERVIDOR (obligatorio): " server_name
            if [ -n "$server_name" ]; then
                echo "$server_name" | sudo tee "$CLIENT_NAME_FILE" >/dev/null
                sudo chmod 600 "$CLIENT_NAME_FILE"
                log "Nombre registrado: $server_name"
                break
            fi
            echo "ERROR: El nombre no puede estar vacío" >&2
        done
    fi
}

# [...] (mantener las demás funciones igual)

# ============ EJECUCIÓN PRINCIPAL ============

# Configuración inicial
[ ! -d "$LOG_DIR" ] && initialize_environment

# Gestión de locks mejorada
safe_lock
trap 'rm -f "$LOCK_FILE"; log "Lock liberado"' EXIT

log "=== INICIO DE EJECUCIÓN ==="

# Validación obligatoria
validate_client_name

# [...] (resto del script igual)

log "=== EJECUCIÓN COMPLETADA ==="
exit 0
