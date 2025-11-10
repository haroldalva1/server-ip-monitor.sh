#!/bin/bash
# INSTALADOR COMPLETO DEL MONITOR DE IP (CON NOMBRE PERSONALIZADO)
# Versión 3.0 - Incluye configuración interactiva inicial

# Verificar root
if [ "$(id -u)" -ne 0 ]; then
    echo -e "\033[1;31mERROR: Debes ejecutar este script como root\033[0m"
    exit 1
fi

# ==============================================
# 1. CREAR EL SCRIPT PRINCIPAL
# ==============================================
cat > /usr/local/bin/server-ip-monitor.sh << 'EOSCRIPT'
#!/bin/bash
# Server IP Monitor v3.0 - Con nombre personalizado

# Configuración
CONFIG_FILE="/etc/ip-monitor.conf"
DEST_SERVER="root@38.52.182.225"
DEST_PORT="2022"
DEST_PATH="/root/ips/ip_list.txt"
LOCK_FILE="/tmp/ip_monitor.lock"
LOG_FILE="/var/log/ip_monitor.log"

# Función para solicitar nombre interactivo
solicitar_nombre() {
    clear
    echo -e "\n\033[1;36mCONFIGURACIÓN INICIAL - NOMBRE DEL SERVIDOR\033[0m"
    echo "============================================"
    echo "  Por favor ingrese un nombre descriptivo para"
    echo "  este servidor. Ejemplos:"
    echo "  - ServidorWeb-Producción"
    echo "  - BD-Finanzas"
    echo "  - BackupNAS"
    echo "============================================"
    while true; do
        read -p "  ➤ Nombre del servidor: " SERVER_NAME
        if [ -z "$SERVER_NAME" ]; then
            echo -e "\033[1;31m  ✖ El nombre no puede estar vacío\033[0m"
        else
            echo "SERVER_NAME='$SERVER_NAME'" > "$CONFIG_FILE"
            chmod 600 "$CONFIG_FILE"
            echo -e "\033[1;32m  ✔ Nombre guardado en $CONFIG_FILE\033[0m"
            break
        fi
    done
}

# Cargar configuración
load_config() {
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
    else
        solicitar_nombre
    fi
}

# Función de logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${SERVER_NAME}] $1" >> "$LOG_FILE"
}

# Main
load_config
exec 200>"$LOCK_FILE"
flock -n 200 || { log "Ejecución duplicada detectada"; exit 1; }

# Obtener IPs
PRIVATE_IP=$(hostname -I | awk '{print $1}')
PUBLIC_IP=$(curl -4 -s --max-time 5 https://ifconfig.me/ip || echo "N/A")

# Validar IPs
validate_ip() {
    [[ $1 =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]
}

validate_ip "$PRIVATE_IP" || PRIVATE_IP="INVALID_IP"
validate_ip "$PUBLIC_IP" || PUBLIC_IP="INVALID_IP"

# Generar reporte
DATA_TEMP=$(mktemp)
cat > "$DATA_TEMP" <<EOF
===== $SERVER_NAME =====
Hostname Real: $(hostname)
Timestamp: $(date '+%Y-%m-%d %H:%M:%S %Z')
Private IP: $PRIVATE_IP
Public IP: $PUBLIC_IP
Uptime: $(uptime -p)
Load Average: $(awk '{print $1,$2,$3}' /proc/loadavg)
----------------------------------------
EOF

# Enviar datos
send_data() {
    if scp -P "$DEST_PORT" -o "StrictHostKeyChecking=no" "$DATA_TEMP" "${DEST_SERVER}:${DEST_PATH}-${SERVER_NAME// /_}" >/dev/null 2>&1; then
        ssh -p "$DEST_PORT" -o "StrictHostKeyChecking=no" "$DEST_SERVER" "cat ${DEST_PATH}-* >> ${DEST_PATH} && rm -f ${DEST_PATH}-*" >/dev/null 2>&1
        return $?
    fi
    return 1
}

if send_data; then
    log "Datos enviados correctamente"
else
    log "Error al enviar datos"
fi

rm -f "$DATA_TEMP"
flock -u 200
exit 0
EOSCRIPT

# ==============================================
# 2. CONFIGURAR PERMISOS Y DEPENDENCIAS
# ==============================================
chmod 700 /usr/local/bin/server-ip-monitor.sh
chown root:root /usr/local/bin/server-ip-monitor.sh
mkdir -p /etc/
touch /var/log/ip_monitor.log
chmod 644 /var/log/ip_monitor.log

# Instalar dependencias (curl)
if ! command -v curl &> /dev/null; then
    echo -e "\033[1;33mInstalando curl...\033[0m"
    apt-get update > /dev/null && apt-get install -y curl > /dev/null || yum install -y curl > /dev/null
fi

# ==============================================
# 3. EJECUTAR CONFIGURACIÓN INICIAL
# ==============================================
echo -e "\n\033[1;36mINICIANDO CONFIGURACIÓN\033[0m"
if [ ! -f "/etc/ip-monitor.conf" ]; then
    /usr/local/bin/server-ip-monitor.sh
else
    echo -e "\033[1;32m✓ Configuración ya existente (/etc/ip-monitor.conf)\033[0m"
    echo -e "  Nombre actual: $(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)"
fi

# ==============================================
# 4. PROGRAMAR EN CRON
# ==============================================
CRON_JOB="*/5 * * * * /usr/local/bin/server-ip-monitor.sh >> /var/log/ip_monitor.log 2>&1"
if ! crontab -l | grep -q "server-ip-monitor"; then
    (crontab -l 2>/dev/null; echo "$CRON_JOB") | crontab -
    echo -e "\033[1;32m✓ Cron job configurado cada 5 minutos\033[0m"
else
    echo -e "\033[1;33m✓ El cron job ya estaba configurado\033[0m"
fi

# ==============================================
# 5. VERIFICACIÓN FINAL
# ==============================================
echo -e "\n\033[1;36mRESUMEN DE INSTALACIÓN\033[0m"
echo -e "\033[1;32m✓ Script instalado en:\033[0m /usr/local/bin/server-ip-monitor.sh"
echo -e "\033[1;32m✓ Configuración en:\033[0m /etc/ip-monitor.conf"
echo -e "\033[1;32m✓ Logs en:\033[0m /var/log/ip_monitor.log"
echo -e "\033[1;32m✓ Cron job:\033[0m"
crontab -l | grep "server-ip-monitor"
echo -e "\033[1;32m✓ Nombre asignado:\033[0m $(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)"

echo -e "\n\033[1;35mINSTALACIÓN COMPLETADA!\033[0m"
echo "El monitor comenzará a funcionar automáticamente en 5 minutos"
