# 7. Copia de seguridad

Se va a instalar una herramienta de gestión de copias de seguridad, se desarrollará un script para automatizar las copias de seguridad y se creará un registro de consulta. Además se hará una serie de pruebas al sistema de copia de seguridad, se programará la copia de seguridad y se automatizará la rotación de archivos de registro.

**Apartados:**
- Preparar copia.
- Pruebas de copia.
- Desarrollo del script.
- Permisos y primera ejecución.
- Pruebas al sistema.
- Programar copia de seguridad.
- Restauración de copia.
- Rotación de archivos de registro.

## Preparar copia

Instalar `rsync` para gestionar copias de seguridad:
```bash
sysadmin@servidor:~$ sudo apt install rsync

Installing:                             
  rsync
```

Crear carpetas de destino:

```bash
sysadmin@servidor:~$ sudo mkdir -p /srv/copias/user1 /srv/copias/user2
```

Se crea un "archivo testigo" en cada volumen, de forma que si el script no lo encuentra (por desconexión del disco u otra razón) se detiene.
```bash
sysadmin@servidor:~$ sudo touch /srv/user1/.testigo_user1 /srv/user2/.testigo_user2 /srv/copias/.testigo_copias
```

Para evitar que sean borrados accidentalmente por algún usuario se cambian las propiedades (atributos) con `chattr` (`+i` agrega inmutabilidad):
```bash
sysadmin@servidor:~$ sudo chattr +i /srv/user1/.testigo_user1 /srv/user2/.testigo_user2 /srv/copias/.testigo_copias
```

Además conviene ocultarlos agregando una línea a la configuración de `smb.conf`:

```bash
sysadmin@servidor:~$ sudo nano /etc/samba/smb.conf
```

Dentro de la sección `[global]`:

```ini
veto files = /.testigo_*/
```

Se comprueba la configuración y se reinicia Samba para aplicar el cambio:

```bash
sysadmin@servidor:~$ testparm -s

sysadmin@servidor:~$ sudo systemctl restart smbd
```

Comprobación de los archivos testigo:

```bash
sysadmin@servidor:~$ sudo ls -la /srv/user1 /srv/user2 /srv/copias

/srv/copias:
total 32
drwx------ 5 root root  4096 oct 30 22:06 .
drwxr-xr-x 6 root root  4096 oct 29 13:48 ..
drwx------ 2 root root 16384 oct 29 13:38 lost+found
-rw-r--r-- 1 root root     0 oct 30 22:06 .testigo_copias
drwxr-xr-x 2 root root  4096 oct 30 21:54 user1
drwxr-xr-x 2 root root  4096 oct 30 21:54 user2

/srv/user1:
total 24
drwx------ 3 user1 user1  4096 oct 30 22:06 .
drwxr-xr-x 6 root  root   4096 oct 29 13:48 ..
drwx------ 2 root  root  16384 oct 29 13:12 lost+found
-rw-r--r-- 1 root  root      0 oct 30 22:06 .testigo_user1

/srv/user2:
total 24
drwx------ 3 user2 user2  4096 oct 30 22:06 .
drwxr-xr-x 6 root  root   4096 oct 29 13:48 ..
drwx------ 2 root  root  16384 oct 29 13:12 lost+found
-rw-r--r-- 1 root  root      0 oct 30 22:06 .testigo_user2
```

## Pruebas de copia

Hecho esto se puede hacer una prueba de cómo trabajaría `rsync` con la opción `--dry-run`. El manual (`man rsync`) muestra una buena cantidad de opciones:
- `-a` es una opción que por sí sola actúa implicando recursividad de la copia, los permisos, fechas de modificación, grupo, propietario, etc.
- `-H` conserva enlaces duros (evita duplicado de bloques de datos).
- `-A` conserva las ACL.
- `-X` mantiene los atributos extendidos.
- `--delete` elimina de la carpeta de destino lo que ya no hay en el origen.
- `--exclude=PATTERN` se puede usar para excluir (como el *lost+found*).
- `--dry-run` hace que se realice como prueba de ejecución (sin cambios reales).
- `--itemize-changes` devuelve una lista de los cambios.

```bash
sysadmin@servidor:~$ sudo rsync -aHAX --delete --exclude=lost+found --dry-run --itemize-changes /srv/user1/ /srv/copias/user1/

.d..tpog... ./
>f+++++++++ .testigo_user1
```

