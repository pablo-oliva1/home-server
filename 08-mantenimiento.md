# 8. Mantenimiento

En esta parte del proceso se va a realizar una serie de configuraciones orientadas al mantenimiento del servidor, empezando por la configuración de un informe del estado del servidor basado en cinco comprobaciones.

**Apartados:**

- Comprobaciones de estado
- Elaborar un informe de estado.
- Probar el informe.
- Configurar msmtp.
- Prueba de envío por email.
- Automatizar envío de informe por email.
- Programar envío de informe.
- Aviso urgente en caso de fallo de copia.

## Comprobaciones de estado

Se deben realizar al menos 5 comprobaciones:

1. Espacio en disco de cada volumen.
2. Volúmenes montados.
3. Última copia de seguridad y su resultado.
4. Servicios fallidos.
5. Últimas actualizaciones de seguridad.

Comprobar espacio en discos:

```bash
sysadmin@servidor:~$ df -h / /srv/user1 /srv/user2 /srv/comun /srv/copias

S.ficheros              Tamaño Usados  Disp Uso% Montado en
/dev/mmcblk0p2             27G   1,6G   24G   7% /
/dev/mapper/datos-user1   442G   2,1M  442G   1% /srv/user1
/dev/mapper/datos-user2   442G   2,1M  442G   1% /srv/user2
/dev/mapper/datos-comun   295G   2,1M  295G   1% /srv/comun
/dev/sdb1                 916G   2,1M  916G   1% /srv/copias
```

Comprobar volúmenes montados:

```bash
sysadmin@servidor:~$ for d in /srv/user1 /srv/user2 /srv/comun /srv/copias; do mountpoint "$d"; done

/srv/user1 es un punto de montaje
/srv/user2 es un punto de montaje
/srv/comun es un punto de montaje
/srv/copias es un punto de montaje
```

Comprobar última copia de seguridad:

```bash
sysadmin@servidor:~$ sudo tac /var/log/copias.log | sed '/Inicio de la copia/q' | tac

2026-11-01 03:00:13 Inicio de la copia
2026-11-01 03:00:13 OK: user1 copiado (0 archivos eliminados de la copia).
2026-11-01 03:00:13 OK: user2 copiado (0 archivos eliminados de la copia).
2026-11-01 03:00:13 Fin de la copia
```

Comprobar servicios fallidos:
```bash
sysadmin@servidor:~$ systemctl --failed --no-pager
  UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.
```

Comprobar cuando se ejecutaron por última vez las actualizaciones automáticas:

```bash
sysadmin@servidor:~$ systemctl list-timers apt-daily-upgrade.timer --no-pager

NEXT               LEFT LAST            PASSED UNIT           ACTIVATES     
Sun 2026-11-0… 2h 50min Sat 2026-10-3… 21h ago apt-daily-upg… apt-daily-upg…

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```

Comprobar si hay reinicio pendiente:

```bash
sysadmin@servidor:~$ ls /var/run/reboot-required

ls: no se puede acceder a '/var/run/reboot-required': No existe el fichero o el directorio
```

## Elaborar un informe de estado

Para que el informe sea capaz de generarse automáticamente se va a desarrollar un script que automatice su composición.

El script debe:
- Mostrar cabecera con nombre del servidor y fecha en que se genera.
- Mostrar las cinco comprobaciones, cada una en su apartado y con un título.
- FInalizar con una conclusión de una línea.

Se crea el script en:

```bash
sysadmin@servidor:~$ sudo nano /usr/local/sbin/informe-estado.sh
```

Script:
- La marca errores empieza en 0 y cualquier apartado la pone a 1 si detecta un problema.


