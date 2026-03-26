# 🍎 Hackintosh — ASUS VivoBook X515JA

> **macOS Ventura 13 en dual boot con Windows 11 sobre un único disco SSD de 512 GB**  
> Bootloader: OpenCore 0.8.8 · Periodo: 19–22 de marzo de 2026  
> Autor: Alejandro Vázquez

---

## ⚠️ Aviso Legal

La instalación de macOS en hardware no oficial de Apple (Hackintosh) viola el acuerdo de licencia de usuario final (EULA) de Apple. Este repositorio tiene carácter **exclusivamente educativo y de documentación técnica personal**.

---

## 📋 Tabla de Contenidos

- [Resumen del Proyecto](#-resumen-del-proyecto)
- [Hardware](#-hardware)
- [Estado de Funcionalidades](#-estado-de-funcionalidades)
- [Día 1 — Virtualización y Preparación del USB](#-día-1--jueves-19-de-marzo)
- [Día 2 — Instalación de macOS y Primera Crisis](#-día-2--viernes-20-de-marzo)
- [Día 3 — Reinstalación de Windows y Batalla con el WiFi](#-día-3--sábado-21-de-marzo)
- [Aspectos Técnicos](#-aspectos-técnicos)
- [Lecciones Aprendidas](#-lecciones-aprendidas)
- [Herramientas Utilizadas](#-herramientas-utilizadas)
- [Referencias](#-referencias)

---

## 🧠 Resumen del Proyecto

Este proyecto nació de la curiosidad y el deseo de aprender. En 3 días se consiguió instalar **macOS Ventura 13 de forma nativa** en un portátil ASUS VivoBook X515JA con Intel Ice Lake, manteniendo un **dual boot estable con Windows 11** en el mismo disco SSD.

El proceso implicó:
- Virtualización de macOS en VMware como entorno de aprendizaje
- Creación manual de USB booteable con OpenCore y `macrecovery.py`
- Particionado del disco en caliente sin perder Windows 11
- Resolución de **dos crisis críticas** que dejaron el sistema completamente inutilizable
- Reinstalación completa de Windows 11 tras corrupción del gestor de arranque

---

## 💻 Hardware

### PC de escritorio (máquina host)

| Componente | Especificación |
|---|---|
| CPU | Intel Core i9-13900F |
| RAM | 32 GB DDR5 |
| GPU | NVIDIA RTX 5060 |
| Almacenamiento | 2 TB SSD NVMe |
| Sistema Operativo | Windows 11 Enterprise |

### Portátil objetivo (ASUS VivoBook X515JA)

| Componente | Especificación | Compatibilidad macOS |
|---|---|---|
| CPU | Intel Core i7-1065G7 (Ice Lake, 10ª gen) | ✅ PERFECTA |
| GPU | Intel Iris Plus Graphics (integrada) | ✅ BUENA |
| Trackpad | ELAN 1200 I2C | ✅ BUENA |
| Audio | Intel Smart Sound | ✅ BUENA |
| SSD | Micron 2450 512 GB NVMe | ✅ PERFECTA |
| WiFi | Realtek 8821CE | ❌ INCOMPATIBLE |
| BIOS | American Megatrends X515JAB.310 (UEFI) | — |

---

## ✅ Estado de Funcionalidades

| Funcionalidad | Estado | Notas |
|---|---|---|
| macOS Ventura arrancando | ✅ FUNCIONA | — |
| Windows 11 arrancando | ✅ FUNCIONA | Reinstalado desde cero |
| Dual boot con OpenCore | ✅ FUNCIONA | Sin necesidad de USB |
| Trackpad con gestos | ✅ FUNCIONA | ELAN I2C |
| Audio | ✅ FUNCIONA | Intel Smart Sound |
| Control de brillo | ✅ FUNCIONA | — |
| Batería | ✅ FUNCIONA | SMCBatteryManager |
| Internet por cable | ✅ FUNCIONA | Adaptador USB-RJ45 |
| WiFi | ❌ NO FUNCIONA | Realtek 8821CE incompatible |

---

## 📅 Día 1 — Jueves 19 de Marzo

### Virtualización en VMware Workstation 17

Como punto de partida, se virtualizó macOS Ventura en VMware para familiarizarse con el entorno antes de la instalación real.

**Configuración de la VM:**

| Parámetro | Valor |
|---|---|
| SO Guest | macOS 13 (Ventura) |
| CPUs | 8 Cores |
| RAM | 8192 MB (8 GB) |
| Disco | 150 GB SATA |
| Firmware | EFI |

Fue necesario modificar el archivo `.vmx` manualmente añadiendo:
```
smc.version = "0"
firmware = "EFI"
```

Se probaron múltiples ISOs antes de encontrar una funcional (ISO de Sequoia 15.2 corrupta, VMDK con `Info.plist` vacío causando el error `OC: Grabbed zero system-id for SB`).

### Preparación del USB con OpenCore

**EFI utilizada:** [Bhavinjain260/Asus-Vivobook15-X515JA-Opencore](https://github.com/Bhavinjain260/Asus-Vivobook15-X515JA-Opencore)

**Kexts principales:**

| Kext | Función |
|---|---|
| `Lilu.kext` | Base requerida por todos los demás |
| `VirtualSMC.kext` | Simula el chip SMC de los Mac reales |
| `WhateverGreen.kext` | Soporte gráfico Intel Iris Plus |
| `AppleALC.kext` | Audio Intel Smart Sound |
| `VoodooI2C.kext` | Trackpad ELAN 1200 I2C |
| `AsusSMC.kext` | Teclas de función especiales del ASUS |
| `SMCBatteryManager.kext` | Gestión y porcentaje de batería |

**Herramientas convencionales que fallaron con macOS:**
- ❌ **Ventoy** — Error `Invalid ISO size`
- ❌ **balenaEtcher** — Error `Missing partition table`
- ❌ **Rufus** — Error `Tipo de imagen no soportada`

**Solución:** Estructura manual del USB con `diskpart` + `DiskGenius` + `macrecovery.py`:

```bash
# Descargar recovery directamente de los servidores de Apple
python macrecovery.py -b Mac-4B682C642B45593E -m 00000000000000000 download
```

> El resultado fue un `BaseSystem.dmg` de 673 MB verificado con su `chunklist`.

**Correcciones críticas en `config.plist`:**

```xml
SystemUUID = 2168A42A-0E20-4F75-9569-B4340F04FE56
SystemSerialNumber = C02849302QXGPGXAA
MLB = C02849302QXGPGXAB
SecureBootModel = Disabled
DmgLoading = Any
boot-args = keepsyms=1 igfxfw=2 -noDC9 dc6config=0 -v
```

> ⚠️ **Obstáculo del día:** La instalación de macOS requiere internet. El portátil tiene WiFi Realtek incompatible y sin entrada RJ45. Se realizó pedido urgente de adaptador USB-RJ45 en Amazon a las 23:30.

---

## 📅 Día 2 — Viernes 20 de Marzo

### Instalación de macOS Ventura

Con el adaptador RJ45 llegado por la mañana, se arrancó el portátil desde el USB y se accedió al menú de OpenCore.

**Esquema de particiones final:**

| Partición | Contenido | Tamaño |
|---|---|---|
| disk0s1 | EFI System Partition (FAT32) | 272.6 MB |
| disk0s2 | Microsoft Reserved | 16.8 MB |
| disk0s3 | Windows 11 (NTFS) | 197.1 GB |
| disk0s4 | Macintosh HD (APFS) | 314.7 GB |

La instalación tardó ~45 minutos con múltiples reinicios automáticos descargando todo desde los servidores de Apple.

**Hardware funcionando tras la instalación:** CPU, GPU Intel Iris Plus, trackpad ELAN con gestos, audio Intel Smart Sound, control de brillo y batería. ✅

### Transferencia de OpenCore al disco interno

```bash
sudo diskutil mount disk0s1
sudo cp -r /Volumes/NO\ NAME/EFI /Volumes/SYSTEM/
# Verificación:
ls /Volumes/SYSTEM/EFI/
# Resultado: Boot  Microsoft  OC
```

A partir de este momento, el portátil arrancaba directamente sin necesidad del USB externo.

### 🔴 Crisis 1: Sistema completamente bloqueado

Al intentar instalar el kext WiFi `AirportItlwm`, el kext se copió con estructura interna incompleta (faltaba `Contents/Info.plist`). Resultado:

```
OC: Plist Kexts/AirportItlwm.kext/Contents/Info.plist is missing for injected kext
Halting on critical error
```

**Consecuencia:** Ni macOS ni Windows arrancaban. El portátil era completamente inutilizable.

### 🔴 Crisis 2: Windows también roto

OpenCore bloqueaba todos los sistemas operativos. El único acceso era a través de `OpenShell.efi`.

### Recuperación vía Terminal de Recovery

```bash
# Desde la Terminal del recovery de macOS:
sudo diskutil mount disk0s1
sudo rm -rf /Volumes/SYSTEM/EFI/OC/Kexts/AirportItlwm.kext
```

> ✅ macOS recuperado. ❌ Windows irrecuperable — gestor de arranque corrupto. Fin del día: 3:00 AM.

---

## 📅 Día 3 — Sábado 21 de Marzo

### Reinstalación de Windows 11

Con Windows completamente inutilizable, fue necesario reinstalarlo desde cero. El proceso supuso ~3-4 horas de pérdida: instalación, activación, aplicaciones y traspaso de archivos personales.

### Batalla con el WiFi

Esta vez, conociendo el error anterior, se verificó la estructura del kext antes de copiarlo:

```
AirportItlwm.kext/
└── Contents/
    ├── Info.plist   ← CRÍTICO: debe existir
    └── MacOS/
        └── AirportItlwm   ← Ejecutable binario
```

El kext se instaló correctamente y el WiFi apareció en Preferencias del Sistema, pero:

```bash
networksetup -setairportpower en0 on
# Resultado: You cannot set Wi-Fi power because all AirPort network services are disabled.
```

> **Causa raíz:** `AirportItlwm` está diseñado para chips **WiFi Intel**. El portátil ASUS tiene un chip **Realtek 8821CE** que es incompatible con macOS independientemente del kext instalado.

---

## 🔧 Aspectos Técnicos

### Estructura de OpenCore en la partición EFI

```
EFI/
├── BOOT/
│   └── BOOTx64.efi          ← Punto de entrada EFI
└── OC/
    ├── OpenCore.efi          ← Bootloader principal
    ├── config.plist          ← Configuración completa
    ├── ACPI/                 ← Tablas ACPI personalizadas (SSDTs)
    ├── Drivers/              ← Drivers EFI
    │   ├── AudioDxe.efi
    │   ├── HfsPlus.efi
    │   ├── OpenCanopy.efi
    │   └── OpenRuntime.efi
    ├── Kexts/                ← Extensiones del kernel (30+ kexts)
    │   ├── Lilu.kext
    │   ├── VirtualSMC.kext
    │   ├── WhateverGreen.kext
    │   └── ...
    ├── Resources/            ← Temas visuales
    └── Tools/
        └── OpenShell.efi     ← Herramienta de recuperación de emergencia
```

### PlatformInfo — Identidad del Mac simulado

| Campo | Valor |
|---|---|
| SystemProductName | MacBookAir9,1 (MacBook Air 2020 Intel) |
| SystemSerialNumber | C02849302QXGPGXAA |
| MLB | C02849302QXGPGXAB |
| SystemUUID | 2168A42A-0E20-4F75-9569-B4340F04FE56 |
| SecureBootModel | Disabled |

> Se eligió `MacBookAir9,1` por ser el modelo más compatible para hardware Ice Lake: el MacBook Air 2020 con Intel usa exactamente el mismo CPU i7-1065G7.

---

## 📚 Lecciones Aprendidas

1. **Siempre hacer copia de seguridad del `config.plist`** antes de cualquier modificación
2. **Verificar la estructura interna de los kexts** antes de copiarlos a la EFI
3. macOS requiere internet para instalarse — **preparar el adaptador ethernet con antelación**
4. **`OpenShell.efi` es una herramienta de emergencia invaluable** para recuperar sistemas bloqueados
5. La creación de USB de macOS desde Windows requiere **`macrecovery.py`**, no herramientas convencionales
6. `SecureBootModel` debe estar en `Disabled` para instalaciones Hackintosh
7. Ante un sistema bloqueado, la **paciencia y el método** son más importantes que la velocidad

---

## 🛠 Herramientas Utilizadas

| Herramienta | Uso | Resultado |
|---|---|---|
| VMware Workstation 17 | Virtualización de macOS | ✅ Éxito |
| VMware Unlocker | Habilitar soporte macOS en VMware | ✅ Éxito |
| DiskGenius | Gestión de particiones EFI | ✅ Éxito |
| macrecovery.py | Descarga del recovery de Apple | ✅ Éxito |
| OpenCore 0.8.8 | Bootloader principal | ✅ Éxito |
| OpenShell.efi | Recuperación de emergencia | ✅ Éxito |
| balenaEtcher / Rufus | Creación del USB | ❌ Fallo con macOS |
| Ventoy | Creación del USB | ❌ Fallo con macOS |
| OCAuxiliaryTools | Edición del config.plist | ⚠️ Uso parcial |

---

## 🔗 Referencias

### Repositorios de GitHub
- [Bhavinjain260/Asus-Vivobook15-X515JA-Opencore](https://github.com/Bhavinjain260/Asus-Vivobook15-X515JA-Opencore) — EFI principal utilizada
- [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg) — Proyecto OpenCore oficial
- [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm) — Kexts WiFi Intel para macOS
- [paolo-projects/auto-unlocker](https://github.com/paolo-projects/auto-unlocker) — Unlocker de VMware

### Documentación
- [dortania.github.io/OpenCore-Install-Guide](https://dortania.github.io/OpenCore-Install-Guide/) — Guía oficial de instalación

---

> **Proyecto completado:** 22 de marzo de 2026  
> macOS Ventura 13 + Windows 11 · ASUS VivoBook X515JA · OpenCore 0.8.8  
> *Autor: Alejandro Vázquez*
