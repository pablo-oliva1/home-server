# 4. Discos

Se conectará los discos secuencialmente para identificarlos con `lsblk`, y dado que el disco principal ocupa 2 TB y el destinado a backups 1 TB, se gestionará el principal de la siguiente forma:

|Uso|Tamaño|
|---|------|
|Espacio de usuario 1|450 GiB|
|Espacio de usuario 2|450 GiB|
|Espacio compartido|300 GiB|

Para esto se hará uso de LVM. Si en otra ocasión se dispone de otro disco de 2 TB para copias de seguridad se podrá aumentar el espacio destinado a los usuarios. Del espacio compartido no se hará compia de seguridad.

**Apartados:**
- Conectar e identificar discos.
- Borrado de las firmas previas de los discos.
- Particionar disco principal para LVM.
- Estructura LVM del disco principal.
- Formatear volúmenes del LVM.
- Preparar disco de backups.
- Automatización de los montajes.
- Aplicar y comprobar.

## Conectar e identificar discos

Referencia sin discos:
```bash
sysadmin@servidor:~$ lsblk -d -o NAME,SIZE,TRAN,MODEL

NAME          SIZE TRAN MODEL
mmcblk0      29,1G mmc  
mmcblk0boot0    4M mmc  
mmcblk0boot1    4M mmc  
```

Conectar el disco 1 de 2TB a puerto azul y comprobar:
```bash
sysadmin@servidor:~$ lsblk -d -o NAME,SIZE,TRAN,MODEL

NAME          SIZE TRAN MODEL
sda           1,8T usb  Expansion SW
mmcblk0      29,1G mmc  
mmcblk0boot0    4M mmc  
mmcblk0boot1    4M mmc  

sysadmin@servidor:~$ lsusb -t

/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/7p, 480M
    |__ Port 004: Dev 002, If 0, Class=Wireless, Driver=btusb, 12M
    |__ Port 004: Dev 002, If 1, Class=Wireless, Driver=btusb, 12M
    |__ Port 005: Dev 003, If 0, Class=Video, Driver=uvcvideo, 480M
    |__ Port 005: Dev 003, If 1, Class=Video, Driver=uvcvideo, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/6p, 5000M
    |__ Port 001: Dev 010, If 0, Class=Mass Storage, Driver=uas, 5000M
```
Conectar HGST de 1 TB al puerto restante y comprobar:
``` bash
sysadmin@servidor:~$ lsblk -d -o NAME,SIZE,TRAN,MODEL

NAME           SIZE TRAN MODEL
sda            1,8T usb  Expansion SW
sdb          931,5G usb  HGST HTS721010A9E630
mmcblk0       29,1G mmc  
mmcblk0boot0     4M mmc  
mmcblk0boot1     4M mmc 

sysadmin@servidor:~$ lsusb -t

/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/7p, 480M
    |__ Port 002: Dev 026, If 0, Class=Mass Storage, Driver=usb-storage, 480M
    |__ Port 004: Dev 002, If 0, Class=Wireless, Driver=btusb, 12M
    |__ Port 004: Dev 002, If 1, Class=Wireless, Driver=btusb, 12M
    |__ Port 005: Dev 003, If 0, Class=Video, Driver=uvcvideo, 480M
    |__ Port 005: Dev 003, If 1, Class=Video, Driver=uvcvideo, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/6p, 5000M
    |__ Port 001: Dev 010, If 0, Class=Mass Storage, Driver=uas, 5000M
```

## Borrado de las firmas previas de los discos

Se borra las firmas del sistema de archivos y de la tabla de particiones, para trabajar con los discos sin restos de uso anterior.