```bash
#!/bin/bash

discos="/srv/user1 /srv/user2 /srv/comun /srv/copias"

echo "---INFORME DE ESTADO DEL SERVIDOR---"
echo
echo "Server: $(hostname)"
echo "Fecha: $(date)"
echo
echo "ESPACIOS EN DISCOS"
echo
df -h / /srv/user1 /srv/user2 /srv/comun /srv/copias
echo
echo
echo "DISCOS MONTADOS"
echo
errores=0
for disco in $discos; do
    if mountpoint -q "$disco"
    then echo "$disco montado"
    else echo "$disco no se encuentra montado"
    errores=1
    fi
done
echo
echo
echo "ULTIMA COPIA DE SEGURIDAD"
echo
ultima_copia=$(tac /var/log/copias.log | sed '/Inicio de la copia/q' | tac)
echo "$ultima_copia"
    if echo "$ultima_copia" | grep -E -q 'ERROR|AVISO'
    then echo "Hay errores"
    errores=1
    else echo "Sin errores"
    fi
echo
echo
echo "SERVICIOS FALLIDOS"
echo
servicios_fallidos=$(systemctl --failed --no-pager)
echo "$servicios_fallidos"
    if echo "$servicios_fallidos" | grep -E -q '^0'
    then echo "No hay servicios fallidos"
    else echo "Hay servicios fallidos"
    errores=1
    fi
echo
echo
echo "ACTUALIZACIONES Y REINICIO"
echo
echo "Última ejecución de las actualizaciones automáticas:"
systemctl list-timers apt-daily-upgrade.timer --no-pager
    if [ -f /var/run/reboot-required ]
    then echo "Hay un reinicio pendiente"
    errores=1
    else echo "No hay reinicio pendiente"
    fi
echo
echo
echo "CONCLUSIONES"
echo
    if [ "$errores" -ne "0" ];
    then echo "Se han detectado errores"
    else echo "No se han detectado errores"
    fi
```

Se modifican permisos:

```bash
sysadmin@servidor:~$ sudo chown root:root /usr/local/sbin/informe-estado.sh

sysadmin@servidor:~$ sudo chmod 750 /usr/local/sbin/informe-estado.sh
```

Se revisa:

```bash
sysadmin@servidor:~$ sudo bash -n /usr/local/sbin/informe-estado.sh
```

Se ejecuta:

```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/informe-estado.sh

---INFORME DE ESTADO DEL SERVIDOR---

Server: servidor
Fecha: dom 01 nov 2026 15:25:40 CET

ESPACIOS EN DISCOS

S.ficheros              Tamaño Usados  Disp Uso% Montado en
/dev/mmcblk0p2             27G   1,6G   24G   7% /
/dev/mapper/datos-user1   442G   2,1M  442G   1% /srv/user1
/dev/mapper/datos-user2   442G   2,1M  442G   1% /srv/user2
/dev/mapper/datos-comun   295G   2,1M  295G   1% /srv/comun
/dev/sdb1                 916G   2,1M  916G   1% /srv/copias


DISCOS MONTADOS

/srv/user1 montado
/srv/user2 montado
/srv/comun montado
/srv/copias montado


ULTIMA COPIA DE SEGURIDAD

2026-11-01 03:00:13 Inicio de la copia
2026-11-01 03:00:13 OK: user1 copiado (0 archivos eliminados de la copia).
2026-11-01 03:00:13 OK: user2 copiado (0 archivos eliminados de la copia).
2026-11-01 03:00:13 Fin de la copia
Sin errores


SERVICIOS FALLIDOS

  UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.
No hay servicios fallidos


ACTUALIZACIONES Y REINICIO

Última ejecución de las actualizaciones automáticas:
NEXT            LEFT LAST             PASSED UNIT             ACTIVATES       
Mon 2026-11-02…  14h Sun 2026-11-01 … 8h ago apt-daily-upgra… apt-daily-upgra…

1 timers listed.
Pass --all to see loaded but inactive timers, too.
No hay reinicio pendiente


CONCLUSIONES

No se han detectado errores
```

## Probar el informe

Se desmonta uno de los discos:

```bash
sysadmin@servidor:~$ sudo umount /srv/copias
```

Se pide informe:

