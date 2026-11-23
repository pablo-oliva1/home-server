# Código 23 de rsync

- Descripción del problema.
- Comprobaciones.
- Configuración de scripts.
- Integración en el informe semanal.
- Programar cada hora.


## Descripción del problema

La copia de seguridad falló con el código 23, descrito en `man rsync` como *Partial transfer due to error*: la mayor parte de los archivos se copió, pero algunos archivos o atributos no.

Puede deberse a:
- Falta de permisos para leer algún archivo.
- Disco de destino lleno.
- Enlaces simbólicos rotos.
- Nombres de archivo incompatibles.
- Errores de lectura o escritura en el disco.

En ese caso el registro mostraba *Input/output error* porque el disco se había desconectado en algún momento.

Para detectar a tiempo un volumen que deja de responder se va a preparar un script que compruebe cada hora una serie de señales.

Primero se va a realiazr las comprobaciones a mano.

## Comprobaciones

### Comprobaciones previas

Necesitamos comprobar que realmente hay un volumen montado en `/srv/user1`:

```bash
sysadmin@servidor:~$ mountpoint /srv/user1
/srv/user1 es un punto de montaje
```

Comprobar qué volumen está montado, solamente la columna del origen (el dispositivo):

```bash
sysadmin@servidor:~$ findmnt -no SOURCE /srv/user1
/dev/mapper/datos-user1
```
- `findmnt` enlista todos los sistemas de archivos montados.
- `-n` quita la cabecera y `-o SOURCE` muestra solo la columna de origen.

### Comprobar lectura directa en disco

Linux guarda en memoria una copia de lo que lee en disco, una lectura normal podría deberse a la lectura de esa copia aunque el disco estuviera desconectado (la comprobación daría resultado correcto sin serlo). Por eso se realiza una lectura directa en el disco.

Entonces se va a pedir al sistema que lea un bloque del volumen de user1 directamente del disco:

```bash
sysadmin@servidor:~$ sudo dd if=/dev/mapper/datos-user1 of=/dev/null bs=4096 count=1 iflag=direct
[sudo] contraseña para sysadmin: 
1+0 records in
1+0 records out
4096 bytes (4,1 kB, 4,0 KiB) copied, 0,00245336 s, 1,7 MB/s
```
- `dd` convierte y copia un fichero (`man dd`).
- `if` indica dónde leer el dispositivo que hemos identificado con `findmnt`.
- `of` identifica dónde escribir; en este caso se descarta la salida con `/dev/null` porque es irrelevante.
- `bs` y `count` establece la lectura en un bloque de 4096 bytes.
- `iflag` señala saltarse la lectura en la memoria caché y obliga a que la lectura sea en el disco.

Se comprueba si la lectura funciona:

```bash
sysadmin@servidor:~$ echo $?
0
```

El comando ha generado una lectura de 1 bloque completo (`1+0 records in`), y ha descartado la la escritura (`1+0 records out`). El comando termina sin errores (`0`).

### Comprobar las opciones

El montaje establece no solamente la carpeta donde se monta un volumen sino las opciones de montaje.

```bash
sysadmin@servidor:~$ findmnt -no OPTIONS /srv/user1
rw,relatime
```

El volumen que se monta en `/srv/user1` admite lectura y escritura. Cuando se decta un problema grave ext4 lo vuelve a montar como `ro` (*read only*) para proteger los datos.

### Comprobar escritura (archivo canario)

Un canary file se guarda en el volumen y se borra inmediatamente, con el objetivo de comprobar que se está pudiendo operar en él.

```bash
sysadmin@servidor:~$ sudo touch /srv/user1/.testigo_escritura && sudo rm /srv/user1/.testigo_escritura
[sudo] contraseña para sysadmin: 
sysadmin@servidor:~$ echo $?
0
```

El archivo se crea y se borra sin errores (`0`).

## Configuración de scripts

El informe semanal por email comprueba si cada volumen está montado, pero los actuales discos por USB pueden desconectarse sin que el sistema deje de considerarlos montados.

Se va a desarrollar un script que para cada volumen:
1. Compruebe que está montado y qué dispositivo hay.
2. Lea un bloque directamente del disco (para confirmar que responde).
3. Compruebe que no está en solo lectura.
4. Cree y borre un archivo canario para confirmar que se puede escribir.

De esta forma se obtiene seguridad respecto a la noción de montaje de volumen.

