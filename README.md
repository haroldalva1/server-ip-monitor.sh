#!/bin/bash
# INSTALADOR COMPLETO DEL MONITOR DE IP - VERSIÓN FINAL 4.0
# Incluye: 
# - Solicitud obligatoria de nombre
# - Corrección de permisos
# - Validación robusta
# - Configuración automática

# Verificar ejecución como root
if [ "$(id -u)" -ne 0 ]; then
    echo -e "\033[1;31mERROR: Este script debe ejecutarse como root\033[0m" >&2
    exit 1
fi

# ==============================================
# 1. CREAR EL SCRIPT PRINCIPAL CON PERMISOS
# ==============================================
cat > /usr/local/bin/server-ip-monitor.sh << 'EOSCRIPT'
#!/bin/bash
# Server IP Monitor v4.0 - Versión Final

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
    echo -e "\n\033[1;36mCONFIGURACIÓN INICIAL DEL SERVIDOR\033[0m"
    echo "============================================"
    echo "  POR FAVOR INGRESE UN NOMBRE PARA ESTE SERVIDOR"
    echo "  Requisitos:"
    echo "  - Mínimo 3 caracteres"
    echo "  - Sin espacios"
    echo "  - Solo letras, números, guiones (-) y guiones bajos (_)"
    echo "  Ejemplos válidos:"
    echo "  - servidor-web-01"
    echo "  - bd_finanzas_primary"
    echo "  - backup-nas"
    echo "============================================"
    
    while true; do
        read -p "  ➤ Nombre del servidor: " SERVER_NAME
        
        # Validaciones
        if [ -z "$SERVER_NAME" ]; then
            echo -e "\033[1;31m  ✖ ERROR: El nombre no puede estar vacío\033[0m"
        elif [ ${#SERVER_NAME} -lt 3 ]; then
            echo -e "\033[1;31m  ✖ ERROR: Mínimo 3 caracteres\033[0m"
        elif [[ "$SERVER_NAME" =~ [[:space:]] ]]; then
            echo -e "\033[1;31m  ✖ ERROR: No se permiten espacios\033[0m"
        elif [[ "$SERVER_NAME" =~ [^a-zA-Z0-9_\-] ]]; then
            echo -e "\033[1;31m  ✖ ERROR: Caracteres no permitidos\033[0m"
        else
            # Guardar configuración
            echo "SERVER_NAME='$SERVER_NAME'" > "$CONFIG_FILE"
            chmod 600 "$CONFIG_FILE"
            echo -e "\033[1;32m  ✔ Nombre guardado correctamente\033[0m"
            return 0
        fi
        
        echo -e "\n  \033[1;33mPOR FAVOR INGRESE UN NOMBRE VÁLIDO\033[0m"
    done
}

# Cargar configuración existente o solicitar nueva
if [ -f "$CONFIG_FILE" ]; then
    source "$CONFIG_FILE"
else
    solicitar_nombre || {
        echo -e "\033[1;31mERROR: Configuración fallida - No se definió nombre\033[0m" >&2
        exit 1
    }
fi

[ -z "$SERVER_NAME" ] && {
    echo -e "\033[1;31mERROR: Nombre no configurado\033[0m" >&2
    exit 1
}

# Función de logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${SERVER_NAME}] $1" >> "$LOG_FILE"
}

# Bloqueo para ejecución única
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
# 2. ASIGNAR PERMISOS CORRECTOS
# ==============================================
chown root:root /usr/local/bin/server-ip-monitor.sh
chmod 755 /usr/local/bin/server-ip-monitor.sh
mkdir -p /etc/
touch /var/log/ip_monitor.log
chmod 644 /var/log/ip_monitor.log

# Instalar dependencias (curl)
if ! command -v curl &> /dev/null; then
    echo -e "\033[1;33mInstalando curl...\033[0m"
    apt-get update > /dev/null && apt-get install -y curl > /dev/null || yum install -y curl > /dev/null
fi

# ==============================================
# 3. CONFIGURACIÓN INICIAL OBLIGATORIA
# ==============================================
echo -e "\n\033[1;36mCONFIGURACIÓN INICIAL REQUERIDA\033[0m"

if [ -f "/etc/ip-monitor.conf" ]; then
    CURRENT_NAME=$(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)
    echo -e "\033[1;32m✓ Configuración existente detectada\033[0m"
    echo -e "  Nombre actual: \033[1;37m$CURRENT_NAME\033[0m"
    
    read -p "¿Desea cambiar el nombre? (s/n): " CHANGE_NAME
    if [[ "$CHANGE_NAME" =~ ^[SsYy] ]]; then
        rm -f /etc/ip-monitor.conf
        echo -e "\n\033[1;33mCONFIGURACIÓN DEL NOMBRE\033[0m"
        /usr/local/bin/server-ip-monitor.sh || exit 1
    fi
else
    echo -e "\033[1;33mDEBE INGRESAR UN NOMBRE PARA EL SERVIDOR:\033[0m"
    /usr/local/bin/server-ip-monitor.sh || {
        echo -e "\033[1;31mERROR: La configuración falló\033[0m"
        exit 1
    }
fi

# ==============================================
# 4. CONFIGURAR CRON JOB
# ==============================================
CRON_JOB="*/5 * * * * /usr/local/bin/server-ip-monitor.sh >> /var/log/ip_monitor.log 2>&1"
if ! crontab -l | grep -q "server-ip-monitor"; then
    (crontab -l 2>/dev/null; echo "$CRON_JOB") | crontab -
    echo -e "\033[1;32m✓ Tarea programada configurada cada 5 minutos\033[0m"
else
    echo -e "\033[1;33m✓ El cron job ya estaba configurado\033[0m"
fi

# ==============================================
# 5. VERIFICACIÓN FINAL
# ==============================================
echo -e "\n\033[1;36mRESUMEN DE INSTALACIÓN\033[0m"
echo -e "\033[1;32m✓ Script instalado en:\033[0m /usr/local/bin/server-ip-monitor.sh"
echo -e "\033[1;32m✓ Configuración guardada en:\033[0m /etc/ip-monitor.conf"
echo -e "\033[1;32m✓ Nombre del servidor:\033[0m \033[1;37m$(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)\033[0m"
echo -e "\033[1;32m✓ Logs del sistema:\033[0m /var/log/ip_monitor.log"
echo -e "\033[1;32m✓ Tarea programada:\033[0m"
crontab -l | grep "server-ip-monitor"

echo -e "\n\033[1;35mINSTALACIÓN COMPLETADA CON ÉXITO!\033[0m"
echo "El sistema comenzará a monitorear automáticamente"