```bash
sysadmin@servidor:~$ sudo wipefs -a /dev/sda1 /dev/sda

/dev/sda1: se han borrado 8 bytes en el desplazamiento 0x00000003 (ntfs): 4e 54 46 53 20 20 20 20
wipefs: /dev/sda1: se hace caso omiso de la tabla de partición anidada "dos" sobre el dispositivo de disco no completo
wipefs: Utilice la opción --force para forzar el borrado.
/dev/sda: se han borrado 8 bytes en el desplazamiento 0x00000200 (gpt): 45 46 49 20 50 41 52 54
/dev/sda: se han borrado 8 bytes en el desplazamiento 0x1d1c1115c00 (gpt): 45 46 49 20 50 41 52 54
/dev/sda: se han borrado 2 bytes en el desplazamiento 0x000001fe (PMBR): 55 aa
/dev/sda: llamando a ioctl() para volver a leer la tabla de particiones: Conseguido
```

```bash
sysadmin@servidor:~$ sudo wipefs -a /dev/sdb1 /dev/sdb

/dev/sdb1: se han borrado 8 bytes en el desplazamiento 0x00000003 (ntfs): 4e 54 46 53 20 20 20 20
wipefs: /dev/sdb1: se hace caso omiso de la tabla de partición anidada "dos" sobre el dispositivo de disco no completo
wipefs: Utilice la opción --force para forzar el borrado.
/dev/sdb: se han borrado 2 bytes en el desplazamiento 0x000001fe (dos): 55 aa
/dev/sdb: llamando a ioctl() para volver a leer la tabla de particiones: Conseguido
```

```bash
sysadmin@servidor:~$ lsblk -f /dev/sda /dev/sdb

NAME FSTYPE FSVER LABEL UUID FSAVAIL FSUSE% MOUNTPOINTS
sda                                         
sdb      
```

## Particionar disco principal para LVM

Las particiones son compartimentaciones rígidas difíciles de modificar si cambian las necesidades. LVM supone una capa de gestión que flexibiliza la compartimentación con la siguiente estructura:

|Capa|Descripción|
|----|-----------|
|PV o Physical Volume|Es el dispositivo (disco o partición de disco) sobre el que se levanta LVM. En este caso es `/dev/sda1`.|
|VG o Volume Group|Grupo de espacios administrados por LVM; puede reunir dentro uno o varios PV.|
|LV o Logical Volume|Porción de espacio de VG que se puede usar como si fuera un dispositivo independiente.|
|Sistema de archivos|Se crea dentro del LV y tras eso puede montarse en una carpeta para guardar archivos.|

Utilidades que le podemos dar:
  - Cambiar tamaños sin tener que reorganizar todas las particiones.
  - Combinar discos bajo un mismo VG.
  - Separar datos, copias, etc en volúmenes distintos.

Instalación:
```bash
sysadmin@servidor:~$ sudo apt install -y lvm2
```

Se crea una tabla GPT con una sola partición que ocupe todo el disco:
```bash
sysadmin@servidor:~$ sudo parted /dev/sda mklabel gpt

Information: You may need to update /etc/fstab.
```

Alinea la partición con los bloques físicos del disco y crea una partición (`datos`) para todo el disco:
```bash
sysadmin@servidor:~$ sudo parted -a optimal /dev/sda mkpart datos 0% 100%

Information: You may need to update /etc/fstab.
```
Marca la partición 1 destinada a LVM:

```bash
sysadmin@servidor:~$ sudo parted /dev/sda set 1 lvm on                    

Information: You may need to update /etc/fstab.
```
Muestra el resultado:
```bash
sysadmin@servidor:~$ sudo parted /dev/sda print

Model: Seagate Expansion SW (scsi)
Disk /dev/sda: 2000GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name   Flags
 1      1049kB  2000GB  2000GB               datos  lvm
```

Conclusión:
- Tenemos una partición (`datos`) de 2000 GB con la marca `lvm`.
- Va de los 1049 kB a los 2000 GB.

## Estructura LVM del disco principal