```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/informe-estado.sh

---INFORME DE ESTADO DEL SERVIDOR---

Server: servidor
Fecha: dom 01 nov 2026 15:31:35 CET

ESPACIOS EN DISCOS

S.ficheros              Tamaño Usados  Disp Uso% Montado en
/dev/mmcblk0p2             27G   1,6G   24G   7% /
/dev/mapper/datos-user1   442G   2,1M  442G   1% /srv/user1
/dev/mapper/datos-user2   442G   2,1M  442G   1% /srv/user2
/dev/mapper/datos-comun   295G   2,1M  295G   1% /srv/comun
/dev/mmcblk0p2             27G   1,6G   24G   7% /


DISCOS MONTADOS

/srv/user1 montado
/srv/user2 montado
/srv/comun montado
/srv/copias no se encuentra montado


ULTIMA COPIA DE SEGURIDAD

2026-11-01 03:00:13 Inicio de la copia
2026-11-01 03:00:13 OK: user1 copiado (0 archivos eliminados de la copia).
2026-11-01 03:00:13 OK: user2 copiado (0 archivos eliminados de la copia).
2026-11-01 03:00:13 Fin de la copia
Sin errores


SERVICIOS FALLIDOS

  UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.
No hay servicios fallidos


ACTUALIZACIONES Y REINICIO

Última ejecución de las actualizaciones automáticas:
NEXT            LEFT LAST             PASSED UNIT             ACTIVATES       
Mon 2026-11-02…  14h Sun 2026-11-01 … 8h ago apt-daily-upgra… apt-daily-upgra…

1 timers listed.
Pass --all to see loaded but inactive timers, too.
No hay reinicio pendiente


CONCLUSIONES

Se han detectado errores
```
- Detecta que el disco no está montado.

Se vuelve a montar el disco:

```bash
sysadmin@servidor:~$ sudo mount /srv/copias
```

## Configurar msmtp

Para no tener que estar revisando manualmente el estado del servidor se va a optar por gestionar un envío del informe por email.

Pasos previos:
1. Hacer una cuenta de Gmail.
2. Activar verificación en dos pasos.
3. Crear una contraseña de aplicación.

Se instala `msmtp` para entregar mensajes a Gmail:

```bash
sysadmin@servidor:~$ sudo apt install -y msmtp msmtp-mta

Installing:                             
  msmtp  msmtp-mta
```

Se configura `msmtp` en `/etc/msmtprc`:

```bash
sysadmin@servidor:~$ sudo nano /etc/msmtprc
```

Y se edita:

```ini
defaults
auth    on
tls     on
tls_trust_file  /etc/ssl/certs/ca-certificates.crt
logfile /var/log/msmtp.log

account gmail
host    smtp.gmail.com
port    587
from    <email>
user    <email>
password        <contraseña_aplicación>

account default : gmail
```

Se protege con permisos:

```bash
sysadmin@servidor:~$ sudo chmod 600 /etc/msmtprc

sysadmin@servidor:~$ sudo ls -l /etc/msmtprc
-rw------- 1 root root 268 nov  1 16:25 /etc/msmtprc
```

## Prueba de envío por email

```bash
sysadmin@servidor:~$ printf "To: <email_destino>\nSubject: Prueba del servidor\n\nMensaje de prueba enviado desde servidor.\n" | sudo msmtp <email_destino>
```

Se puede comprobar el envío mirando el registro líneas con `exitcode=EX_OK`:
```bash
sudo cat /var/log/msmtp.log
```

## Automatizar envío de informe por email

La automatización del envío del informe se va a hacer con otro script que debe hacer tres cosas:
1. Ejecutar el informe y guardar el resultado en una variable.
2. Escoger un asunto de correo según si hay errores o no.
3. Enviarlo con `msmtp`.

El script además debe activarse en dos momentos:
- Los lunes por la mañana.
- Cuando falle la copia nocturna.

Se crea en:

```bash
sysadmin@servidor:~$ sudo nano /usr/local/sbin/enviar-informe.sh
```

Script:
- Se aplica una cabecera que indica que se transmite texto plano (`text/plain`) con el conjunto de caracteres UTF-8 (`charset=UTF-8`).

