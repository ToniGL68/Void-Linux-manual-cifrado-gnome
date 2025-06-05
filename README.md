# Void-Linux-manual-cifrado-gnome
Instalación manual de Void Linux en disco cifrado con entorno de escritorio gnome

## Información de los discos
```
lsblk
```
## Borrado del disco completo (suponemos vda)
```
cryptsetup open --type plain --key-file /dev/urandom --sector-size 4096 /dev/vda discoa
dd if=/dev/zero of=/dev/mapper/discoa bs=1M status=progress
cryptsetup close discoa
```

+ Se puede ver contenido de disco
```
od -N 500 /dev/vda
```
## Particionado
```
cfdisk /dev/vda
```
+ Ejemplo vda1 para boot/efi, vda2 para swap, vda3 para sistema

## Cifrado de la partición de sistema vda3
```
cryptsetup luksFormat --type luks1 /dev/vda3
```
+ Pide asignar y confirmar la contraseña

```
cryptsetup luksOpen /dev/vda3 disco
```
+ Pide contraseña anterior y el dispositivo es mapeado en **/dev/mapper/disco**
  
## Formateo de particiones
```
mkfs.vfat /dev/vda1
mkswap /dev/vda2
mkfs.ext4 /dev/mapper/disco
```

## Montado de particiones
```
mount /dev/mapper/disco /mnt
mount /dev/vda1 /mnt/boot/efi --mkdir
swapon /dev/vda2
```
## Instalar sistema base en /mnt
```
xbps-install -S -r /mnt -R https://repo-de.voidlinux.org/current base-system grub-x86_64-efi cryptsetup nano
```
## Montar directorios necesarios en /mnt
```
for dir in dev proc sys run; do mount --rbind /$dir /mnt/$dir; done
```
## Copiar archivo para dns en /mnt/etc/
```
cp /etc/resolv.conf /mnt/etc/
```
## Hacer chroot a /mnt
```
chroot /mnt /bin/bash
```
### Configurar sistema
+ Contraseña y shell para root:
```
passwd
``` 
```
chsh -s /bin/bash
```
+ Teclado y locales
```
nano /etc/rc.conf
```
+ Descomentar la línea **KEYMAP=es**
   
```
nano /etc/locale.conf
```
+ Modificar la línea _LANG= ..._ En mi caso queda como **LANG=es_ES.UTF-8**
```
nano /etc/default/libc-locales
```
+ Descomentar la línea **es_ES.UTF-8 UTF-8**

+ Nombre para la máquina
```
nano /etc/hostname
```
+ Establecer zona horaria a **Europa/Madrid**
```
ln -s /usr/share/zoneinfo/Europe/Madrid /etc/localtime
```
+ Crear nuevo usuario y su contraseña
```
useradd -m -s /bin/bash username
passwd username
```
+ Grupos para el usuario, **wheel** para que tenga permisos de root

```
usermod -aG wheel,audio,video,lp,storage,users username
```
+ Editar /etc/sudoers para permitir acciones de root al grupo wheel
```
visudo
```
+ Descomentar la línea  **%wheel ALL=(ALL:ALL) ALL**