Crear Physical Volume y comprobar:
```bash
sysadmin@servidor:~$ sudo pvcreate /dev/sda1

WARNING: dos signature detected on /dev/sda1 at offset 510. Wipe it? [y/n]: y
  Wiping dos signature on /dev/sda1.
  Physical volume "/dev/sda1" successfully created.

sysadmin@servidor:~$ sudo pvs

  PV         VG Fmt  Attr PSize  PFree 
  /dev/sda1     lvm2 ---  <1,82t <1,82t
```
- Creado un PV de 1,82 TB.


Crear Volume Group y comprobar:
```bash
sysadmin@servidor:~$ sudo vgcreate datos /dev/sda1

  Volume group "datos" successfully created

sysadmin@servidor:~$ sudo vgs

  VG    #PV #LV #SN Attr   VSize  VFree 
  datos   1   0   0 wz--n- <1,82t <1,82t
```
- Creado un VG llamado "*datos*", aún sin volúmenes lógicos.

Crear los distintos Logical Volume y comprobar:
```bash
sysadmin@servidor:~$ sudo lvcreate -L 450G -n user1 datos
  Logical volume "user1" created.

sysadmin@servidor:~$ sudo lvcreate -L 450G -n user2 datos
  Logical volume "user2" created.

sysadmin@servidor:~$ sudo lvcreate -L 300G -n comun datos
  Logical volume "comun" created.

sysadmin@servidor:~$ sudo lvs

  LV    VG    Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  comun datos -wi-a----- 300,00g                                                    
  user1 datos -wi-a----- 450,00g                                                    
  user2 datos -wi-a----- 450,00g                                                    

sysadmin@servidor:~$ sudo vgs
  VG    #PV #LV #SN Attr   VSize  VFree  
  datos   1   3   0 wz--n- <1,82t 663,01g
```
- Se han creado dentro del único Physical Volume 3 LV (y quedan 663 GiB libres).

## Formatear volúmenes del LVM
Formateo del primer LV:
```bash
sysadmin@servidor:~$ sudo mkfs.ext4 -m 0 -L user1 /dev/datos/user1

mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 117964800 4k blocks and 29491200 inodes
Filesystem UUID: XXXX-XXXX-XXXX-XXXX
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
        102400000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done     
```
Formateo del segundo LV:
```bash
sysadmin@servidor:~$ sudo mkfs.ext4 -m 0 -L user2 /dev/datos/user2

mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 117964800 4k blocks and 29491200 inodes
Filesystem UUID: XXXX-XXXX-XXXX-XXXX
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
        102400000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done     
```
Formateo del tercer LV:
```bash
sysadmin@servidor:~$ sudo mkfs.ext4 -m 0 -L comun /dev/datos/comun

mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 78643200 4k blocks and 19660800 inodes
Filesystem UUID: XXXX-XXXX-XXXX-XXXX
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done  
```

## Preparar disco de backups
Crear la tabla de particiones:
```bash
sysadmin@servidor:~$ sudo parted /dev/sdb mklabel gpt
Information: You may need to update /etc/fstab.
```
Aplicar partición:
```bash
sysadmin@servidor:~$ sudo parted -a optimal /dev/sdb mkpart copias 0% 100%
Information: You may need to update /etc/fstab.
```
Ver tabla de particiones:
```bash
sysadmin@servidor:~$ sudo parted /dev/sdb print

Model: HGST HTS 721010A9E630 (scsi)
Disk /dev/sdb: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name    Flags
 1      1049kB  1000GB  1000GB               copias
```
Aplicar formato a la partición:
```bash
sysadmin@servidor:~$ sudo mkfs.ext4 -m 0 -L copias /dev/sdb1

mke2fs 1.47.2 (1-Jan-2025)
Found a dos partition table in /dev/sdb1
Proceed anyway? (y,N) y
Creating filesystem with 244190208 4k blocks and 61054976 inodes
Filesystem UUID: XXXX-XXXX-XXXX-XXXX
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
        102400000, 214990848

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done    
```

## Automatización de los montajes
Crear puntos de montaje en `/srv`:
```bash
sysadmin@servidor:~$ sudo mkdir /srv/user1 /srv/user2 /srv/comun /srv/copias
```

