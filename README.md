#!/bin/bash
# INSTALADOR CON SOLICITUD OBLIGATORIA DE NOMBRE
# Versión 3.2 - Pide nombre manualmente siempre

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
# Server IP Monitor v3.2 - Con solicitud obligatoria de nombre

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
    echo -e "\n\033[1;36mINGRESE EL NOMBRE DE ESTE SERVIDOR\033[0m"
    echo "============================================"
    echo "  Este nombre identificará al servidor en el"
    echo "  sistema central. Debe ser:"
    echo "  - Único en su red"
    echo "  - Descriptivo (ej: 'servidor-web' o 'bd-finanzas')"
    echo "  - Sin espacios (use guiones)"
    echo "============================================"
    
    while true; do
        echo -e "\n\033[1;37mPOR FAVOR ESCRIBA EL NOMBRE MANUALMENTE:\033[0m"
        read -p "  ➤ Nombre del servidor: " SERVER_NAME
        
        # Validaciones
        if [ -z "$SERVER_NAME" ]; then
            echo -e "\033[1;31m✖ ERROR: Debe ingresar un nombre\033[0m"
        elif [[ "$SERVER_NAME" =~ [[:space:]] ]]; then
            echo -e "\033[1;31m✖ ERROR: No use espacios (reemplace por guiones)\033[0m"
        elif [[ "$SERVER_NAME" =~ [^a-zA-Z0-9_\-] ]]; then
            echo -e "\033[1;31m✖ ERROR: Solo letras, números, guiones (-) y guiones bajos (_)\033[0m"
        else
            # Guardar configuración
            echo "SERVER_NAME='$SERVER_NAME'" > "$CONFIG_FILE"
            chmod 600 "$CONFIG_FILE"
            echo -e "\033[1;32m✔ Nombre guardado: $SERVER_NAME\033[0m"
            return 0
        fi
        
        echo -e "\n\033[1;33mINTENTE NUEVAMENTE:\033[0m"
    done
}

# Cargar o solicitar nombre
if [ -f "$CONFIG_FILE" ]; then
    source "$CONFIG_FILE"
else
    solicitar_nombre || {
        echo -e "\033[1;31mFATAL: No se configuró el nombre del servidor\033[0m"
        exit 1
    }
fi

[ -z "$SERVER_NAME" ] && {
    echo -e "\033[1;31mERROR: El nombre del servidor no está configurado\033[0m"
    exit 1
}

# ... (resto del script original igual)
EOSCRIPT

# ==============================================
# 2. CONFIGURACIÓN INICIAL OBLIGATORIA
# ==============================================
echo -e "\n\033[1;36mCONFIGURACIÓN INICIAL REQUERIDA\033[0m"
if [ -f "/etc/ip-monitor.conf" ]; then
    echo -e "\033[1;33mADVERTENCIA: Ya existe una configuración previa\033[0m"
    CURRENT_NAME=$(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)
    echo -e "  Nombre actual: \033[1;37m$CURRENT_NAME\033[0m"
    
    read -p "¿Desea cambiar el nombre? (s/n): " CHANGE_NAME
    if [[ "$CHANGE_NAME" =~ ^[SsYy] ]]; then
        rm -f /etc/ip-monitor.conf
        /usr/local/bin/server-ip-monitor.sh || exit 1
    fi
else
    echo -e "\033[1;33mDEBE INGRESAR EL NOMBRE DEL SERVIDOR MANUALMENTE:\033[0m"
    /usr/local/bin/server-ip-monitor.sh || {
        echo -e "\033[1;31mERROR: No se completó la configuración\033[0m"
        exit 1
    }
fi

# ==============================================
# 3. INSTALAR DEPENDENCIAS Y CONFIGURAR PERMISOS
# ==============================================
chmod 700 /usr/local/bin/server-ip-monitor.sh
chown root:root /usr/local/bin/server-ip-monitor.sh
mkdir -p /etc/
touch /var/log/ip_monitor.log
chmod 644 /var/log/ip_monitor.log

if ! command -v curl &> /dev/null; then
    echo -e "\033[1;33mInstalando curl...\033[0m"
    apt-get update > /dev/null && apt-get install -y curl > /dev/null || yum install -y curl > /dev/null
fi

# ==============================================
# 4. PROGRAMAR EN CRON
# ==============================================
CRON_JOB="*/5 * * * * /usr/local/bin/server-ip-monitor.sh >> /var/log/ip_monitor.log 2>&1"
if ! crontab -l | grep -q "server-ip-monitor"; then
    (crontab -l 2>/dev/null; echo "$CRON_JOB") | crontab -
    echo -e "\033[1;32m✓ Programa configurado para ejecutarse cada 5 minutos\033[0m"
fi

# ==============================================
# 5. VERIFICACIÓN FINAL
# ==============================================
echo -e "\n\033[1;36mRESUMEN DE INSTALACIÓN\033[0m"
echo -e "\033[1;32m✓ Nombre configurado:\033[0m \033[1;37m$(grep SERVER_NAME /etc/ip-monitor.conf | cut -d"'" -f2)\033[0m"
echo -e "\033[1;32m✓ Script instalado en:\033[0m /usr/local/bin/server-ip-monitor.sh"
echo -e "\033[1;32m✓ Configuración guardada en:\033[0m /etc/ip-monitor.conf"
echo -e "\033[1;32m✓ Logs de actividad en:\033[0m /var/log/ip_monitor.log"

echo -e "\n\033[1;35mINSTALACIÓN COMPLETADA CON ÉXITO!\033[0m"
echo "El sistema comenzará a monitorear automáticamente"
