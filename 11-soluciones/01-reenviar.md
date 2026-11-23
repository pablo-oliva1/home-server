# 9. Reenviar informe

Durante unas pruebas el informe del lunes no legó a enviarse.

En este apartado se analiza el problema y se configura el servicio de envío para que reintente enviar el informe automáticamente si falla.

**Apartados:**
- Descripción del problema.
- Configurar reintentos.
- Prueba de reintentos.

## Descripción del problema

Se han hecho unas pruebas para incorporar al servidor un sistema de apagado y encendido programado.

Durante las pruebas el servidor no se encendió y no realizó la copia de seguridad ni envió el informe de inicio de semana.

No debería ser un problema ya que La cconfiguración realizada preve la emisión de un informe tras encenderse el servidor, independientemende de la hora, sin embargo no ocurre.

Se revisan los registros:

```bash
sysadmin@servidor:~$ sudo journalctl -b -u enviar-informe.service -u copia-espejo.service --no-pager
```

Al arrancar se lanzan los dos servicios pendientes:

```bash
Starting copia-espejo.service - Copia de los espacios de usuarios...

Starting enviar-informe.service - Envío del informe de estado del servidor...
```

La copia se realiza correctamente:

```bash
Finished copia-espejo.service - Copia de los espacios de usuarios.
```

Pero falla el envío del informe:

```bash
msmtp: no se puede localizar al host smtp.gmail.com: Fallo temporal en la resolución del nombre

msmtp: no se puede enviar correo

enviar-informe.service: Main process exited, code=exited, status=75/TEMPFAIL

enviar-informe.service: Failed with result 'exit-code'.
```

El envío se lanzó segundos después del arranque, antes de que el Wifi estuviera conectado.

## Configuración de reintentos

Se va a reconfigurar `enviar-informe.service` para realizar una serie de reintentos:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/enviar-informe.service
```

Se edita:

```ini
[Unit]
Description=Envío del informe de estado del servidor
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=900
StartLimitBurst=5

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/enviar-informe.sh
Restart=on-failure
RestartSec=60
```
- Se agrega:
    - StartLimitIntervalSec: define (en `man systemd.unit`) una ventana de tiempo durante las cuáles se tolerarán fallos.
    - StartLimitBurst: número máximo de arranques dentro de la ventana de tiempo.
    - Restart: con el valor *on-failure* el servicio se vuelve a lanzar solo si falla.
    - RestartSec: configura el tiempo antes del reinicio.

Se comprueba:

```bash
sysadmin@servidor:~$ sudo systemd-analyze verify /etc/systemd/system/enviar-informe.service

sysadmin@servidor:~$ sudo systemctl daemon-reload

sysadmin@servidor:~$ systemctl show enviar-informe.service -p Restart -p RestartUSec -p StartLimitBurst -p StartLimitIntervalUSec

Restart=on-failure
RestartUSec=1min
StartLimitIntervalUSec=15min
StartLimitBurst=5
```

## Prueba de reintentos

Para provocar un fallo sin desconectar el WiFi (lo que cortaría la conexión SSH) se usa el archivo `/etc/hosts`. Se añade una línea para desviar `smtp.gmail.com` a `127.0.0.1`.

Primero se hace una copia de seguridad de `/etc/hosts`:

```bash
sysadmin@servidor:~$ sudo cp /etc/hosts /etc/hosts.bak
```

Se añade la línea falsa y se lanza el envío sin esperar a que termine (`--no-block`):

```bash
sysadmin@servidor:~$ echo "127.0.0.1 smtp.gmail.com" | sudo tee -a /etc/hosts

sysadmin@servidor:~$ sudo systemctl start --no-block enviar-informe.service
```

Pasados unos segundos, se restaura el archivo original y se comprueba que la línea falsa ha desaparecido:

```bash
sysadmin@servidor:~$ sudo cp /etc/hosts.bak /etc/hosts

sysadmin@servidor:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	servidor

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Se revisa el registro de eventos:

```bash
sysadmin@servidor:~$ sudo journalctl -b -u enviar-informe.service -u copia-espejo.service --no-pager
```

Que devuelve (salida recortada):

```bash
msmtp: no se puede conectar con smtp.gmail.com, puerto 587: Conexión rehusada
Failed with result 'exit-code'.
enviar-informe.service: Scheduled restart job, restart counter is at 1.
msmtp: no se puede conectar con smtp.gmail.com, puerto 587: Conexión rehusada
enviar-informe.service: Scheduled restart job, restart counter is at 2.
msmtp: no se puede conectar con smtp.gmail.com, puerto 587: Conexión rehusada
enviar-informe.service: Scheduled restart job, restart counter is at 3.
Finished enviar-informe.service - Envío del informe de estado del servidor.
```
- Los intentos fallan hasta que se restaura el archivo.

El envío ya no depende de que la red esté disponible en el momento exacto en que se lanza el servicio, si falla por este motivo `systemd` lo repite cada minuto (hasta un máximo de 5 intentos por cada 15 minutis).