```bash
#!/bin/bash
DESTINATARIO="<email_destino>"
informe=$(/usr/local/sbin/informe-estado.sh)
        if echo "$informe" | grep -q "Se han detectado errores";
        then
                asunto="[servidor] Informe de estado: HAY ERRORES"
        else
                asunto="[servidor] Informe de estado: todo ok"
        fi
        printf "To: %s\nSubject: %s\nMIME-Version: 1.0\nContent-Type: text/plain; charset=UTF-8\n\n%s\n" "$DESTINATARIO" "$asunto" "$informe" | msmtp "$DESTINATARIO"
```

Se aplican permisos y se prueba el script (debería llegar un correo en la dirección de `<email_destino>`).

```bash
sysadmin@servidor:~$ sudo chown root:root /usr/local/sbin/enviar-informe.sh

sysadmin@servidor:~$ sudo chmod 750 /usr/local/sbin/enviar-informe.sh

sysadmin@servidor:~$ sudo bash -n /usr/local/sbin/enviar-informe.sh

sysadmin@servidor:~$ sudo /usr/local/sbin/enviar-informe.sh
```

## Programar envío de informe

Se va a desarrollar dos configuraciones adicionales en `systemd`: en `.service` (`man systemd.service`) y en `.timer` (`man systemd.timer`). 

### .service

En:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/enviar-informe.service
```

Se configura indicando que se requiere usar internet (`Wants=network...`) y que espere a que la conexión de red esté disponible (`After=network...`):

```ini
[Unit]
Description=Envío del informe de estado del servidor
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/enviar-informe.sh
```

### .timer

En:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/enviar-informe.timer
```

Aquí se configura, para que se envíe los lunes a las 8 de la mañana:

```ini
[Unit]
Description=Informe de estado semanal

[Timer]
OnCalendar=Mon *-*-* 08:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

### Comprobar, probar y activar

```bash
sysadmin@servidor:~$ sudo systemd-analyze verify /etc/systemd/system/enviar-informe.service /etc/systemd/system/enviar-informe.timer

sysadmin@servidor:~$ sudo systemctl daemon-reload

sysadmin@servidor:~$ sudo systemctl start enviar-informe.service

sysadmin@servidor:~$ sudo systemctl enable --now enviar-informe.timer
Created symlink '/etc/systemd/system/timers.target.wants/enviar-informe.timer' → '/etc/systemd/system/enviar-informe.timer'.

sysadmin@servidor:~$ systemctl list-timers enviar-informe.timer --no-pager

NEXT           LEFT LAST PASSED UNIT           ACTIVATES      
Mon 2026-11-0…  13h -         - enviar-inform… enviar-informe…

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```

## Aviso urgente en caso de fallo de copia

Se va a modificar el archivo *copia-espejo.service* para incluir el envío de un aviso si no se logra realizar la copia de seguridad:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/copia-espejo.service
```

Se modifica la configuración agregando la opción *OnFailure=* bajo *After*:
```ini
[Unit]
Description=Copia de los espacios de usuarios
After=local-fs.target
OnFailure=enviar-informe.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/copia-espejo.sh
```
- Indica que en caso de fallo se active *enviar-informe.service*, que se ha configurado en el paso anterior (*Programar envío informe*).

### Comprobación

Se prueba desmontando *copias*:

```bash
sysadmin@servidor:~$ sudo systemctl daemon-reload

sysadmin@servidor:~$ sudo umount /srv/copias

sysadmin@servidor:~$ sudo systemctl start copia-espejo.service

Job for copia-espejo.service failed because the control process exited with error code.
See "systemctl status copia-espejo.service" and "journalctl -xeu copia-espejo.service" for details.

sysadmin@servidor:~$ sudo mount /srv/copias

sysadmin@servidor:~$ sudo systemctl reset-failed copia-espejo.service
```

Debería llegar un email notificando del error.

Tras ello volver a arreglar copia:

```bash
sysadmin@servidor:~$ sudo systemctl start copia-espejo.service
```

