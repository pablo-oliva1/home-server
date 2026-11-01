# 6. Samba y SMB
En esta parte del proceso se va a instalar *Samba*, un software para implementar el protocolo SMB en linux, y se configuran las conexiones usuario-servidor tanto de user1 (sistema Linux) como en user2 (en Windows).

**Apartados:**
- Instalación.
- Configurar carpetas compartidas.
- Crear y aplicar contraseñas.
- Probar Samba desde el propio servidor.
- Configurar acceso a servidor y montaje de carpetas en user1 (Linux).
- Configurar acceso a servidor y montaje de carpetas en user2 (Windows).

## Instalación

Instalar Samba:
```bash
sysadmin@servidor:~$ sudo apt install -y samba
Installing:                             
  samba
```

Comprobar estado:
```bash
sysadmin@servidor:~$ systemctl status smbd

● smbd.service - Samba SMB Daemon
     Loaded: loaded (/usr/lib/systemd/system/smbd.service; enabled; preset: enabled)
     Active: active (running)
```

## Configurar carpetas compartidas

En `/etc/samba/smb.conf` se deben definir las carpetas que el servidor ofrece en la red y las reglas de acceso. 

Primero se hace una copia de seguridad del archivo de configuración de Samba:

```bash
sysadmin@servidor:~$ sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

Luego se neutralizan los bloques de `[homes]`, `[printers]` y `[print$]` agregando `#` antes de cada línea.

Ejemplo:
```bash
# [homes]
#   comment = Home Directories
#   browseable = no
# By default, the home directories are exported read-only. Change the
# next parameter to 'no' if you want to be able to write to them.
#   read only = yes
```

En el archivo hay una sección titulada `[global]`, con varios apartados y al final del apartado *Authentication* se encuentra el parámetro `map to guest` cuya activación permite que todo login cuyo usuario no exista pasa a ser una conexión como invitado. Es importante marcar este parámetro como *never*:

```bash
# This option controls how unsuccessful authentication attempts are mapped
# to anonymous connections
   map to guest = never
```
Luego al final del documento se pueden agregar los parámetros relacionados con las carpetas y ciertas reglas de acceso como:

- El nombre del recurso en la red.
- Ruta de la carpeta que se comparte.
- Quién puede conectarse.
- Si se puede escribir o será de solo lectura.
- Si se ve solamente los recursos a los que se tiene acceso.
- Los permisos de los archivos nuevos.
- Los permisos de las nuevas carpetas.
- Grupo de pertenencia si hace falta.

```bash
[user1]
        comment = Espacio de user1
        path = /srv/user1
        valid users = user1
        read only = no
        access based share enum = yes
        create mask = 0600
        directory mask = 0700

[user2]
        comment = Espacio de user2
        path = /srv/user2
        valid users = user2
        read only = no
        access based share enum = yes
        create mask = 0600
        directory mask = 0700

[comun]
        comment = Espacio personal común
        path = /srv/comun
        valid users = @compartido
        read only = no
        access based share enum = yes
        force group = compartido
        create mask = 0660
        directory mask = 2770
```

Después de guardar el archivo se puede realizar una comprobación que devuelve el texto:

```bash
sysadmin@servidor:~$ testparm -s
```

Luego se reinicia Samba para aplicar los cambios:

```bash
sysadmin@servidor:~$ sudo systemctl restart smbd
```

## Crear y aplicar contraseñas

Samba gestiona su propia base de datos de usuarios y contraseñas, por lo que se deben ir introduciendo en ella tanto los usuarios ya en en SO, y agregar las contraseñas.

Una herramienta de la suite de Samba para esto es `smbpasswd`, se puede consultar su manual con `man smbpasswd`, donde encontramos como opciones del comando:
- `-x` elimina un usuario.
- `-a` añade un usuario a Samba.
- `-d` (disable) desactiva una cuenta.
- `-e` (enabled) reactiva una cuenta.

Por otro lado base de datos se administra con `pdbedit` y consultando `man pdbedit` se identifican opciones como:
- `-L` lista todos los usuarios.
- `-u` usuario sobre el que se actúa.
- `-x` elimina un usuario.

Por lo tanto:
```bash
sysadmin@servidor:~$ sudo smbpasswd -a user1

New SMB password:
Retype new SMB password:
Added user user1.

sysadmin@servidor:~$ sudo smbpasswd -a user2

New SMB password:
Retype new SMB password:
Added user user2.
```

Se comprueba:
```bash
sysadmin@servidor:~$ sudo pdbedit -L

user1:1001:
user2:1002:
```
- Ambos usuarios están inscritos en la base de datos.

## Probar Samba desde el propio servidor

