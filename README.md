# Void-Linux-manual-cifrado-gnome-Void-Linux-gnome
Instalación manual de Void Linux en disco cifrado con entorno de escritorio gnome (en construcción)


## Información de los discos
	lsblk
## Borrado del disco completo (suponemos vda)
	cryptsetup open --type plain -d /dev/urandom /dev/vda disco
	dd if=/dev/zero of=/dev/mapper/disco bs=1M status=progress
	cryptsetup close disco
 **Se puede ver contenido de disco**
 
	od -N 500 /dev/vda
## Particionado
	cfdisk /dev/vda
**Ejemplo vda1 para boot/efi, vda2 para swap, vda3 para sistema**

## Cifrado de la partición de sistema vda3
	cryptsetup luksFormat --type luks1 /dev/vda3
**Pide dos veces la contraseña para el disco**

	cryptsetup luksOpen /dev/vda3 disco
**Pide contraseña anterior y el dispositivo es mapeado en /dev/mapper/disco**
## Formateo de particiones
	mkfs.vfat /dev/vda1
	mkswap /dev/vda2
	mkfs.ext4 /dev/mapper/disco
## Montado de particiones
	mount /dev/mapper/disco /mnt
	mount /dev/vda1 /mn/boot/efi --mkdir
	swapon /dev/vda2
## Instalar sistema base en /mnt
	xbps-install -S -r /mnt -R https://repo-de.voidlinux.org/current base-system grub-x86_64-efi cryptsetup nano
## Montar directorios necesarios en /mnt
	for dir in dev proc sys run; do mount --rbind /$dir /mnt/$dir; done
## Copiar archivo de dns en /mnt/etc/
	cp /etc/resolv.conf /mnt/etc/
## Hacer chroot a /mnt
	chroot /mnt /bin/bash	
### Configurar sistema
      passwd  (contraseña root)
			chsh -s /bin/bash	(shell por defecto para root en el próximo login)
			nano /etc/rc.conf
				Descomentar la línea KEYMAP=es
			nano /etc/locale.conf
				LANG=es_ES.UTF-8
			nano /etc/default/libc-locales
				Descomentar la línea es_ES.UTF-8 UTF-8					
			nano /etc/hostname
				Escribir el nombre para la máquina
			Establecer zona horaria
				ln -s /usr/share/zoneinfo/Europe/Madrid /etc/localtime
			Añadir usuario
				useradd -m -s /bin/bash tony
				passwd tony
				Grupos para el usuario
					usermod -aG wheel,audio,video,lp,storage,users tony
			Permitir acciones de root al usuario
				visudo
				Descomentar la línea de %wheel ALL=(ALL:ALL) ALL
			Archivo	fstab
					blkid | grep vfat >> /etc/fstab
					nano /etc/fstab
						Para la partición boot usar el UUID, el fichero quedaría así
							UUID=xxxx-xxxx			/boot/efi		vfat		defaults	0		2
							/dev/vda2						none				swap		defaults	0		0
							/dev/mapper/disco		/						ext4		defaults	0		1
						Borrar o comentar la línea que se añadió con el blkid | grep vfat >> /etc/fstab
			Archivo /etc/default/grub
				blkid | grep LUKS >> /etc/default/grub
				nano /etc/default/grub
					Añadir la línea:
						GRUB_ENABLE_CRYPTODISK=y
					Modificar la línea GRUB_CMDLINE_LINUX_DEFAULT=" loglevel=4"
						En esa misma línea hay que añadir dentro de las comillas:
						"rd.luks.name=uuid-del-dispositivo-LUKS=disco root=/dev/mapper/disco loglevel=4"
					Borrar o comentar la línea que se añadió con el blkid | grep LUKS >> /etc/default/grub
			Crear archivo necesario para que funcione
				nano /etc/dracut.conf.d/hostonly.conf
					Dentro de ese archivo añadir lo siguiente:
						hostonly=yes
			Instalar grub
				grub-install /dev/vda1
			Forzar configurar el sistema
				xbps-reconfigure -fa
Salir de chroot
	exit
Desmontar el sistema de ficheros y reiniciar
	umount -R /mnt
	reboot

El sistema arranca y pide clave para descifrar el disco
	Después del menú de grub vuelve a pedir la contraseña
