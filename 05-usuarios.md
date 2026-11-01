# 5. Usuarios
En esta parte del proceso se va a preparar un plan de gestión de usuarios, se aplicarán una serie de permisos y se les asignarán sus carpetas.

La idea es que se establezca esta esta configuración:

|**Carpeta**|**Propietario**|**Permisos**|**Quién entra**|
|-----------|---------------|------------|---------------|
|`/srv/user1`|user1|Solo el propietario|user1|
|`/srv/user2`|user2|Solo el propietario|user2|
`/srv/comun`|grupo compartido|Cualquiera del grupo puede leer y escribir|user1 y user2|
`/srv/copias`|root|Solo root|Nadie salvo el administrador|

**Apartados:**

- Crear grupo y usuarios.
- Aplicar permisos.
- Pruebas de permisos.

## Crear grupo y usuarios
Se crea un grupo para espacio común:
```bash
sysadmin@servidor:~$ sudo groupadd compartido
```

En este apartado se va a buscar la compartimentación de espacio (principio de mínimo privilegio). Por defecto al crear usuarios se les permite acceder a una terminal tras el login, y se les proporciona un espacio (`/home`). Es importante para mantener la compartimentación modificar el comando de creación de usuarios para alterar las opciones por defecto.

En `man useradd` se detallan bastantes opciones de este comando interesantes:
- `-M` no crea la carpeta personal en `/home`.
- `-s` altera la ruta de establecimiento de la shell que permite el inicio de sesión (puede usarse para redirigirla a `nologin`).
- `-G` añade el usuario al grupo `compartido`.

```bash
sysadmin@servidor:~$ sudo useradd -M -s /usr/sbin/nologin -G compartido user1

sysadmin@servidor:~$ sudo useradd -M -s /usr/sbin/nologin -G compartido user2
```
Comprobaciones:
```bash
sysadmin@servidor:~$ id user1
uid=1001(user1) gid=1002(user1) grupos=1002(user1),1001(compartido)

sysadmin@servidor:~$ id user2
uid=1002(user2) gid=1003(user2) grupos=1003(user2),1001(compartido)

sysadmin@servidor:~$ getent group compartido
compartido:x:1001:user1,user2
```

## Aplicar permisos
En cuanto a los permisos basta los permisos POSIX1, para que cada usuario tenga solamente acceso a aquello que se le designa como propietario, mientras que la carpeta compartida queda accesible para los miembros del grupo `compartido`:

```bash
sysadmin@servidor:~$ sudo chown user1:user1 /srv/user1
sysadmin@servidor:~$ sudo chmod 700 /srv/user1

sysadmin@servidor:~$ sudo chown user2:user2 /srv/user2
sysadmin@servidor:~$ sudo chmod 700 /srv/user2

sysadmin@servidor:~$ sudo chown root:compartido /srv/comun
sysadmin@servidor:~$ sudo chmod 2770 /srv/comun

sysadmin@servidor:~$ sudo chmod 700 /srv/copias

sysadmin@servidor:~$ ls -ld /srv/user1 /srv/user2 /srv/comun /srv/copias

sysadmin@servidor:~$ ls -ld /srv/user1 /srv/user2 /srv/comun /srv/copias

drwxrws--- 3 root  compartido 4096 oct 29 16:32 /srv/comun
drwx------ 3 root  root       4096 oct 29 13:38 /srv/copias
drwx------ 3 user1 user1      4096 oct 29 16:32 /srv/user1
drwx------ 3 user2 user2      4096 oct 29 16:32 /srv/user2
```

## Pruebas de permisos
Comprobar que user1 tiene acceso a su espacio:
```bash
sysadmin@servidor:~$ sudo -u user1 touch /srv/user1/prueba.txt

sysadmin@servidor:~$ sudo -u user1 ls /srv/user1
lost+found  prueba.txt

```

Comprobar que user1 no puede entrar en el espacio de user2:
```bash
sysadmin@servidor:~$ sudo -u user1 touch /srv/user2/prueba.txt
touch: no se puede efectuar `touch' sobre '/srv/user2/prueba.txt': Permiso denegado

```

Comprobar que user1 no puede ver lo que hay en el disco de  `copias`:
```bash
sysadmin@servidor:~$ sudo -u user1 ls /srv/copias
ls: no se puede abrir el directorio '/srv/copias': Permiso denegado
```

Comprobar que user1 puede crear en la carpeta compartida:
```bash
sysadmin@servidor:~$ sudo -u user1 touch /srv/comun/prueba2.txt

sysadmin@servidor:~$ sudo -u user1 ls /srv/comun
lost+found  prueba2.txt

```

Comprobar que user2 puede también crear en su carpeta:
```bash
sysadmin@servidor:~$ sudo -u user2 touch /srv/user2/prueba3.txt

sysadmin@servidor:~$ sudo -u user2 ls /srv/user2
lost+found  prueba3.txt
```

Comprobar que user2 puede crear en la carpeta compartida
```bash
sysadmin@servidor:~$ sudo -u user2 touch /srv/comun/prueba4.txt
```
Comprobar que user2 no puede acceder al directorio de user1:
```bash
sysadmin@servidor:~$ sudo -u user2 ls /srv/user1
ls: no se puede abrir el directorio '/srv/user1': Permiso denegado
```
Borrar archivos de prueba:
```bash
sysadmin@servidor:~$ sudo rm /srv/user1/prueba.txt /srv/user2/prueba3.txt /srv/comun/prueba2.txt /srv/comun/prueba4.txt
```