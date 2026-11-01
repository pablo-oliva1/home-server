# 1. Instalación

Descripción del equipo, instalación de Debian y comprobaciones básicas del sistema.

**Apartados**

- Especificaciones del equipo.
- Instalación de sistema operativo.
- Primer arranque y comprobación.


## Especificaciones del equipo

Fabricante y modelo (salida recortada):
```bash
~$ sudo dmidecode -t system

System Information
        Manufacturer: LENOVO
        Product Name: 80LY
        Version: Lenovo FLEX 3-1130
```

Procesador (salida recortada):
```bash
~$ lscpu

ID de fabricante:                        GenuineIntel
  Nombre del modelo:                     Intel(R) Celeron(R) CPU  N3060  @ 1.60GHz
    Familia de CPU:                      6
    Modelo:                              76
    Hilo(s) de procesamiento por núcleo: 1
    Núcleo(s) por «socket»:              2
```

RAM:
```bash
~$ free -h

               total       usado       libre  compartido   búf/caché  disponible
Mem:           1,8Gi       323Mi       1,5Gi       4,3Mi       161Mi       1,5Gi
Inter:         1,5Gi          0B       1,5Gi
```

Puertos y velocidad:
```bash
~$ lsusb -t

/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/7p, 480M
    |__ Port 004: Dev 002, If 0, Class=Wireless, Driver=btusb, 12M
    |__ Port 004: Dev 002, If 1, Class=Wireless, Driver=btusb, 12M
    |__ Port 005: Dev 003, If 0, Class=Video, Driver=uvcvideo, 480M
    |__ Port 005: Dev 003, If 1, Class=Video, Driver=uvcvideo, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/6p, 5000M
```

**Tabla resumen**

| Componente | Detalle |
| --- | --- |
| Modelo | Yoga 300-11IBR |
| Año de fabricación | 2016 |
| Procesador | Intel Celeron N3060 @ 1.60 GHz, 2 núcleos, 64 bits |
| RAM | 2 GB (1,8 GiB utilizables) |
| Almacenamiento interno | eMMC de 32 GB (29,1 GiB) |
| Wi-Fi | Intel Wireless 3160 |
| USB | 1 puerto USB 3.0 (azul) y 2 USB 2.0 |

## Instalación de sistema operativo

Se opta por Debian porque es estable, es ligero y por la experiencia previa con este sistema. 

Identificar pendrive:
```bash
~$ lsblk -d -o NAME,SIZE,TYPE,TRAN,MODEL

NAME      SIZE TYPE TRAN MODEL
sda     115,5G disk usb  DataTraveler 3.0
```

Se descarga Debian 13.7.0 (iso netinst) desde debian.org y se comprueba la integridad de la descarga:
```bash
~$ cd Descargas

~/Descargas$ sha256sum -c SHA256SUMS --ignore-missing
debian-13.7.0-amd64-netinst.iso: La suma coincide
```

Se graba la iso en bloques de 4 MiB:
```bash
~/Descargas$ ls -l

-rw-rw-r-- 1 792723456 debian-13.7.0-amd64-netinst.iso
-rw-rw-r-- 1 302 SHA256SUMS

~/Descargas$ sudo dd bs=4M of=/dev/sda if=debian-13.7.0-amd64-netinst.iso

189+0 records in
189+0 records out
792723456 bytes (793 MB, 756 MiB) copied, 118,177 s, 6,7 MB/s

~/Descargas$ sync
```

Durante la instalación del SO se opta por un esquema de particionado simple (*Todos los ficheros en una partición*) por los siguientes motivos:
- Se descarta particionado /home porque no se usará.
- Lo mismo para /home, /var y /tmp.
- También la que recomienda para servidores (/srv) porque los datos irán en discos externos y no tiene sentido compartimentar en este caso.

En la fase de *Selección de programas* se dejan sin marcar los entornos gráficos (consumen RAM), y se deja marcado:
- SSH server.
- Utilidades estándar del sistema.

## Primer arranque y comprobación

Tras el primer arranque del sistema operativo se hace login con el primer usuario y se realizan una serie de comprobaciones:

Versión del sistema:
```bash
sysadmin@servidor:~$ cat /etc/debian_version

13.7
```

Versión del kernel:
```bash
sysadmin@servidor:~$ uname -r

6.12.113+deb13-amd64
```

Comprobar particionado:
```bash
sysadmin@servidor:~$ lsblk

NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0      179:0    0 29,1G  0 disk
├─mmcblk0p1  179:1    0  976M  0 part /boot/efi
├─mmcblk0p2  179:2    0 26,7G  0 part /
└─mmcblk0p3  179:3    0  1,5G  0 part [SWAP]
mmcblk0boot0 179:256  0    4M  1 disk
mmcblk0boot1 179:512  0    4M  1 disk
```

Comprobar funcionamiento de la red (salida recortada):
```bash
sysadmin@servidor:~$ ip -br a

lo               UNKNOWN    127.0.0.1/8 ::1/128
enp3s0           DOWN
wlp2s0           UP         192.168.1.135/24
```

Comprobar estado de ssh (salida recortada):
```bash
sysadmin@servidor:~$ systemctl status ssh

● ssh.service - OpenBSD Secure Shell server
    Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running)
```