No hay conexión a internet
	Solución: Activar servicio dhcpcd
		ln -s /etc/sv/dhcpcd /var/service
Cifrar swap con clave aleatoria en cada arranque
	nano /etc/crypttab
		Copiar la línea de swap
		Modificar la parte que hace referencia a la partición /dev/hdX por /dev/vda2 , resto queda igual...
	nano /etc/fstab
		Modificar la línea de swap cambiando /dev/vda2 por /dev/mapper/swap
	Después de estos dos cambios, en el siguiente arranque, sale advertencia sobre cifrado - responder YES
Evitar que pida dos veces la contraseña en el arranque
	Esto se consigue creando una archivo que contenga una clave aleatoria en el disco y añadirla como clave válida para /dev/vda3
		mkdir /root/keys
		dd id=/dev/random of=/root/keys/archivo-clave.key bs=1 count=128
	Medidas de protección al directorio y al archivo
		chmod 700 /directorio-que-contiene-el-archivo-de-clave  (/root/keys)			
		chmod 000 /archivo-que-contiene-la-clave                (root/keys/archivo-clave.key)
	Añadir la clave al disco cifrado
		cryptsetup luksAddKey /dev/vda3 /dir/archivo-clave
			Te pide contraseña de disco (la que pide en el arranque)
	Se puede comprobar que ahora hay dos claves válidas con
		cryptsetup luksDump /dev/vda3
			Se debe ver:
				Key Slot 0: Enabled
					...
				Key Slot 1: Enabled
					...					
	Crear archivo para que funcione
		nano /etc/dracut.conf.d/10-disco-key.conf
		Añadir lo siguiente:
			install_items+=" /root/keys/archivo-clave.key /etc/crypttab "
	Añadir la clave a /etc/crypptab
		nano /etc/crypttab
			Añadir lo siguiente: (disco es el nombre que se está usando para el dispositivo mapeado)
				disco		/dev/vda3		/root/keys/archivo-clave.key	luks
	Actualizar el arranque
		Es necesario saber la versión de kernel
			uname -r
			En este momento la salida es 6.12.31.1
			Para el siguiente comando solo se utiliza el 6.12
				xbps-reconfigure -f linux6.12
	Reiniciar para comprobar
		reboot
	Si todo ha ido solo pide la contraseña del disco una vez.
	
	Entorno gráfico gnome con red,sonido,impresorasmkdi
		xbps-install -S xorg gnome pipewire wireplumber cups cups-filters
		Configurar pipewire y wireplumber
			mkdir -p /etc/pipewire/pipewire.conf.d
			ln -s /usr/share/examples/wireplumber/10-wireplumber.conf /etc/pipewire/pipewire.conf.d			
			ln -s /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d
			Archivo para autoarranque de pipewire
				cd /etc/xdg/autostart
				Crear archivo pipewire.desktop
					nano pipewire.desktop
						Añadir lo siguiente:
							[Desktop Entry]
							Type=Application
							Name=Pipewire
							Exec=pipewire
		Eliminar servicio dhcpcd
			rm /var/service/dhcpcd
		Activar servicios necesarios gnome:
			ln -s /etc/sv/dbus /var/service
			ln -s /etc/sv/NetworkManager /var/service
			ln -s /etc/sc/dbus /var/service
			ln -s /etc/sc/gdm /var/service
		A los pocos segundos arranca el entorno de escritorio
		Ya aparece el símbolo de red y volumen.. (NetworkManager y pipewire están funcionando)
			Configurar resolución de pantalla (1920x1080 16:9)
			Configurar teclado español en las opciones de Configuración
			Cerrar y abrir sesión para que se apliquen los cambios del teclado		
	Comprobación de pipewire/wireplumber en terminal
		pactl info
		wpctl status
	Repositorios privativos desde terminal
		sudo xbps-install -Su void-repo-nonfree void-repo-multilib void-repo-multilib-nonfree
		sudo xbps-install -Su
	Códecs audio/video desde terminal
		sudo xbps-install -Su ffmpeg
	Instalar aplicaciones desde terminal		
		sudo xbps-install -Su firefox libreoffice octoxbps
		....
		
		
		
			
	
	

	
		

				
		
	