**Archivo fstab**
+ Averiguar el _UUID de vda1_ y copiarlo en _/etc/fstab_
```
blkid | grep vfat >> /etc/fstab
```
+ Editar el archivo _/etc/fstab_
```
nano /etc/fstab
```
+ Al final del archivo está la línea que contiene el UUID de vda1
+ Seleccionar (Shift+Flecha dirección), copiar (ALT+6) y pegar (Ctrl+U) en el lugar correcto
+ No olvidar comentar o borrar la línea que se añadió con _blkid | grep vfat >> /etc/fstab_
+ El fichero quedaría así:
```
UUID=xxxx-xxxx			/boot/efi		vfat		defaults	0		2

/dev/vda2			none			swap		defaults	0		0

/dev/mapper/disco		/			ext4		defaults	0		1
```
**Archivo /etc/default/grub**
+ Averiguar el _UUID del dispositivo encriptado /dev/vda3_ y copiarlo en _/etc/default/grub_
```
blkid | grep LUKS >> /etc/default/grub
```
+ Editar el archivo _/etc/default/grub_
```
nano /etc/default/grub
```
+ Al final del archivo está la línea que contiene el UUID del dispositivo  encriptado /dev/vda3
+ Seleccionar (Shift+Flecha dirección), copiar (ALT+6) y pegar (Ctrl+U) en el lugar correcto
+ No olvidar comentar o borrar la línea que se añadió con _blkid | grep LUKS >> /etc/default/grub_
+ Añadir la línea:
```
GRUB_ENABLE_CRYPTODISK=y
```
+ Editar la línea **GRUB_CMDLINE_LINUX_DEFAULT="loglevel=4"**
+ En esa misma línea hay que añadir dentro de las comillas:
```
rd.luks.name=uuid-del-dispositivo-LUKS=disco root=/dev/mapper/disco
```
+ Después de la modificaciones el archivo ha quedado así:
```
GRUB_DEFAULT=0
#GRUB_HIDDEN_TIMEOUT=0
#GRUB_HIDDEN_TIMEOUT_QUIET=false
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Void"
GRUB_CMDLINE_LINUX_DEFAULT="rd.luks.name=b23b875b-c9d3-45ab-b7a6-03793ba7e490=disco root=/dev/mapper/disco loglevel=4"
# Uncomment to use basic console
#GRUB_TERMINAL_INPUT="console"
# Uncomment to disable graphical terminal
#GRUB_TERMINAL_OUTPUT=console
#GRUB_BACKGROUND=/usr/share/void-artwork/splash.png
#GRUB_GFXMODE=1920x1080x32
#GRUB_DISABLE_LINUX_UUID=true
#GRUB_DISABLE_RECOVERY=true
# Uncomment and set to the desired menu colors.  Used by normal and wallpaper
# modes only.  Entries specified as foreground/background.
#GRUB_COLOR_NORMAL="light-blue/black"
#GRUB_COLOR_HIGHLIGHT="light-cyan/blue"
GRUB_ENABLE_CRYPTODISK=y
```
+ Crear archivo necesario para que arranque _/etc/dracut.conf.d/hostonly.conf
+ El nombre puede variar, pero debe mantener la extensión _.conf_
```
nano /etc/dracut.conf.d/hostonly.conf
```
+ Dentro de ese archivo añadir lo siguiente:
```
hostonly=yes
```
**Instalar grub**
```
grub-install /dev/vda1
```
**Último paso: configurar el sistema**
```
xbps-reconfigure -fa
```
**Salir de chroot**
```
exit
```
**Desmontar el sistema de ficheros y reiniciar**
```
umount -R /mnt
```
```
reboot
```
+ El sistema arranca y pide clave para descifrar el disco
+ Después del menú de grub vuelve a pedir la contraseña
+ En este punto no hay conexión a internet. Solución: Activar servicio dhcpcd
```
ln -s /etc/sv/dhcpcd /var/service
```

## Cifrar swap con clave aleatoria en cada arranque
```
nano /etc/crypttab
```
+ Copiar/Modificar la línea de swap
+ Modificar la parte que hace referencia a la partición /dev/hdX por /dev/vda2 , resto queda igual...
+ Editar de nuevo /etc/fstab para cambiar la línea de swap
```
nano /etc/fstab
```
+ Modificar la línea de swap cambiando _/dev/vda2_ por **/dev/mapper/swap**
+ Después de estos dos cambios, en el siguiente arranque, sale advertencia sobre cifrado - responder YES
  
## Evitar que pida dos veces la contraseña en el arranque
+ Esto se consigue creando una archivo que contenga una clave aleatoria en el disco y añadirla como clave válida para /dev/vda3
+ Como ejemplo se crea directorio /root/keys y archivo disco.key
```
mkdir /root/keys
```
```
dd id=/dev/random of=/root/keys/disco.key bs=1 count=128
```
+ Medidas de protección al directorio y al archivo
```
chmod 700 /root/keys
```
```
chmod 000 /root/keys/disco.key
```
+ Añadir la clave al disco cifrado
```
cryptsetup luksAddKey /dev/vda3 /root/keys/disco.key
```
+ Te pide contraseña para disco. Es la que se utiliza para el arranque.
+ Se puede comprobar que ahora hay dos claves válidas con:
```
cryptsetup luksDump /dev/vda3
```
+ Se debe ver:
```
Key Slot 0: Enabled
...
Key Slot 1: Enabled
...
```
+ Crear archivo necesario para que funcione
+ Igual que antes, el nombre puede variar, pero ha de tener la extensión .conf

