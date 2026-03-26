# Diario de instalación — Hackintosh ASUS X515JA

> Documentación técnica cronológica completa del proceso de instalación.  
> Para el resumen ejecutivo y las decisiones técnicas, ver el [README principal](../README.md).

---

## Día 1 — Jueves 19 de marzo

### Virtualización en VMware Workstation 17

El jueves comenzó con la virtualización de macOS como entorno de aprendizaje previo a la instalación real. Configuración final de la VM:

| Parámetro | Valor |
|---|---|
| SO Guest | macOS 13 (Ventura) |
| CPUs | 1 procesador · 8 Cores |
| RAM | 8192 MB (8 GB) |
| Disco | 150 GB SATA · archivo único |
| Red | NAT |
| Firmware | EFI |

Modificación manual del `.vmx` para permitir el arranque de macOS en VMware:

```
smc.version = "0"
firmware = "EFI"
```

**Intentos fallidos con imágenes:**

- **ISO Sequoia 15.2 (Archive.org, 29 GB):** Descargada, corrupta. Etcher y Rufus no la reconocían como booteable.
- **VMDK de Ventura (Classroom ASIR 1):** Hash SHA256 correcto, pero el archivo `.mf` causaba errores de verificación al importar en VMware.
- **OVF de Ventura:** Importado correctamente, pero error `No compatible Bootloader found` al arrancar.

**Error crítico encontrado con el VMDK:**
```
OC: Grabbed zero system-id for SB, this is not allowed
Halting on critical error
```
Causa: El `config.plist` del VMDK tenía `SystemUUID` vacío.

**Solución:** ISO de macOS Ventura 13 funcional. Pasos:
1. Formatear disco virtual de 150 GB como APFS con esquema GUID en Utilidad de Discos
2. Asignar la ISO como unidad CD/DVD virtual
3. Instalación de ~45 minutos con varios reinicios automáticos
4. Instalación de VMware Tools para mejorar rendimiento

Se virtualizó también macOS Sonoma en una segunda VM para comparar entornos. ✅

---

### Preparación del USB para Hackintosh

**Análisis de compatibilidad del hardware (MSInfo32):**

| Componente | Detalle | Compatibilidad |
|---|---|---|
| CPU | Intel i7-1065G7 Ice Lake | ✅ PERFECTA |
| GPU | Intel Iris Plus Graphics | ✅ BUENA |
| Trackpad | ELAN 1200 I2C | ✅ BUENA |
| Audio | Intel Smart Sound | ✅ BUENA |
| SSD NVMe | Micron 2450 512 GB | ✅ PERFECTA |
| WiFi | Realtek 8821CE | ❌ INCOMPATIBLE |

