#!/bin/bash

# Variables de configuración
TIMEZONE="America/Lima"
ARCHIVO_A_DESCARGAR="https://guia.mitv.plus/file/instalador.sh.x"
DESTINO_ARCHIVO="/root/instalador.sh.x"
MD5_ESPERADO="[Inserta aquí el hash MD5 CORRECTO del archivo]"  # Reemplaza con el hash real

# Función para mostrar un mensaje de error y salir
error() {
  echo "ERROR: $1" >&2
  exit 1
}

# Actualizar e instalar shc
echo "Actualizando la lista de paquetes..."
sudo apt update || error "No se pudo actualizar la lista de paquetes."

echo "Instalando shc..."
sudo apt install -y shc || error "No se pudo instalar shc."
echo "shc instalado correctamente."

# Verificar si la zona horaria existe
if ! timedatectl list-timezones | grep -q "$TIMEZONE"; then
  error "Zona horaria '$TIMEZONE' no encontrada."
fi

# Establecer la zona horaria
sudo timedatectl set-timezone "$TIMEZONE" || error "No se pudo establecer la zona horaria."
echo "Zona horaria establecida a $TIMEZONE."

# Descargar el archivo
if ! command -v wget &> /dev/null
then
    echo "wget no está instalado. Intentando instalar..."
    sudo apt update
    sudo apt install -y wget
fi

if ! sudo wget "$ARCHIVO_A_DESCARGAR" -O "$DESTINO_ARCHIVO"; then
  error "No se pudo descargar el archivo desde $ARCHIVO_A_DESCARGAR."
fi
echo "Archivo descargado a $DESTINO_ARCHIVO."

# Verificar el hash MD5
MD5_OBTENIDO=$(md5sum "$DESTINO_ARCHIVO" | awk '{print $1}')
if [ "$MD5_OBTENIDO" != "$MD5_ESPERADO" ]; then
  error "Error: El hash MD5 del archivo descargado no coincide.  El archivo puede estar corrupto o haber sido modificado."
fi
echo "Hash MD5 verificado."

#Dar permisos de ejecucion
sudo chmod +x "$DESTINO_ARCHIVO"

# Ejecutar el archivo descargado
echo "Iniciando la ejecucion del archivo descargado: $DESTINO_ARCHIVO"
sudo "$DESTINO_ARCHIVO"
EXIT_CODE=$?
if [ "$EXIT_CODE" -ne 0 ]; then
  error "Error al ejecutar $DESTINO_ARCHIVO (código de salida: $EXIT_CODE)"
fi
echo "Ejecución de $DESTINO_ARCHIVO completada."

echo "Script completado."
exit 0