```
nano /etc/dracut.conf.d/10-disco-key.conf
```

+ Añadir lo siguiente:
```
install_items+=" /root/keys/disco.key /etc/crypttab "
```
+ Añadir la clave al archivo /etc/crypttab
```
nano /etc/crypttab
```
+ Añadir lo siguiente: (disco es el nombre que se está usando para el dispositivo mapeado)
```
disco		/dev/vda3		/root/keys/disco.key	luks
```
+ En este caso el archivo ha quedado así:
```
 crypttab: mappings for encrypted partitions
#
# Each mapped device will be created in /dev/mapper, so your /etc/fstab
# should use the /dev/mapper/<name> paths for encrypted devices.
#
# NOTE: Do not list your root (/) partition here.

# <name>       <device>         <password>              <options>
# home         /dev/hda4        /etc/mypassword1
# data1        /dev/hda3        /etc/mypassword2
# data2        /dev/hda5        /etc/cryptfs.key
# swap         /dev/hdx4        /dev/urandom            swap,cipher=aes-cbc-essiv:sha256,size=256
# vol          /dev/hdb7        none
swap           /dev/vda2        /dev/urandom            swap,cipher=aes-cbc-essiv:sha256,size=256
disco          /dev/vda3        /root/keys/disco.key    luks

```
## Actualizar el arranque
+ Es necesario saber la versión de kernel
```
uname -r
```
+ En este momento la salida es **6.12.31.1**
+ Para el siguiente comando solo se utiliza el **6.12**
```
xbps-reconfigure -f linux6.12
```
+ Reiniciar para comprobar
```
reboot
```
+ Si todo ha ido bien solo pide la contraseña del disco una vez.

## Entorno gráfico gnome con red,sonido,impresoras
```
xbps-install -S xorg gnome pipewire wireplumber cups cups-filters
```
+ **Configurar pipewire y wireplumber**
```
mkdir -p /etc/pipewire/pipewire.conf.d
ln -s /usr/share/examples/wireplumber/10-wireplumber.conf /etc/pipewire/pipewire.conf.d			
ln -s /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d
```
+ Archivo para autoarranque de pipewire
+ Crear archivo pipewire.desktop en /etc/xdg/autostart
```
cd /etc/xdg/autostart
nano pipewire.desktop
```
+ Añadir lo siguiente:
```
[Desktop Entry]
Type=Application
Name=Pipewire
Exec=pipewire
```
+ Eliminar servicio dhcpcd
```
rm /var/service/dhcpcd
```
+ Activar servicios necesarios para gnome
```
ln -s /etc/sv/dbus /var/service
ln -s /etc/sv/NetworkManager /var/service
ln -s /etc/sc/gdm /var/service
```
+ A los pocos segundos arranca el entorno de escritorio
+ Si todo ha ido bien, ya aparece el símbolo de red y volumen.. (NetworkManager y pipewire están funcionando)
+ Configurar resolución de pantalla (1920x1080 16:9)
+ Configurar teclado español en las opciones de Configuración
+ Cerrar y abrir sesión para que se apliquen los cambios del teclado		
+ Comprobación de pipewire/wireplumber en terminal
```
pactl info
```
```
wpctl status
```
+ Repositorios privativos desde terminal
```
sudo xbps-install -Su void-repo-nonfree void-repo-multilib void-repo-multilib-nonfree
```
+ Actualizar la lista de paquetes
```
sudo xbps-install -Su
```
+ Códecs audio/video desde terminal
```
sudo xbps-install -Sy ffmpeg
```
+ Instalar aplicaciones desde terminal
```
sudo xbps-install -Su firefox libreoffice octoxbps
```
**Referencias consultadas:**
+ [Wiki de Arch Linux](https://wiki.archlinux.org/title/Main_page)
+ [Documentación oficial de Void Linux](https://docs.voidlinux.org/)