Una serie de pruebas desde el mismo servidor usando el cliente de Samba.

Instalación
```bash
sysadmin@servidor:~$ sudo apt install -y smbclient
```
Ver la lista de recursos que se le ofrece a un usuario concreto (`-U`):

```bash
sysadmin@servidor:~$ smbclient -L //localhost -U user1

Password for [WORKGROUP\user1]:

        Sharename       Type      Comment
        ---------       ----      -------
        user1           Disk      Espacio de user1
        comun           Disk      Espacio personal común
        IPC$            IPC       IPC Service (Samba 4.22.11-Debian-4.22.11+dfsg-0+deb13u1)
SMB1 disabled -- no workgroup available
```
- `user1` tiene acceso a sus recursos pero no a los de `user2`.

user1 accede a carpeta de user1:

```bash
sysadmin@servidor:~$ smbclient //localhost/user1 -U user1 -c 'ls'

Password for [WORKGROUP\user1]:
  .                                   D        0  Thu Oct 29 16:32:51 2026
  ..                                  D        0  Thu Oct 29 16:32:51 2026
  lost+found                          D        0  Thu Oct 29 13:12:24 2026

                463331120 blocks of size 1024. 463312660 blocks available
```
user1 accede a carpeta común:
```bash
sysadmin@servidor:~$ smbclient //localhost/comun -U user1 -c 'ls'

Password for [WORKGROUP\user1]:
  .                                   D        0  Thu Oct 29 16:32:51 2026
  ..                                  D        0  Thu Oct 29 16:32:51 2026
  lost+found                          D        0  Thu Oct 29 13:13:17 2026

                308521792 blocks of size 1024. 308503332 blocks available
```
user1 intenta acceder a carpeta de user2:
```bash
sysadmin@servidor:~$ smbclient //localhost/user2 -U user1 -c 'ls'

Password for [WORKGROUP\user1]:
tree connect failed: NT_STATUS_ACCESS_DENIED
```
user1 crea carpeta y archivo en área común:
```bash
sysadmin@servidor:~$ smbclient //localhost/comun -U user1 -c 'put /etc/hostname prueba.txt; mkdir carpeta_prueba'

Password for [WORKGROUP\user1]:
putting file /etc/hostname as \prueba.txt (1,3 kb/s) (average 1,3 kb/s)

sysadmin@servidor:~$ sudo ls -l /srv/comun

[sudo] contraseña para sysadmin: 
total 24
drwxrws--- 2 user1 compartido  4096 oct 30 10:36 carpeta_prueba
drwx------ 2 root  root       16384 oct 29 13:13 lost+found
-rw-rw---- 1 user1 compartido     9 oct 30 10:36 prueba.txt
```

## Configurar acceso a servidor y montaje de carpetas en user1 (Linux)

### Instalación smbclient y cifs-utils

En este paso se va a establecer el acceso desde el PC de user1 al servidor.

> **NOTA:** existen protocolos para enviar archivos por una red, uno es *FTP* (File Transfer Protocol), que también funciona con la arquitectura cliente-servidor pero se orienta a carga y descarga completa de archivos. El protocolo *SMB* (Server Message Block) funciona mediante el acceso remoto a los datos que hay en el servidor, trabajando en él.

Se instala de forma conjunta `smbclient` (programa cliente de SMB que va a ejecutar una conexión puntual a una determinada carpeta y luego va a desconectarse al acabar), y `cifs-utils` (permite a un sistema basado en Linux montar una carpeta SMB dentro del sistema de archivos):

```bash
user1@pc~$ sudo apt install smbclient cifs-utils

[sudo] contraseña para user1: 
Installing:                             
  cifs-utils  smbclient

Paquetes sugeridos:
  winbind  heimdal-clients

Summary:
  Upgrading: 0, Installing: 2, Removing: 0, Not Upgrading: 0
  Download size: 606 kB
  Space needed: 2.591 kB / 741 GB available
```
Luego se comprueba la conexión con el servidor y los recursos a los que se puede acceder:

```bash
user1@pc~$ smbclient -L //192.168.1.250 -U user1

Password for [WORKGROUP\user1]:

        Sharename       Type      Comment
        ---------       ----      -------
        user1           Disk      Espacio de user1
        comun           Disk      Espacio personal común
        IPC$            IPC       IPC Service (Samba 4.22.11-Debian-4.22.11+dfsg-0+deb13u1)
SMB1 disabled -- no workgroup available
```

### Acceso persistente

Ahora se va a configurar un acceso persistente desde el PC. Primero se crea los puntos de montaje tanto para el espacio personal como para el espacio comun:
```bash
user1@pc~$ mkdir -p ~/Documentos/Servidor/Personal ~/Documentos/Servidor/Comun
```