Un segundo script se va a ejecutar cada hora, lanza el anterior script y compara el resultado con el de la hora anterior. Solo envía informe cuando el resultado cambia (al detectar un fallo o al recuperarse del mismo).

También se va a modificar el script `informe-estado.sh`, `vigilar-volumenes.service` y `vigilar-volumenes.timer` para adaptarlos.

### Desarrollo del script de comprobación

El objetivo de este script es que se automaticen las comprobaciones que se han hecho a mano en el paso anterior.

Las variables que se van a usar:
- Volúmenes (lista de las carpetas).
- Archivo canario.
- Errores.
- Dispositivo.

La lógica sería:
1. Iniciar un bucle.
2. ¿Está el volumen montado?
	- No → señalar error y pasar al siguiente ciclo.
	- Sí → continuar ciclo.
3. Identificar dispositivo.
4. ¿Se puede leer bloque directamente del disco?
	- No → señalar error y pasar al siguiente ciclo.
	- Sí → continuar ciclo.
5. ¿Está montado en solo lectura?
	- Sí → señalar error y pasar al siguiente ciclo.
	- No → continuar ciclo.
6. ¿Se puede crear y borrar archivo canario?
	- No → señalar error y pasar al siguiente ciclo.
	- Sí → señalar que está todo correcto y pasa al siguiente ciclo.


```ini
#!/bin/bash

VOLUMENES="/srv/user1 /srv/user2 /srv/copias /srv/comun"
CANARIO=".testigo_escritura"
errores=0

for volumen in $VOLUMENES; do
	
	if ! mountpoint -q "$volumen"; then
		echo "ERROR: $volumen no está montado"
		errores=1
		continue
	fi
	
	dispositivo=$(findmnt -no SOURCE "$volumen")
	
	if ! dd if="$dispositivo" of=/dev/null bs=4096 count=1 iflag=direct status=none; then
		echo "ERROR: el disco de $volumen no responde"
		errores=1
		continue
	fi
	
	if findmnt -no OPTIONS "$volumen" | grep -qE '^ro(,|$)'; then
		echo "ERROR: $volumen está en solo lectura"
		errores=1
		continue
	fi
	
	if ! { touch "$volumen"/"$CANARIO" && rm "$volumen"/"$CANARIO"; }; then
		echo "ERROR: no se puede operar en $volumen"
		errores=1
		continue
	fi
	echo "OK: $volumen ha superado todas las comprobaciones"
done

exit $errores
```

Se agrega el script en:

```bash
sysadmin@servidor:~$ sudo nano /usr/local/sbin/comprobar-volumenes.sh
```

Se modifican permisos:

```bash
sysadmin@servidor:~$ sudo chown root:root /usr/local/sbin/comprobar-volumenes.sh
sysadmin@servidor:~$ sudo chmod 750 /usr/local/sbin/comprobar-volumenes.sh
sysadmin@servidor:~$ sudo bash -n /usr/local/sbin/comprobar-volumenes.sh
```

Se prueba:

```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/comprobar-volumenes.sh; echo $?
OK: /srv/user1 ha superado todas las comprobaciones
OK: /srv/user2 ha superado todas las comprobaciones
OK: /srv/copias ha superado todas las comprobaciones
OK: /srv/comun ha superado todas las comprobaciones
0
```

### Desarrollo de un script de comparación

Este script ejecuta el anterior (`comprobar-volumenes.sh`), compara el código de salida (0 o 1) con el anterior y en caso de errores hace un envío de informe con el aviso de recuperaeción. En cualquier caso guarda el estado más actualizado quer ecibe en un archivo para la siguiente comparación.

Variables:
- La ruta de `comprobar-volumenes.sh`.
- El servicio `enviar-informe.service` ya que `enviar-informe.sh` sólo lo intenta 1 vez.
- La ruta del archivo de estado, en `/var/lib`.

La lógica sería:

1. Se ejecuta script de comprobación y se guarda el código de salida como estado actual (0 o 1).
2. ¿Existe archivo de estado?
	- Si → leer el estado anterior.
	- No → considerar el estado anterior como 0 (todo correcto).
3. ¿El estado actual es distinto al anterior?
	- Si → enviar informe.
	- No → no hacer nada.
4. Guardar el estado actual en el archivo de estado.

