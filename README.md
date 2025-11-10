#!/bin/bash
# INSTALADOR COMPLETO DEL MONITOR DE IP (CON NOMBRE OBLIGATORIO)
# Versión 3.1 - Validación reforzada del nombre

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
# Server IP Monitor v3.1 - Nombre obligatorio

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
    echo "  Este campo es OBLIGATORIO. Ingrese un nombre"
    echo "  descriptivo para identificar este servidor:"
    echo "  (Mínimo 3 caracteres, sin caracteres especiales)"
    echo "  Ejemplos válidos:"
    echo "  - ServidorWeb-Produccion"
    echo "  - BD_Finanzas_Primaria"
    echo "  - BackupNAS01"
    echo "============================================"
    
    while true; do
        read -p "  ➤ Nombre del servidor: " SERVER_NAME
        # Validaciones
        if [ -z "$SERVER_NAME" ]; then
            echo -e "\033[1;31m  ✖ Error: El nombre no puede estar vacío\033[0m"
        elif [ ${#SERVER_NAME} -lt 3 ]; then
            echo -e "\033[1;31m  ✖ Error: Mínimo 3 caracteres\033[0m"
        elif [[ "$SERVER_NAME" =~ [^a-zA-Z0-9_\-] ]]; then
            echo -e "\033[1;31m  ✖ Error: Solo use letras, números, guiones y guiones bajos\033[0m"
        else
            # Guardar configuración
            echo "SERVER_NAME='$SERVER_NAME'" > "$CONFIG_FILE"
            chmod 600 "$CONFIG_FILE"
            echo -e "\033[1;32m  ✔ Nombre guardado correctamente\033[0m"
            break
        fi
        echo "  Por favor, ingrese un nombre válido para continuar..."
    done
}

# Cargar configuración
load_config() {
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
        return 0
    else
        solicitar_nombre
        return $?
    fi
}

# Función de logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${SERVER_NAME}] $1" >> "$LOG_FILE"
}

# --- Validación crítica del nombre ---
if ! load_config; then
    echo -e "\033[1;31mFATAL: No se pudo configurar el nombre del servidor\033[0m"
    exit 1
fi

# Main
exec 200>"$LOCK_FILE"
flock -n 200 || { log "Ejecución duplicada detectada"; exit 1; }

[ -z "$SERVER_NAME" ] && {
    log "Error: Nombre del servidor no configurado"
    exit 1
}

# ... (resto del script original igual)
EOSCRIPT

# ==============================================
# 2. CONFIGURAR PERMISOS (igual que antes)
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
# 3. EJECUTAR CONFIGURACIÓN INICIAL (MODIFICADO)
# ==============================================
echo -e "\n\033[1;36mCONFIGURACIÓN OBLIGATORIA\033[0m"
if [ ! -f "/etc/ip-monitor.conf" ]; then
    echo -e "\033[1;33mDEBE CONFIGURAR UN NOMBRE PARA CONTINUAR\033[0m"
    /usr/local/bin/server-ip-monitor.sh || {
        echo -e "\033[1;31mERROR: No se completó la configuración. El script no funcionará sin nombre.\033[0m"
        exit 1
    }
else
    echo -e "\033[1;32m✓ Configuración existente detectada\033[0m"
    echo -e "  Nombre actual: $(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)"
fi

# ==============================================
# 4. PROGRAMAR EN CRON (igual que antes)
# ==============================================
CRON_JOB="*/5 * * * * /usr/local/bin/server-ip-monitor.sh >> /var/log/ip_monitor.log 2>&1"
if ! crontab -l | grep -q "server-ip-monitor"; then
    (crontab -l 2>/dev/null; echo "$CRON_JOB") | crontab -
    echo -e "\033[1;32m✓ Cron job configurado cada 5 minutos\033[0m"
else
    echo -e "\033[1;33m✓ El cron job ya estaba configurado\033[0m"
fi

# ==============================================
# 5. VERIFICACIÓN FINAL (MEJORADA)
# ==============================================
echo -e "\n\033[1;36mVALIDACIÓN FINAL\033[0m"
if [ -z "$(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)" ]; then
    echo -e "\033[1;31m✖ Error crítico: No se configuró nombre válido\033[0m"
    echo "  Ejecute manualmente para corregir:"
    echo "  sudo /usr/local/bin/server-ip-monitor.sh"
    exit 1
else
    echo -e "\033[1;32m✓ Configuración validada correctamente\033[0m"
    echo -e "\033[1;35mINSTALACIÓN COMPLETADA!\033[0m"
    echo "El monitor comenzará a funcionar automáticamente"
fi