Se guarda las credenciales de acceso al servidor en un archivo solamente accesible para el root:
```bash
user1@pc~$ sudo nano /root/.smb-user1

username=user1
password=xxxxxxxxxxxxxxxx
```
Se aplican credenciales de protección y se comprueba:
```bash
user1@pc~$ sudo chmod 600 /root/.smb-user1

user1@pc~$ sudo ls -l /root/.smb-user1
-rw------- 1 root root 40 oct 30 11:31 /root/.smb-user1
```

Se busca el número de usuario (UID) y de grupo (GID) en `id`:
```bash
user1@pc~$ id

uid=1000(user1) gid=1000(user1) grupos=1000(user1)
```
Hay que diferenciar dos cosas: una cosa es que el PC pueda acceder al servidor (lo cual está dictaminado por Samba en el servidor), y otra que el PC permita a un usuario en concreto acceder a los archivos del servidor. La configuración por defecto permitiría esto, pero además que cualquier otro usuario del mismo PC accediera a esos archivos (`rwxr-xr-x`).

Se va a agregar en `fstab` el montaje de las carpetas con permisos concretos. ANteshacemos se hace una copia de seguridad y luego se abre:
```bash
user1@pc~$ sudo cp /etc/fstab /etc/fstab.bak

user1@pc~$ sudo nano /etc/fstab
```
Y se edita:
- `fstab` es el archivo donde se informa de los sistemas que se montan, se edita para autoamtizar el montaje de las carpetas del servidor en el PC (y no tener que montarlas cada vez que vayan a usarse).
- Primero se establece la ruta de origen (192.168... etc).
- Se indica el tipo de montaje (cifs).
- Indica dónde están las credenciales de acceso al servidor.
- Se indica que los archivos van a pertenecer a un usuario y grupo (UID y GID).
- Se define los permisos de los archivos (file_mode, etc...).
- Establece uso de caracteres utf8.
- Importante, supedita el montaje a que haya red (netdev).
- Para el caso de que el servidor esté apagado que esto no interrumpa el proceso general del sistema (nofail).
- Se monta la carpeta cuando se accede a ella.

```bash
//192.168.1.250/user1  /home/user1/Documentos/Servidor/Personal  cifs  credentials=/root/.smb-user1,uid=1000,gid=1000,file_mode=0600,dir_mode=0700,iocharset=utf8,_netdev,nofail,x-systemd.automount  0  0

//192.168.1.250/comun  /home/user1/Documentos/Servidor/Comun     cifs  credentials=/root/.smb-user1,uid=1000,gid=1000,file_mode=0600,dir_mode=0700,iocharset=utf8,_netdev,nofail,x-systemd.automount  0  0
```

Hecho esto se recarga `fstab` y se monta:
```bash
user1@pc~$ sudo systemctl daemon-reload

user1@pc~$ sudo mount -a
```

Se comprueba acceso a carpetas y escritura desde el PC de user1:
```bash
user1@pc~$ ls -la ~/Documentos/Servidor/Personal ~/Documentos/Servidor/Comun

/home/user1/Documentos/Servidor/Comun:
total 4
drwx------ 2 user1 user1    0 oct 30 10:44 .
drwxrwxr-x 4 user1 user1 4096 oct 30 11:28 ..
drwx------ 2 user1 user1    0 oct 29 13:13 lost+found

/home/user1/Documentos/Servidor/Personal:
total 4
drwx------ 2 user1 user1    0 oct 29 16:32 .
drwxrwxr-x 4 user1 user1 4096 oct 30 11:28 ..
drwx------ 2 user1 user1    0 oct 29 13:12 lost+found

user1@pc~$ touch ~/Documentos/Servidor/Personal/desde_pc.txt

user1@pc~$ touch ~/Documentos/Servidor/Comun/desde_pc.txt

user1@pc~$ ls -l ~/Documentos/Servidor/Personal ~/Documentos/Servidor/Comun

/home/user1/Documentos/Servidor/Comun:
total 0
-rw------- 1 user1 user1 0 oct 30 12:26 desde_pc.txt
drwx------ 2 user1 user1 0 oct 29 13:13 lost+found

/home/user1/Documentos/Servidor/Personal:
total 0
-rw------- 1 user1 user1 0 oct 30 12:25 desde_pc.txt
drwx------ 2 user1 user1 0 oct 29 13:12 lost+found
```