**EFI descargada:**  
[Bhavinjain260/Asus-Vivobook15-X515JA-Opencore](https://github.com/Bhavinjain260/Asus-Vivobook15-X515JA-Opencore) — configurada para macOS Sonoma 14, con todos los kexts necesarios.

**Herramientas que fallaron:**

| Herramienta | Error |
|---|---|
| Ventoy | `Invalid ISO size` |
| balenaEtcher | `Missing partition table` |
| Rufus | `Tipo de imagen no soportada` |

**Proceso manual de creación del USB:**

```bash
# 1. Preparación con diskpart
diskpart
> clean
> convert gpt

# 2. Partición EFI de 1 GB en FAT32 via DiskGenius
# 3. Partición de datos en exFAT para el resto del USB
# 4. Copia de la EFI de OpenCore a la partición EFI via DiskGenius

# 5. Descarga del recovery de macOS directamente de servidores Apple
python macrecovery.py -b Mac-4B682C642B45593E -m 00000000000000000 download
# Resultado: BaseSystem.dmg (673 MB) + BaseSystem.chunklist

# 6. Copiar a com.apple.recovery.boot en la partición de datos
```

**Correcciones en el `config.plist`:**

| Campo | Problema | Valor corregido |
|---|---|---|
| `SystemUUID` | Vacío → error `Grabbed zero system-id` | `2168A42A-0E20-4F75-9569-B4340F04FE56` |
| `SystemSerialNumber` | Vacío → macOS no identifica el modelo | `C02849302QXGPGXAA` |
| `MLB` | Vacío → identificación de hardware incompleta | `C02849302QXGPGXAB` |
| `SecureBootModel` | `Default` → bloqueaba el arranque con Secure Boot | `Disabled` |
| `DmgLoading` | `Signed` → rechazaba el BaseSystem.dmg | `Any` |
| `boot-args` | Sin flags de depuración | `keepsyms=1 igfxfw=2 -noDC9 dc6config=0 -v` |

**Obstáculo final del día (23:30):**  
La instalación de macOS requiere conexión a internet. El portátil no tiene WiFi compatible ni entrada RJ45. Sin adaptador USB-RJ45, imposible continuar. Pedido realizado en Amazon con entrega al día siguiente.

---

## Día 2 — Viernes 20 de marzo

### Llegada del adaptador y primer arranque

A primera hora llegó el adaptador USB-A a RJ45. Con ethernet conectado, primer arranque desde el USB → menú de OpenCore → selección del recovery (NO NAME dmg) → código verbose en pantalla → **menú de recuperación de macOS en el portátil ASUS**. ✅

### Particionado del disco

Con Windows 11 ya instalado en el único SSD de 512 GB, se creó la partición de macOS desde Utilidad de Discos en el recovery:

1. Vista → "Mostrar todos los dispositivos"
2. Seleccionar el SSD completo (`disk0`)
3. Nueva partición de 314 GB en APFS → nombre `Macintosh HD`
4. Confirmación sin tocar las particiones de Windows existentes

### Instalación de macOS Ventura

~45 minutos con múltiples reinicios. El instalador descargó todos los archivos desde los servidores de Apple vía ethernet USB.

Hardware funcionando tras la instalación: CPU, GPU Intel Iris Plus, trackpad ELAN con gestos, audio Intel Smart Sound, control de brillo y batería. ✅

### Transferencia de OpenCore al disco interno

```bash
sudo diskutil mount disk0s1
sudo cp -r /Volumes/NO\ NAME/EFI /Volumes/SYSTEM/

# Verificación
ls /Volumes/SYSTEM/EFI/
# → Boot  Microsoft  OC
```

El portátil ya arrancaba directamente sin USB, mostrando el menú de OpenCore con opciones: Windows, Macintosh HD, Recovery, OpenShell y Reset NVRAM. ✅

### Crisis 1 — Sistema completamente bloqueado (23:00)

Intento de instalación del kext WiFi `AirportItlwm`:

```bash
curl -L -o ~/Downloads/AirportItlwm_Ventura.zip \
  "https://github.com/OpenIntelWireless/itlwm/releases/download/v2.3.0/AirportItlwm_v2.3.0_stable_Ventura.kext.zip"
sudo diskutil mount disk0s1
sudo cp -r ~/Downloads/AirportItlwm.kext /Volumes/SYSTEM/EFI/OC/Kexts/
```

Al reiniciar:
```
OC: Plist Kexts/AirportItlwm.kext/Contents/Info.plist is missing for injected kext AirportItlwm.kext
Halting on critical error
```

El kext se copió sin su estructura interna. Resultado: ni macOS ni Windows arrancaban.

### Crisis 2 — Windows también roto

OpenCore no podía inicializar y bloqueaba todos los sistemas. El único acceso: menú de OpenCore → `OpenShell.efi`.

### Recuperación

Navegación desde OpenShell (comandos EFI limitados) hasta lograr acceso a la Terminal del recovery de macOS:

```bash
sudo diskutil mount disk0s1
sudo rm -rf /Volumes/SYSTEM/EFI/OC/Kexts/AirportItlwm.kext
```

macOS recuperado. ✅  
Windows: gestor de arranque corrupto, irrecuperable. ❌  
Fin del día: 3:00 AM.

---

## Día 3 — Sábado 21 de marzo

### Reinstalación de Windows 11

~3-4 horas: instalación limpia sobre `disk0s3` (197 GB), preservando EFI intacta. Activación, aplicaciones, traspaso de archivos personales. Dual boot verificado. ✅

### Segunda instalación del kext WiFi (con verificación previa)

Esta vez, verificación de estructura antes de copiar:

```bash
# Estructura requerida (verificada antes de copiar)
AirportItlwm.kext/
└── Contents/
    ├── Info.plist        ← DEBE EXISTIR
    └── MacOS/
        └── AirportItlwm

# Descarga de ambas versiones para prueba
curl -L -o ~/Downloads/AirportItlwm_Ventura.zip \
  "https://github.com/OpenIntelWireless/itlwm/releases/download/v2.3.0/AirportItlwm_v2.3.0_stable_Ventura.kext.zip"
curl -L -o ~/Downloads/AirportItlwm_Sonoma.zip \
  "https://github.com/OpenIntelWireless/itlwm/releases/download/v2.3.0/AirportItlwm_v2.3.0_stable_Sonoma.kext.zip"

sudo diskutil mount disk0s1
sudo cp -r ~/Downloads/AirportItlwm.kext /Volumes/SYSTEM/EFI/OC/Kexts/
```

El kext se instaló sin errores. El WiFi apareció en Preferencias del Sistema con estado "desactivado", pero no era posible activarlo:

```bash
networksetup -setairportpower en0 on
# You cannot set Wi-Fi power because all AirPort network services are disabled.
```

Causa confirmada: `AirportItlwm` es exclusivo para chips Intel. El portátil tiene Realtek 8821CE. No hay solución por software disponible.

---

## Día 4 — Domingo 22 de marzo

Día dedicado íntegramente a la documentación técnica del proyecto.

---

*Ver [README principal](../README.md) para el resumen ejecutivo y las decisiones técnicas.*