Hacer una copia de la tabla de sistemas de archivos:
```bash
sysadmin@servidor:~$ sudo cp /etc/fstab /etc/fstab.bak
```

AHora se consulta la información de los sistemas de archivos (`-f` o *filesystem*) que tiene la tabla de sistemas de archivos tanto para `sda` como para `sdb` para sacar los identificadores (UUID) de cada partición:
```bash
sysadmin@servidor:~$ lsblk -f /dev/sda /dev/sdb

NAME        FSTYPE FSVER LABEL  UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                   
└─sda1      LVM2_m LVM2         XXXX-XXXX-XXXX-XXXX                
  ├─datos-user1
  │         ext4   1.0   user1  XXXX-XXXX-XXXX-XXXX                  
  ├─datos-user2
  │         ext4   1.0   user2  XXXX-XXXX-XXXX-XXXX                  
  └─datos-comun
            ext4   1.0   comun  XXXX-XXXX-XXXX-XXXX                  
sdb                                                                                   
└─sdb1      ext4   1.0   copias XXXX-XXXX-XXXX-XXXX
```

Con los UUID de cada volumen se van a agregar unas líneas a `fstab` con el sistema que se quiere montar (la UUID), dónde montarlo (los directorios que hemos creado como puntos de montaje), el tipo de sistema de archivos (que es ext4), las opciones de montaje (*defaults* con las opciones por defecto y *nofail* para que el sistema arranque aun indisponibilidad del disco).

En cualquier caso puede consultarse `man fstab` para ver más sobre este tipo de estructura y opciones.
```bash
sysadmin@servidor:~$ sudo nano /etc/fstab

# Discos del servidor
UUID=XXXX-XXXX-XXXX-XXXX  /srv/user1   ext4  defaults,nofail  0  2
UUID=XXXX-XXXX-XXXX-XXXX  /srv/user2   ext4  defaults,nofail  0  2
UUID=XXXX-XXXX-XXXX-XXXX  /srv/comun   ext4  defaults,nofail  0  2
UUID=XXXX-XXXX-XXXX-XXXX  /srv/copias  ext4  defaults,nofail  0  2
```

## Aplicar y comprobar
Se verifica con `findmnt`:
```bash
sysadmin@servidor:~$ sudo findmnt --verify

   [W] fstab ha sido modificado, pero systemd todavía utiliza los nombres de la
       versión antigua; utilice 'systemctl daemon-reload' para recargar

0 errores de sintaxis, 0 errores, 1 aviso
```
- Vemos que notifica de que se ha modificado fstab pero que se mantiene la configuración previa.

Se recarga y se vuelve a montar todo:
```bash
sysadmin@servidor:~$ sudo systemctl daemon-reload

sysadmin@servidor:~$ sudo mount -a
```
Se comprueba:
```bash
sysadmin@servidor:~$ df -h /srv/user1 /srv/user2 /srv/comun /srv/copias

S.ficheros              Tamaño Usados  Disp Uso% Montado en
/dev/mapper/datos-user1   442G   2,1M  442G   1% /srv/user1
/dev/mapper/datos-user2   442G   2,1M  442G   1% /srv/user2
/dev/mapper/datos-comun   295G   2,1M  295G   1% /srv/comun
/dev/sdb1                 916G   2,1M  916G   1% /srv/copias
```
Se reinicia:
```bash
sysadmin@servidor:~$ sudo reboot
```

Y se comprueba persistencia:
```bash
sysadmin@servidor:~$ df -h /srv/user1 /srv/user2 /srv/comun /srv/copias
S.ficheros              Tamaño Usados  Disp Uso% Montado en
/dev/mapper/datos-user1   442G   2,1M  442G   1% /srv/user1
/dev/mapper/datos-user2   442G   2,1M  442G   1% /srv/user2
/dev/mapper/datos-comun   295G   2,1M  295G   1% /srv/comun
/dev/sdb1                 916G   2,1M  916G   1% /srv/copias
```