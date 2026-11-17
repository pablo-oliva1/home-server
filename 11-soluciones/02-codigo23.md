# Código 23 de rsync

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

Confirmar que hay un volumen montado en `/srv/user1`:

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

Archivo que se guarda en el volumen y se borra inmediatamente, con el objetivo de comprobar que se está pudiendo operar en él.

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

Un segundo script se va a ejecutar cada hora, lanza el anterior script y compara el resultado con el de la hora anterior. Solo envía informe cuando el resultado cambia (al detectar un fallo o al recuperarse del mismo).

También se va a modificar el script `informe-estado.sh`, `vigilar-volumenes.service` y `vigilar-volumenes.timer`.

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