```ini
#!/bin/bash

COMPROBAR="/usr/local/sbin/comprobar-volumenes.sh"
ENVIAR="enviar-informe.service"
ESTADO="/var/lib/vigilar-volumenes/estado"
CARPETA="/var/lib/vigilar-volumenes/"

"$COMPROBAR" > /dev/null
actual=$?

if [ -f "$ESTADO" ]; then
	anterior=$(cat "$ESTADO")
else
	anterior=0
fi
if [ "$actual" -ne "$anterior" ]; then
	systemctl start --no-block "$ENVIAR"
fi
if ! [ -d "$CARPETA" ]; then
	mkdir "$CARPETA"
fi
echo "$actual" > "$ESTADO"
```

Se guarda el script en:

```bash
sysadmin@servidor:~$ sudo nano /usr/local/sbin/vigilar-volumenes.sh
```

Asignar propietario:

```bash
sysadmin@servidor:~$ sudo chown root:root /usr/local/sbin/vigilar-volumenes.sh
```

Dar permisos:

```bash
sysadmin@servidor:~$ sudo chmod 750 /usr/local/sbin/vigilar-volumenes.sh
```

Se prueba una ejecución:

```bash
sysadmin@servidor:~$ sudo /usr/local/sbin/vigilar-volumenes.sh
```

Se comprueba archivo de estado:

```bash
sysadmin@servidor:~$ sudo cat /var/lib/vigilar-volumenes/estado
0
```

## Integración en el informe semanal

El informe semanal realiza su propia comprobación en discos, que fue la que falló indicando un montaje que realmente no estaba, así que se debe modificar para que si hay fallo real en el montaje informe de ello. De esta forma cuando `vigilar-voluemes.sh` detecte el fallo envíe el informe por email.

El apartado de *Discos montados* de `informe-estado.sh` se modifica así:

```ini
echo "ESTADO DE VOLÚMENES"
echo
errores=0
if ! "/usr/local/sbin/comprobar-volumenes.sh"; then
	errores=1
fi
echo
echo
```

Se comprueba:

```bash
sysadmin@servidor:/usr/local/sbin$ sudo /usr/local/sbin/informe-estado.sh
```

Devuelve un informe completo con las siguientes líneas nuevas:

```bash
ESTADO DE VOLÚMENES

OK: /srv/user1 ha superado todas las comprobaciones
OK: /srv/user2 ha superado todas las comprobaciones
OK: /srv/copias ha superado todas las comprobaciones
OK: /srv/comun ha superado todas las comprobaciones
```

## Programar cada hora

Se programó anteriormente con las configuraciones que hay en `/etc/systemd/system/`. Se debe crear ahora archivos de configuración para `vigilar-volumenes.sh`:

### vigilar-volumenes.service

Indica qué se ejecuta:

```ini
[Unit]
Description=Vigila el estado de los volúmenes
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/vigilar-volumenes.sh
```

Se crea:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/vigilar-volumenes.service
```

### vigilar-volumenes.timer

Indica cuándo, en este caso se opta por ejecutarlo a cualquier hora, al minuto 30, para que no coincida con la hora de realización de copias de seguridad (3:00 AM).

```ini
[Unit]
Description=Vigilancia cada hora

[Timer]
OnCalendar=*-*-* *:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

Se crea:

```bash
sysadmin@servidor:~$ sudo nano /etc/systemd/system/vigilar-volumenes.timer
```

### Revisión y activación

```bash
sysadmin@servidor:~$ sudo systemd-analyze verify /etc/systemd/system/vigilar-volumenes.service /etc/systemd/system/vigilar-volumenes.timer
```

Se recarga `system`, se ejecuta `.service`, se activa `.timer` y se imprime cuándo será la próxima ejecución:

```bash
sysadmin@servidor:~$ sudo systemctl daemon-reload

sysadmin@servidor:~$ sudo systemctl start vigilar-volumenes.service

sysadmin@servidor:~$ sudo systemctl enable --now vigilar-volumenes.timer

Created symlink '/etc/systemd/system/timers.target.wants/vigilar-volumenes.timer' → '/etc/systemd/system/vigilar-volumenes.timer'.
sysadmin@servidor:~$ systemctl list-timers vigilar-volumenes.timer --no-pager
NEXT                        LEFT LAST PASSED UNIT                    ACTIVATES                
Mon 2026-11-23 15:30:00 CET 8min -         - vigilar-volumenes.timer vigilar-volumenes.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.
sysadmin@servidor:~$ 
```

