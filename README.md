# Arch Linux

## SETUP FROM SCRATCH

ARCH is a Linux Distribution, whose philosophy is to install everything by hand, and for that reason is very important to be neat in order to undestand what are you doing in each step.

This guide, pretends to be a DevLog for myself, to create a reusable Arch installation using parameters that I consider apropiate for a WORKSTATION to development and main work.

### DOWNLOAD LINK

First place, of course, you need to download the ISO file with the tools and package for a BASE installation

LINK: [https://archlinux.org/download/](https://archlinux.org/download/)

### PREPARING USB

The downloaded ISO, can be installed in any USB Drive, using RUFUS if you are in Windows.

- MANUAL INSTALL FROM LINUX
    
    Alternatively, you can burn the ISO file manually from Linux doing the next steps:
    
    - CREATE AN UNIQUE FAT32 PARTITION IN USB DRIVE
    
    ```bash
    
    # fdisk usage
    # WARNING: If you need more details for fdisk usage or not sure what are you doing, better investigate and play with disk you are not afraid to lose.
    # Failed operations here can delete an entire DISK and screw up all your important work.
    
    fdisk -l # List Disks and Devices
    fdisk /dev/sdX # Open a promt to edit the X Disk
    # Commands that you can use in the fdisk prompt
    # g - Overwrites a new GPT partition table
    # n - Create a new partition with a numbrer, and starting and ending block (setting the size)
    # p - Print the current partition table
    # w - Writes the changes and exit.
    
    ```
    
    - FIND THE PARTITION, MOUNT AND EXTRACT THE ISO THERE:
    
    ```bash
    mount /dev/disk/by-id/usb-My_flash_drive-partn /mnt
    bsdtar -x -f archlinux-version-x86_64.iso -C /mnt
    ```
    
    - BE SURE THAT SYSLINUX AND MTOOLS PACKAGE ARE INSTALLED IN YOUR LINUX HOST AND RUN THIS TO FINISH THE SETUP OF USB:
        
        ```bash
        umount /mnt
        syslinux --directory boot/syslinux --install /dev/disk/by-id/usb-My_flash_drive-partn
        dd bs=440 count=1 conv=notrunc if=/usr/lib/syslinux/bios/gptmbr.bin of=/dev/disk/by-id/usb-My_flash_drive
        
        # Replace gptmbr.bin by mbr.bin if you are using a MBR Partition table instead GPT.
        ```
        

### STARTING ARCH USB LIVE

Conectar el USB en el equipo a instalar y seleccionarlo cono dispositivo de booteo desde el BIOS o BOOTLOADER del equipo.

Luego en la pantalla de selección de inicio seleccionar Arch installation media y esperar a que lance el PROMT de zsh. Si se quiere usar otra consola se puede usar *Alt + flecha*

<aside>
💡

Para que funcione correctamente el USB Live se debe deshabilitar la opción de Secure Boot temporalmente. Una vez hecha la instalación puede volver a habilitarse.

</aside>

### (Opcional) Modificar distribución de teclado

### Verificar si booteamos en UEFI

```bash
cat /sys/firmware/efi/fw_platform_size
```

Si el resultado es 64 o 32, el sistema booteo en UEFI, caso contrario estamos en modo BIOS.

### Conexion a Internet

<aside>
💡

Arch es un Sistema Operativo con updates Rolling Release, por lo tanto NO soporta mantener los paquetes desactualizados. Por ello es requerido que la instalacion descargue los paquetes actualizados usando Pacman via Internet. Se puede instalar sin acceso a INET, pero requiere de otro Linux en la misma red que si tenga.

</aside>

Se puede verificar las placas de red con el comando:

```bash
ip link
```

Luego para realizar una conexión WiFi se pueden usar las siguientes utilidades:

```bash
rfkill # Verifica si hay algun bloqueo a nivel hardware o Kernel (como el boton para apagar wifi)
iwctl # Utilidad interafctiva para conectar WiFi. Comenzar con 'help' para el detalle. Ctrl + d cancela el prompt
```

### Particionando el disco con LVM

Para tener flexibilidad a la hora de gestionar el tamaño de los Filesystems, es recomendable utilizar LVM, de esta manera se abstraen las particiones de los discos y ubicaciones físicas.

En primer lugar hay que particionar los discos con el espacio necesario. En caso de optar por un bootloader distinto a GRUB, se debe dejar la partición /boot por fuera de LVM con un tamaño de 1 GB.

El esquema de particiones seria el siguiente:

- /boot - FAT32 - 1 GB (UEFI y loaders)
- / - EXT4 - 100 GB - (Archivos Linux)
- /home - NTFS - 50 GB? (Archivos personales)
- /backup - BTRFS - 50 GB?(Backups del sistema)

Para la configuración de LVM solo necesitamos por medio de fdisk configurar la partición boot y el resto del espacio con el partition type en Linux LVM.

Creación de Physical Volume:

Este comando inicializa una partición “real” para ser usada en LVM

```bash
lvmdiskscan #Escanea los discos disponibles
pvcreate /dev/sdX #Crea el PV en el dispositivo elegido
pvdisplay # Muestra PV existentes
pvscan #Show info of PV
```

Creación de VolumeGroup:

Esta sección asigna PVs a un grupo lógico que representa un disco en LVM

```bash
vgcreate VolGroupXX /dev/sdaX # Crea el VG con la PV sdX. pueden añadirse mas concatenandolas como argumentos
vgextend volume_group physical_volume # Permite añadir PVs a un VG ya existente
vgdisplay # Muestra los VG y su composición
```

Creacion de LogicalVolume:

```bash
# Crea el LV (particion en Linux) con el espacio del VG elegido
lvcreate -L size volume_group -n logical_volume 
# Muestra info de los LV creados
lvdisplay 
# Cambia el tamaño tomando espacio del VG. 
#Dependiendo del Filesystem luego hay que resizear eso tambien
lvresize -L +2G MyVolGroup/mediavol 

# Resize ext
resize2fs /dev/MyVolGroup/mediavol NewSize
# Para NTFS y BTRFS buscar como hacerlo
```

Una vez creados los espacios, crear los Filesystems que se necesiten y montarlos:

```bash
# Crear y montar ext4
mkfs.ext4 /dev/VolGroupXX/logivVolumeName
mount /dev/root_partition /mnt
# Crear espacio SWAP y habilitarlo
mkswap /dev/swap_partition
swapon /dev/swap_partition
# Crear UEFI y montarlo
mkfs.fat -F 32 /dev/efi_system_partition
mount --mkdir /dev/efi_system_partition /mnt/boot
```

Posteriormente, se debe realizar los links entre las particiones creadas desde el ArchInstaller y el sistema ya instalado, lo retomaremos luego (Agregar link a esa parte)

### Instalacion de paquetes BASE

Verificar que el archivo /etc/pacman.d/mirrorlist tenga MIRRORS que sean coherentes ya que es la que se copiara luego al sistema instalado.

Instalacion de paquetes escenciales:

```bash
pacstrap -K /mnt base linux linux-firmware
```

Una vez finalizada esta instalacion pueden comenzarse a instalarse otros paquetes que se consideren escenciales para comenzar a trabajar en el sistema. Considerar en particular los siguientes:

- **amd-ucode / intel-ucode (Para corregir posibles bugs del micro)**
- Utilidades para gestionar los FS creados
    - e2fsprogs
    - btrfs-progs
    - exfatprogs
    - ntfs-3g
- Utilidad para LVM - lvm2
- Drivers especificos para networking o GPUs
    - AMDGPU (mesa - vulkan-radeon)
    - NetworkManager (networkmanager)
- Editor de texto (nano)
- Paquetes para acceder a los manuales (man-db - man-pages - texinfo)

### Configurar el sistema pre-booteo

Primero generar el fstab con los puntos de montaje:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Configurar hora, configuraciones de region y redes:

```bash
# Entrar al root del nuevo sistema
arch-chroot /mnt/newInstall

# Configuracion hora. Ante la duda listar la carpeta zoneinfo primero
ln -sf /usr/share/zoneinfo/nombreRegion/nombreCiudad /etc/localtime
# Configurar la hora del reloj de hardware al reloj del SO
hwclock --systohc
# Verificar configuracion de NTP. Si este paquete no esta hay que instalarlo
timedatectl show-timesync --all 

# Editar el archivo /etc/locale.gen y descomentar los idiomas que se usaran. Luego ejecutar:
locale-gen
#Luego crear el archivo /etc/locale.conf y setear la linea LANG=en_US.UTF8 o segun corresponda

# Para cambiar la distribucion del teclado editar /etc/vconsole.conf con KEYMAP=de-latin1 segun corresponda
# Para verificar cuales son las configuraciones disponibles de teclado
localectl list-keymaps

# Para las fuentes ver la carpeta /usr/share/kbd/consolefonts/ y setearlas con el comando:
setfont ter-132b # O la fuente que elijas

# Configurar hostname en el archivo /etc/hostname
# Recordar dejar el networkmanager como servicio
systemctl enable NetworkManager.service
```

Configuración initram: en este paso, si configuramos LVM hay que tener en cuenta agregarlo a los HOOKS de mkinit… También tener en cuenta si se quiere usar systemd como gestor de los scripts de arranque, hay que adaptarlo según la tabla de configs comunes en el articulo de mkinit de ARCH, o bien hacer prueba de los distintos hooks.

Una vez configurado ejecutar mkinit en el sistema chrooteado.

Instalar Bootloader:

Eligiendo GRUB como bootloader. En principio verificar que haya una partición disponible en alguno de los formatos compatibles (FAT32 el mas común). Luego verificar que la partición este montada, instalar el paquete GRUB, y luego instalar el GRUB en la partición de UEFI. 

(Insertar código con el grub-install)

Posteriormente, hay que generar el archivo de configuración de GRUB 

(Insertar código con mkgrubconfig)

Una vez realizado esto, reiniciar y verificar que el sistema bootee desde el disco con GRUB.

## Configuraciones y consideraciones post-instalacion

### Protegiendo user root y seguridad

Configuracion doas:

```bash
#En el archivo /etc/doas.conf
permite setenv {PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin} username
#Esta configuracion permite correr cualquier comando como root para el username designado. 
#Tambien pueden configurarse grupos poniendo :groupname (ej: :wheel)

#Luego segurizar el archivo para que solo pueda leerlo root
chown root /etc/doas.conf
chmod 0400 /etc/doas.conf
```

passwd -l root → BLOQUEA USER ROOT

### Configuracion entorno escritorio

Luego de instalar plasma-desktop y sddm

systemctl enable sddm

### ANEXO PAQUETES USADOS:

Paquetes BASE

| Nombre Paquete  | Funcion | Necesario para Instalacion |
| --- | --- | --- |
| base |  | X |
| linux | Instala Kernel y paquetes booteo | X |
| linux-firmware | Drivers para el Kernel Linux | X |
| lvm2 | Tools y core para particionar con LVM | X |
| e2fsprogs | Tools para formatear en ext |  |
| btrfs-progs | Tools para formatear en BTRFS |  |
| exfatprogs | Tools para formatear en FAT |  |
| networkmanager | Gestor de redes mas completo | X |
| nano | Editor de texto por consola | X |
| man-db | Base de datos paginas MANUALES | X |
| man-pages | Paginas en si misma de MANUALSE | X |
| texinfo |  | X |
| grub | Bootloader con mayoria de funciones | X |
| opendoas o sudo | Gestor de usuarios privilegiados (simil sudo) | X |
| plasma-desktop | Instalacion minima de KDE con Windows Manager y algunos plugins |  |
| sddm y sddm-kcm | Display Manager para lanzar la sesion graficamente y su configuracion en KDE Settings |  |
| konsole | Emulador de consola nativo KDE |  |
| kwallet-pam | Autologin para KWallet (gestor de secretos como claves wifi y credenciales SSH) |  |
| plasma-nm | GUI para gestor redes |  |
| kscreen | Manejo de resoluciones |  |
| dolphin | Gestor de archivos |  |
| firefox | Navegador Web |  |
| pulseaudio | Servidor de sonido |  |
| plasma-pa | GUI de KDE para modificar configuracion de volumen y dispositivos sonido |  |
| wget y curl | Terminales para peticiones HTTP |  |
| base-devel | Para buildear paquetes AUR a manopla |  |

OPCIONALES / RECOMENDADOS

| Nombre Paquete  | Funcion | Necesario para Instalacion |
| --- | --- | --- |
| code | Vistual Studio Code paquete OpenSource |  |
| bash-completion | Autocompletado comandos BASH |  |
| git | Control de versiones y paquetes |  |
| openssh | Gestion de claves para Git y otros servicios |  |
