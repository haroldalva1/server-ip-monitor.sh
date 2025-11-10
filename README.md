#!/bin/bash
# Server IP Monitor v1.3
# Permite asignar nombres personalizados a los servidores

# Configuración
CONFIG_FILE="/etc/ip-monitor.conf"
DEST_SERVER="root@38.52.182.225"
DEST_PORT="2022"
DEST_PATH="/root/ips/ip_list.txt"
LOCK_FILE="/tmp/ip_monitor.lock"
LOG_FILE="/var/log/ip_monitor.log"

# Cargar configuración o inicializar
load_config() {
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
    else
        # Solicitar nombre si es primera ejecución
        if [ ! -f "$CONFIG_FILE" ] && [ -t 0 ]; then
            clear
            echo "============================================"
            echo "  CONFIGURACIÓN INICIAL DEL SERVIDOR"
            echo "============================================"
            echo -n "Ingrese un nombre descriptivo para este servidor: "
            read -r SERVER_NAME
            echo "SERVER_NAME=\"$SERVER_NAME\"" > "$CONFIG_FILE"
            chmod 600 "$CONFIG_FILE"
            echo "Configuración guardada en $CONFIG_FILE"
        else
            SERVER_NAME=$(hostname -f)
        fi
    fi
}

# Función de logging mejorada
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$SERVER_NAME] $1" >> "$LOG_FILE"
}

# Crear directorios necesarios
mkdir -p "$(dirname "$LOCK_FILE")" "$(dirname "$CONFIG_FILE")"
[ -f "$LOG_FILE" ] || touch "$LOG_FILE"

# Cargar configuración
load_config

# Bloqueo para ejecución única
exec 200>"$LOCK_FILE"
flock -n 200 || { log "Ejecución duplicada detectada. Saliendo."; exit 1; }

# Obtener información de red
PRIVATE_IP=$(ip route get 1 | awk '{print $7}' | head -1)
PUBLIC_IP=$(curl -s --max-time 5 --retry 2 https://ifconfig.me/ip || echo "N/A")

# Validar IPs
validate_ip() {
    local ip=$1
    [[ $ip =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]] || return 1
    return 0
}

validate_ip "$PRIVATE_IP" || PRIVATE_IP="INVALID_IP"
validate_ip "$PUBLIC_IP" || PUBLIC_IP="INVALID_IP"

# Preparar datos
DATA_TEMP=$(mktemp)
cat <<EOF > "$DATA_TEMP"
===== $SERVER_NAME =====
Hostname Real: $(hostname -f)
Timestamp: $(date -Is)
Private IP: $PRIVATE_IP
Public IP: $PUBLIC_IP
System Uptime: $(uptime -p)
Load Average: $(awk '{print $1,$2,$3}' /proc/loadavg)
----------------------------------------
EOF

# Enviar datos
if scp -P "$DEST_PORT" -o "StrictHostKeyChecking=no" "$DATA_TEMP" "$DEST_SERVER:$DEST_PATH-$SERVER_NAME"; then
    if ssh -p "$DEST_PORT" -o "StrictHostKeyChecking=no" "$DEST_SERVER" "cat $DEST_PATH-* >> $DEST_PATH && rm -f $DEST_PATH-*"; then
        log "Datos enviados exitosamente al servidor central"
    else
        log "Error al consolidar datos en el servidor destino"
    fi
else
    log "Error al enviar datos al servidor destino"
fi

# Limpieza
rm -f "$DATA_TEMP"
flock -u 200
exit 0