Ahora se procede a realizar dos copias reales:
```bash
sysadmin@servidor:~$ sudo rsync -aHAX --delete --exclude=lost+found --itemize-changes /srv/user1/ /srv/copias/user1/

.d..tpog... ./
>f+++++++++ .testigo_user1

sysadmin@servidor:~$ sudo rsync -aHAX --delete --exclude=lost+found --itemize-changes /srv/user2/ /srv/copias/user2/

.d..tpog... ./
>f+++++++++ .testigo_user2
```

Se comprueba:
```bash
sysadmin@servidor:~$ sudo ls -la /srv/copias/user1 /srv/copias/user2

/srv/copias/user1:
total 8
drwx------ 2 user1 user1 4096 oct 30 22:06 .
drwx------ 5 root  root  4096 oct 30 22:06 ..
-rw-r--r-- 1 root  root     0 oct 30 22:06 .testigo_user1

/srv/copias/user2:
total 8
drwx------ 2 user2 user2 4096 oct 30 22:06 .
drwx------ 5 root  root  4096 oct 30 22:06 ..
-rw-r--r-- 1 root  root     0 oct 30 22:06 .testigo_user2
```

## Desarrollo del script

Para desarrollar el script de automatización primero se seleccionan una serie de variables con las que tendrá que trabajar el script. Las básicas para copia de seguridad son:
- Usuarios.
- Origen de la copia.
- Destino de la copia.

Como van a automatizarse las opciones de `rsync` y se va a implementar una serie de barreras de seguridad (para evitar borrados totales de la copia en caso de que haya desconectado el disco), otras dos variables necesarias son:
- Un número límite a la cantidad de archivos que puede borrar el programa por sí solo.
- Las opciones reiteradas que tendrá el comando `rsync` (recursividad, conservación, ACL... etc).

Se van implementar 5 condiciones de seguridad:
1. Comprobar que la carpeta de destino es un punto de montaje.
2. Comprobar que esa carpeta tiene el archivo testigo.
3. Comprobar que la carpeta de origen es un punto de montaje.
4. Comprobar que en esa carpeta está el archivo testigo.
5. COmprobar que no van a eliminarse un número mayor de archivos que el límite fijado.

Si no se cumplen las condiciones anteriores no se realizará la copia de seguridad de ese usuario para proteger la que ya está almacenada.

La lógica sería:
1. ¿El destino es un punto de montaje?
    - No → registra el error y finaliza script.
    - Si → continúa.
2. ¿Está en la carpeta destino el archivo testigo?
    - No → registra el error y finaliza script.
    - Si → continúa.
3. Inicia un bucle: 
    - A1. ¿La carpeta origen user1 es un punto de montaje?
        - No → registra y marca error, y pasa al siguiente ciclo.
        - Si → continúa ciclo.
    - B1. ¿Tiene la carpeta origen de user1 el archivo testigo?
        - No → registra y marca error, y pasa al siguiente ciclo.
        - Si → continúa ciclo.
    - C1. Simula el copiado y borrado de lo que ya no hay en carpeta de origen de user1, filtra y cuenta cuántas líneas empiezan por "deleting". ¿El número de líneas es mayor que el límite determinado?
        - No → continúa ciclo.
        - Si → registra y marca error, y pasa al siguiente ciclo.
    - D1. ¿Se completa copiado de origen a destino?
        - No → registra y marca error, y pasa al siguiente ciclo.
        - Si → registra confirmación y pasa al siguiente ciclo.
    - A2. ¿La carpeta origen de user2 es un punto de montaje?
        - No → registra y marca error, y finaliza bucle.
        - Si → continúa ciclo.
    - B2. ¿Tiene la carpeta origen de user2 el archivo testigo?
        - No → registra y marca error, y finaliza bucle.
        - Si → continúa ciclo.
    - C2. Simula el copiado y borrado de lo que ya no hay en carpeta de origen de user2, filtra y cuenta cuántas líneas empiezan por "deleting". ¿El número de líneas es mayor que el límite determinado?
        - No → continúa ciclo.
        - Si → registra y marca error, y finaliza bucle.
    - D2. ¿Se completa copiado de origen a destino?
        - No → registra y marca error, y finaliza bucle.
        - Si → registra confirmación y finaliza bucle.
4. Fin del bucle, marca de fin de proceso, señala si hubo errores y finaliza el script.

Además como es necesario que se genere un registro de copias el script debe integrar una función que comience por datar la fecha y hora completas (función *registrar*), sin que sobreescriba registros anteriores (>> y no >).

