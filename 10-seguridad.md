# Seguridad

## Retirar servicios en ejecución no imprescindibles

### Identificar servicios

Con `systemctl` para consultar eventos, enlistando las unidades cargadas (servicios, temporizadores, montajes, etc), pero filtrando por servicios que estén en ejecución.

```bashrc
sysadmin@servidor:~$ systemctl list-units --type=service --state=running --no-pager

  UNIT                        LOAD   ACTIVE SUB     DESCRIPTION                           
  bluetooth.service           loaded active running Bluetooth service
  cron.service                loaded active running Regular background program processing…
  dbus.service                loaded active running D-Bus System Message Bus
  getty@tty1.service          loaded active running Getty on tty1
  nmbd.service                loaded active running Samba NMB Daemon
  smbd.service                loaded active running Samba SMB Daemon
  ssh.service                 loaded active running OpenBSD Secure Shell server
  systemd-journald.service    loaded active running Journal Service
  systemd-logind.service      loaded active running User Login Management
  systemd-timesyncd.service   loaded active running Network Time Synchronization
  systemd-udevd.service       loaded active running Rule-based Manager for Device Events …
  unattended-upgrades.service loaded active running Unattended Upgrades Shutdown
  user@1000.service           loaded active running User Manager for UID 1000
  winbind.service             loaded active running Samba Winbind Daemon
  wpa_supplicant.service      loaded active running WPA supplicant

Legend: LOAD   → Reflects whether the unit definition was properly loaded.
        ACTIVE → The high-level unit activation state, i.e. generalization of SUB.
        SUB    → The low-level unit activation state, values depend on unit type.

15 loaded units listed.
```

- Bluetooth es un servicio innecesario.
- nmbd si se consulta el manual `man nmbd` se identifica como un servidor que puede responder a solicitudes de servicio de nombres NetBIOS generadas por clientes tipo Windows 95/98/ME/NT/2000 o Windows XP.
- winbind (consultable con `man winbindd`) está orientado a dominios Active Directory.

Antes de tocar nada es buena idea mirar sockets abiertos desde dentro del servidor, con `ss` (`man ss`). Opciones `-t` para TCP, `-u` para UDP (TCP y UDP son los principales conjuntos de protocolos). Específicamente `-l` los que estén escuchando (listening), `-p` para ver qué procesos están usando,  y los puertos `-n`.

```bash
sysadmin@servidor:~$ sudo ss -tulpn

[sudo] contraseña para sysadmin: 
Netid    State     Recv-Q    Send-Q       Local Address:Port         Peer Address:Port    Process                                                                                   
udp      UNCONN    0         0            192.168.1.255:137               0.0.0.0:*        users:(("nmbd",pid=784,fd=15))                                                           
udp      UNCONN    0         0            192.168.1.250:137               0.0.0.0:*        users:(("nmbd",pid=784,fd=14))                                                           
udp      UNCONN    0         0                  0.0.0.0:137               0.0.0.0:*        users:(("nmbd",pid=784,fd=12))                                                           
udp      UNCONN    0         0            192.168.1.255:138               0.0.0.0:*        users:(("nmbd",pid=784,fd=17))                                                           
udp      UNCONN    0         0            192.168.1.250:138               0.0.0.0:*        users:(("nmbd",pid=784,fd=16))                                                           
udp      UNCONN    0         0                  0.0.0.0:138               0.0.0.0:*        users:(("nmbd",pid=784,fd=13))                                                           
tcp      LISTEN    0         50                 0.0.0.0:139               0.0.0.0:*        users:(("smbd",pid=912,fd=31))                                                           
tcp      LISTEN    0         128                0.0.0.0:22                0.0.0.0:*        users:(("sshd",pid=760,fd=6))                                                            
tcp      LISTEN    0         50                 0.0.0.0:445               0.0.0.0:*        users:(("smbd",pid=912,fd=30))                                                           
tcp      LISTEN    0         50                    [::]:139                  [::]:*        users:(("smbd",pid=912,fd=29))                                                           
tcp      LISTEN    0         128                   [::]:22                   [::]:*        users:(("sshd",pid=760,fd=7))                                                            
tcp      LISTEN    0         50                    [::]:445                  [::]:*        users:(("smbd",pid=912,fd=28))     
```