Y ahora se comprueba desde el servidor:
```bash
sysadmin@servidor:~$ sudo ls -l /srv/user1 /srv/comun

[sudo] contraseña para sysadmin: 
/srv/comun:
total 16
-rw-rw---- 1 user1 compartido     0 oct 30 12:26 desde_pc.txt
drwx------ 2 root  root       16384 oct 29 13:13 lost+found

/srv/user1:
total 16
-rw------- 1 user1 user1     0 oct 30 12:25 desde_pc.txt
drwx------ 2 root  root  16384 oct 29 13:12 lost+found
```
## Configurar acceso a servidor y montaje de carpetas en user2 (Windows)

### Acceso al servidor

EN este  punto del proceso se va a crear el acceso desde un PC con Windows 11 26H2.

En PowerlShell se abre la herramienta de administración de credenciales para establecer la conexión de user2 con el servidor:
```powershell
PS C:\Users\user2> cmdkey /add:192.168.1.250 /user:user2 /pass 

Escriba la contraseña de "user2" para conectar a "192.168.1.250": 

CMDKEY: credencial agregada correctamente.
```

Ahora se conecta la primera unidad. Se notifica recurso de red (net use), la letra de la unidad (Z), la dirección del recurso y se establece persistencia:
```powershell
PS C:\Users\user2> net use Z: \\192.168.1.250\user2 /persistent:yes 

Se ha completado el comando correctamente. 
```
Se conecta la segunda unidad en Y:
```powershell
PS C:\Users\user2> net use Y: \\192.168.1.250\comun /persistent:yes 

Se ha completado el comando correctamente.
```
Se comprueba estado:
```powershell
PS C:\Users\user2> net use 

Se registrarán las nuevas conexiones. 

Estado       Local     Remoto                    Red 

------------------------------------------------------------------------------- 

Conectado    Y:        \\192.168.1.250\comun     Microsoft Windows Network 

Conectado    Z:        \\192.168.1.250\user2     Microsoft Windows Network 

Se ha completado el comando correctamente.
```

En este punto las carpetas son visibles desde el Explorador de archivos de Windows.

### Comprobaciones

Comprobaciones de escritura de archivos y acceso a carpeta de user1:
```powershell
PS C:\Users\user2> New-Item Z:\desde_windows.txt 

    Directorio: Z:\ 
Mode                 LastWriteTime         Length Name 
----                 -------------         ------ ---- 
-a----        30/10/2026     13:14              0 desde_windows.txt 


PS C:\Users\user2> New-Item Y:\desde_windows.txt

    Directorio: Y:\ 
Mode                 LastWriteTime         Length Name 
----                 -------------         ------ ---- 
-a----        30/10/2026     13:14              0 desde_windows.txt 
 

PS C:\Users\user2> dir \\192.168.1.250\user1 

dir : Acceso denegado 
En línea: 1 Carácter: 1 
+ dir \\192.168.1.250\user1 
+ ~~~~~~~~~~~~~~~~~~~~~~~~~ 
    + CategoryInfo          : PermissionDenied: (\\192.168.1.250\user1:String) [Get-ChildItem], UnauthorizedAccessExce 
   ption 
    + FullyQualifiedErrorId : ItemExistsUnauthorizedAccessError,Microsoft.PowerShell.Commands.GetChildItemCommand 
dir : No se encuentra la ruta de acceso '\\192.168.1.250\user1' porque no existe. 
En línea: 1 Carácter: 1 
+ dir \\192.168.1.250\user1 
+ ~~~~~~~~~~~~~~~~~~~~~~~~~ 
    + CategoryInfo          : ObjectNotFound: (\\192.168.1.250\user1:String) [Get-ChildItem], ItemNotFoundException 
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.GetChildItemCommand.
```

Se comprueba en servidor:
```bash
sysadmin@servidor:~$ sudo ls -l /srv/user2 /srv/comun
[sudo] contraseña para sysadmin: 

/srv/comun:
total 16
-rw-rw---- 1 user2 compartido     0 oct 30 13:14 desde_windows.txt
drwx------ 2 root  root       16384 oct 29 13:13 lost+found

/srv/user2:
total 16
-rw------- 1 user2 user2     0 oct 30 13:14 desde_windows.txt
drwx------ 2 root  root  16384 oct 29 13:12 lost+found
```
Para comprobar que puede editarse un mismo archivo colaborativo, se escribe desde el PC de user1:

```bash
user1@pc~$ echo "Editado por user1 desde Linux" >> ~/Documentos/Servidor/Comun/desde_windows.txt

user1@pc~$ cat ~/Documentos/Servidor/Comun/desde_windows.txt
Editado por user1 desde Linux
```

Y luego se comprueba desde el PC de user2:

```powershell
PS C:\Users\user2> cat Y:\desde_windows.txt 

Editado por user1 desde Linux 
```