Se crea en archivo del script en:

```bash
sysadmin@servidor:~$ sudo nano /usr/local/sbin/copia-espejo.sh
```

Y se desarrolla:
```bash
#!/bin/bash

USUARIOS="user1 user2"
DESTINO="/srv/copias"
REGISTRO="/var/log/copias.log"
MAX_BORRADOS=300
OPCIONES=(-aHAX --delete --exclude=lost+found)

registrar() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') $1" >> "$REGISTRO"
}

registrar "Inicio de la copia"

if ! mountpoint -q "$DESTINO" || [ ! -f "$DESTINO/.testigo_copias" ]; then
        registrar "ERROR: el disco de copias no está montado o falta su testigo. Copia cancelada"
        exit 1
fi

errores=0

for usuario in $USUARIOS; do
        origen="/srv/$usuario"

        if ! mountpoint -q "$origen" || [ ! -f "$origen/.testigo_$usuario" ]; then
                registrar "ERROR: $origen no está montado o falta su testigo. Se omite $usuario."
                errores=1
                continue
        fi

        borrados=$(rsync "${OPCIONES[@]}" --dry-run --itemize-changes "$origen/" "$DESTINO/$usuario/" | grep -c '^\*deleting')
        if [ "$borrados" -gt "$MAX_BORRADOS" ]; then
                registrar "AVISO: la copia de $usuario borraría $borrados archivos (límite: $MAX_BORRADOS). Se omite; revisar a mano."
                errores=1
                continue
        fi

        if rsync "${OPCIONES[@]}" "$origen/" "$DESTINO/$usuario/"; then
                registrar "OK: $usuario copiado ($borrados archivos eliminados de la copia)."
        else
                registrar "ERROR: rsync falló al copiar $usuario (código $?)."
                errores=1
        fi
done

registrar "Fin de la copia"
exit $errores
```
> [!NOTE]
> En una versión anterior del documento `MAX_BORRADOS` estaba en 100 (por eso la *Prueba 2* usa 101 archivos). Más adelante en el paso 9 se subió a 300.


## Permisos y primera ejecución

Se agrega propietario y se modifican permisos atendiendo a los permisos de ejecución:

```bash
sysadmin@servidor:~$ sudo chown root:root /usr/local/sbin/copia-espejo.sh

sysadmin@servidor:~$ sudo chmod 750 /usr/local/sbin/copia-espejo.sh

sysadmin@servidor:~$ ls -l /usr/local/sbin/copia-espejo.sh
-rwxr-x--- 1 root root 1290 oct 31 17:46 /usr/local/sbin/copia-espejo.sh
```

Primera ejecución:

```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/copia-espejo.sh

sysadmin@servidor:~$ echo $?
0
```

Se comprueba el registro:

```bash
sysadmin@servidor:~$ sudo cat /var/log/copias.log

2026-10-31 17:55:53 Inicio de la copia
2026-10-31 17:55:53 OK: user1 copiado (0 archivos eliminados de la copia).
2026-10-31 17:55:53 OK: user2 copiado (0 archivos eliminados de la copia).
2026-10-31 17:55:53 Fin de la copia
```

## Pruebas al sistema

### Prueba 1: disco de copias desmontado

Se desmonta disco de copias con `umount` y se comprueba con `mountpoint`:

```bash
sysadmin@servidor:~$ sudo umount /srv/copias

sysadmin@servidor:~$ mountpoint /srv/copias
/srv/copias no es un punto de montaje
```

Se ejecuta script:
```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/copia-espejo.sh

sysadmin@servidor:~$ echo $?
1
```

Se comprueba el registro:

```bash
sysadmin@servidor:~$ sudo tail -n 3 /var/log/copias.log

2026-10-31 17:55:53 Fin de la copia
2026-10-31 18:05:14 Inicio de la copia
2026-10-31 18:05:14 ERROR: el disco de copias no está montado o falta su testigo. Copia cancelada
```

Se vuelve a montar disco:

```bash
sysadmin@servidor:~$ sudo mount /srv/copias

sysadmin@servidor:~$ mountpoint /srv/copias
/srv/copias es un punto de montaje
```

### Prueba 2: límite de borrados

Se crean 101 archivos en user1:

```bash
sysadmin@servidor:~$ sudo -u user1 touch /srv/user1/prueba_{1..101}.txt

sysadmin@servidor:~$ sudo ls /srv/user1 | wc -l
102
```