Ahora se consulta en el listado de puertos (se puede consultar el listado completo con `less /etc/services` o en https://www.iana.org/assignments/service-names-port-numbers):

| Puerto | Servicio |
|--------|----------|
| 22 | ssh |
| 137 | netbios-ns |
| 138 | netbios-dgm |
| 139 | netbios-ssn |
| 445 | microsoft-ds |

Se podrían retirar los servicios asociados a los puertos 137, 138 y 139.

### Retirar bluetooth

Bluetooth se controla en Linux con el paquete de protocolos BlueZ (https://bluez.github.io/).

```bash
sysadmin@servidor:~$ sudo apt purge bluez

[sudo] contraseña para sysadmin: 
Los paquetes indicados a continuación se instalaron de forma automática y ya no son necesarios.
  alsa-topology-conf  alsa-ucm-conf  libasound2-data  libasound2t64
Utilice «sudo apt autoremove» para eliminarlos.

REMOVING:
  bluetooth*  bluez*

Summary:
  Upgrading: 0, Installing: 0, Removing: 2, Not Upgrading: 1
  Freed space: 5.007 kB

Continue? [S/n] 
(Leyendo la base de datos ... 41331 ficheros o directorios instalados actualmente.)
Desinstalando bluetooth (5.82-1.1) ...
Desinstalando bluez (5.82-1.1) ...
Procesando disparadores para dbus (1.16.2-2) ...
Procesando disparadores para man-db (2.13.1-1) ...
(Leyendo la base de datos ... 41216 ficheros o directorios instalados actualmente.)
Purgando ficheros de configuración de bluez (5.82-1.1) ...
```

### Cerrar puertos 137, 138 y 139

Se cierran desde la configuración de Samba:

```bash
sysadmin@servidor:~$ sudo nano /etc/samba/smb.conf
```

En la sección `[global]`, debajo de donde se configuró antes el `veto files...`:

```bash
disable netbios = yes
smb ports = 445
```

Y se reinicia Samba:

```bash
sysadmin@servidor:~$ sudo systemctl restart smbd
```

Se apaga `nmbd`:

```bash
sysadmin@servidor:~$ sudo systemctl disable --now nmbd

Synchronizing state of nmbd.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install disable nmbd
Removed '/etc/systemd/system/multi-user.target.wants/nmbd.service'.
Removed '/etc/systemd/system/nmb.service'.
```

### Retirar `winbind`

```bash
sysadmin@servidor:~$ sudo apt purge winbind
```

### Comprobación

```bash
sysadmin@servidor:~$ systemctl list-units --type=service --state=running --no-pager

  UNIT                        LOAD   ACTIVE SUB     DESCRIPTION                                   
  cron.service                loaded active running Regular background program processing daemon
  dbus.service                loaded active running D-Bus System Message Bus
  getty@tty1.service          loaded active running Getty on tty1
  smbd.service                loaded active running Samba SMB Daemon
  ssh.service                 loaded active running OpenBSD Secure Shell server
  systemd-journald.service    loaded active running Journal Service
  systemd-logind.service      loaded active running User Login Management
  systemd-timesyncd.service   loaded active running Network Time Synchronization
  systemd-udevd.service       loaded active running Rule-based Manager for Device Events and Files
  unattended-upgrades.service loaded active running Unattended Upgrades Shutdown
  user@1000.service           loaded active running User Manager for UID 1000
  wpa_supplicant.service      loaded active running WPA supplicant

Legend: LOAD   → Reflects whether the unit definition was properly loaded.
        ACTIVE → The high-level unit activation state, i.e. generalization of SUB.
        SUB    → The low-level unit activation state, values depend on unit type.

12 loaded units listed.
```