Se hace un copiado estándar con `rsync`:

```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/copia-espejo.sh

sysadmin@servidor:~$ echo $?
0

sysadmin@servidor:~$ sudo ls /srv/copias/user1 | wc -l
101
```

Se borran los 101 archivos de la carpeta original de user1:

```bash
sysadmin@servidor:~$ sudo rm /srv/user1/prueba_{1..101}.txt
```

Se ejecuta el script:

```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/copia-espejo.sh

sysadmin@servidor:~$ echo $?
1

sysadmin@servidor:~$ sudo tail -n 4 /var/log/copias.log
2026-10-31 18:15:09 Inicio de la copia
2026-10-31 18:15:09 AVISO: la copia de user1 borraría 101 archivos (límite: 100). Se omite; revisar a mano.
2026-10-31 18:15:10 OK: user2 copiado (0 archivos eliminados de la copia).
2026-10-31 18:15:10 Fin de la copia
```

Se comprueba que la copia de user1 sigue intacta:

```bash
sysadmin@servidor:~$ sudo ls /srv/copias/user1 | wc -l
101
```

> **Resolución del aviso:**
> 1. Ver las primeras líneas de qué se borraría:    
    `sudo rsync -aHAX --delete --exclude=lost+found --dry-run --itemize-changes /srv/user1/ /srv/copias/user1/ | grep '^\*deleting' | head`
> 2. Si el borrado procede aplicar rsync sin límite: 
    `sudo rsync -aHAX --delete --exclude=lost+found /srv/user1/ /srv/copias/user1/`
> 3. Comprobar:
>- `sudo ls /srv/copias/user1 | wc -l`
>- `sudo /usr/local/sbin/copia-espejo.sh`
>- `echo $?`
>- `sudo tail -n 4 /var/log/copias.log`


## Programar copia de seguridad

Se establece la programación de copias con `systemd`, un programa de administración de sistemas y servicios para sistemas basados en Linux. En este caso el programa diferencia entre qué se ejecuta y cuando, en dos archivos diferentes: `.service` y `.timer`.

Ambos tipos de archivos se estructuran de una forma determinada y en cualquiera de los dos casos puede consultarse `man systemd.service` y `man systemd.timer`.

|`.service`|`.timer`|
|----|----|
|Se compone de una sección *Unit* (con la descripción, servicios asociados, etc), y una sección *Service* donde se define el tipo de servicio, los comandos, permisos, etc.|También con una sección *Unit*, la sección *Timer* (temporizadores de calendario, de eventos y opciones de control), y una sección *Install* donde se define cómo se habilita.|

### .service

Se crea el archivo en:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/copia-espejo.service
```

Y se edita:

```ini
[Unit]
Description=Copia de los espacios de usuarios
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/copia-espejo.sh
```
- *After=local-fs.target*: si se lanza al arrancar espera a que los discos estén montados.
- *Type=oneshot*: finaliza en cuanto termina su ejecución.
- *ExecStart*: se ejecuta el script.

### .timer

Se crea el archivo:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/copia-espejo.timer
```

Y se edita:

```ini
[Unit]
Description=Copia espejo diaria

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

- *OnCalendar*: cuándo se ejecuta (todos los días a las 3).
- *Persistent=true*: en caso de que el servidor estuviera apagado ejecuta la copia al arrancarse.
- *WantedBy=timers.target*: activa el temporizador en cada arranque.

### Carga de archivos y prueba

```bash
sysadmin@servidor:~$ sudo systemd-analyze verify /etc/systemd/system/copia-espejo.service

sysadmin@servidor:~$ sudo systemctl daemon-reload

sysadmin@servidor:~$ sudo systemctl start copia-espejo.service

sysadmin@servidor:~$ systemctl status copia-espejo.service --no-pager
○ copia-espejo.service - Copia de los espacios de usuarios
     Loaded: loaded (/etc/systemd/system/copia-espejo.service; static)
     Active: inactive (dead)

sysadmin@servidor:~$ sudo tail -n 4 /var/log/copias.log

2026-10-31 23:09:54 Inicio de la copia
2026-10-31 23:09:54 OK: user1 copiado (0 archivos eliminados de la copia).
2026-10-31 23:09:54 OK: user2 copiado (0 archivos eliminados de la copia).
2026-10-31 23:09:54 Fin de la copia
```

Se activa el temporizador:
```bash
sysadmin@servidor:~$ sudo systemctl enable --now copia-espejo.timer

Created symlink '/etc/systemd/system/timers.target.wants/copia-espejo.timer' → '/etc/systemd/system/copia-espejo.timer'.
```

Se comprueba el estado del temporizador:

```bash
sysadmin@servidor:~$ systemctl list-timers copia-espejo.timer --no-pager

NEXT                   LEFT LAST PASSED UNIT               ACTIVATES          
Sun 2026-11-01 03… 3h 46min -         - copia-espejo.timer copia-espejo.servi…

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```

## Restauración de copia

Se va a hacer una simulación de restauración de copia de seguridad. Por ejemplo, que user1 tien un determinado archivo, se hace la copia pero por error lo borra de su espacio personal y quiere recuperarlo.

user1 crea un archivo:

```bash
sysadmin@servidor:~$ echo "Documento importante" | sudo -u user1 tee /srv/user1/importante.txt
```

Se simula la copia:

```bash
sysadmin@servidor:~$ sudo systemctl start copia-espejo.service

sysadmin@servidor:~$ sudo ls -l /srv/copias/user1/importante.txt

-rw-rw-r-- 1 user1 user1 21 oct 31 23:23 /srv/copias/user1/importante.txt
```

user1 borra archivo:

```bash
sysadmin@servidor:~$ sudo -u user1 rm /srv/user1/importante.txt

sysadmin@servidor:~$ sudo ls -l /srv/user1/importante.txt

ls: no se puede acceder a '/srv/user1/importante.txt': No existe el fichero o el directorio
```

El administrador lo restaura desde la copia (copia desde el disco de copias a la carpeta):

```bash
sysadmin@servidor:~$ sudo rsync -aHAX /srv/copias/user1/importante.txt /srv/user1
```

> Al restaurar:
> - El comando no lleva `--delete` porque borraría del espacio del usuario todo lo que haya tras la última copia.
> - Es importante restaurar antes de la siguiente hora de copia automática.

Comprobación:

```bash
sysadmin@servidor:~$ sudo ls -l /srv/user1/importante.txt
-rw-rw-r-- 1 user1 user1 21 oct 31 23:23 /srv/user1/importante.txt

sysadmin@servidor:~$ sudo cat /srv/user1/importante.txt
Documento importante
```

## Rotación de archivos de registro

Este apartado se introduce porque el script que se ha desarrollado antes escribe cuatro líneas de registro cada vez que se ejecuta, pudiendo el archivo de registro crecer de forma indefinida y, potencialmente, saturar el disco.

La rotación de archivos de registro va a consistir en:
1. Archivar con otro nombre el registro actual,
2. Empezar un registro nuevo.
3. Ir borrando los archivos más antiguos.

Esto se va a realizar con la herramienta `logrotate` cuyo manual puede consultarse con `man logrotate`.

Se crea el archivo:

```bash
sysadmin@servidor:~$ sudo nano /etc/logrotate.d/copias
```

Configuración:

```ini
/var/log/copias.log {
        monthly
        rotate 12
        compress
        delaycompress
        missingok
        notifempty
}
```
- Monthly: frecuencia con la que debe rotar el archivo.
- Rotate 12: cuántas copias antiguas se conservan (las de los últimos 12 meses.)
- Compress: comprime los registros archivados.
- Delaycompress: retrasa la compresión del nuevo archivo hasta el siguiente ciclo.
- Missingok: evita que se genere error si el archivo de registro no existe o no se encuentra.
- Notifempty: no realiza rotación si el archivo está vacío

Comprobar rotación y prueba:

```bash
sysadmin@servidor:~$ sudo logrotate --debug /etc/logrotate.d/copias

warning: logrotate in debug mode does nothing except printing debug messages!  Consider using verbose mode (-v) instead if this is not what you want.

reading config file /etc/logrotate.d/copias
Reading state from file: /var/lib/logrotate/status
Allocating hash table for state file, size 64 entries
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state

Handling 1 logs

rotating pattern: /var/log/copias.log monthly empty log files are not rotated, (12 rotations), old logs are removed
considering log /var/log/copias.log
Creating new state
  Now: 2026-11-01 01:48
  Last rotated at 2026-11-01 01:00
  log does not need rotating (log has already been rotated)
```
- *Last rotated at...* no significa que haya rotado sino que es la primera vez que `logrotate` ve el registro y coloca la fecha como punto de partida.