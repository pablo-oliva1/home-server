# home-server

## Descripción del proyecto y objetivo

El objetivo del proyecto es crear un pequeño servidor doméstico de almacenamiento utilizando un portátil obsoleto y dos discos HDD. 

La idea es practicar una serie de habilidades:
- Gestionar un sistema totalmente por línea de comandos (sin entorno gráfico).
- Configurar una IP estática y administrar en remoto por SSH.
- Hacer diagnósticos básicos de conectividad (ip, ping, etc...).
- Particionar discos, gestionar volúmenes lógicos, etc.
- Automatizar el montaje de discos.
- Administrar usuarios, grupos, etc.
- Automatizar tareas con scripts en Bash.
- Programar tareas temporizadas en systemd.


Para mantener el objetivo e idea de la práctica a la hora de implementar un sistema operativo en el servidor se opta por Debian en terminal, para prescindir de entorno gráfico.

## Equipo y discos

El equipo que hará de servidor es un Lenovo Yoga 300-11IBR. Aunque puede haber duda en esto porque el firmware informa *Lenovo Flex 3-1130*, la pegatina refiere a Lenovo Yoga 300-11IBR. Es un equipo de 2016 que cuenta con 3 puertos USB por los que se conectarán los discos.

En cuanto a los discos uno se extrajo de otro portátil en desuso, y otro es un disco externo.


|Disco|Modelo|Capacidad|Función|
|-----|------|---------|-------|
|Principal|Seagate Expansion Portable|2 TB|Espacios de uso principal para usuarios|
|Copias|HGST Travelstar|1 TB|Copias de seguridad|

## Resultado

|Dato|Valor|
|----|-----|
|Sistema operativo|Debian 13.7 sin entorno gráfico|
|IP|192.168.1.250 fijada|
|Usuarios|Administrador (sysadmin), user1 (Linux) y user2 (Windows)|
|Acceso a los archivos|Samba, montaje autmático en Linux y unidades de red en Windows|
|Copias de seguridad|Copia espejo diaria a las 3:00|
|Informe de estado|Por email los lunes a las 8:00, y aviso urgente si falla la copia de seguridad|


## Proceso

- [**1. Instalación**](01-instalacion.md): se instala sistema operativo en el equipo que hará de servidor.
    - Especificaciones del equipo.
    - Instalación de sistema operativo.
    - Primer arranque y comprobación.
- [**2. SSH y red**](02-ssh-red.md): se establece una IP fija.
    - Conexión con el servidor.
    - Comprobación de configuración de red.
    - Establecer una IP fija.
- [**3. Ajustes**](03-ajustes.md): se establece una configuración básica al equipo que hará de servidor.
    - Actualizaciones automáticas.
    - Modificar comportamiento de la tapa.
- [**4. Discos**](04-discos.md):
    - Conectar e identificar discos.
    - Borrado de las firmas previas de los discos.
    - Particionar disco principal para LVM.
    - Estructura LVM del disco principal.
    - Formatear volúmenes del LVM.
    - Preparar disco de backups.
    - Automatización de los montajes.
    - Aplicar y comprobar.
- [**5. Usuarios**](05-usuarios.md):
    - Crear grupo y usuarios.
    - Aplicar permisos.
    - Pruebas de permisos.
- [**6. Samba y smb**](06-samba.md):
    - Instalación.
    - Configurar carpetas compartidas.
    - Crear y aplicar contraseñas.
    - Probar Samba desde el propio servidor.
    - Configurar acceso a servidor y montaje de carpetas en user1 (Linux).
    - Configurar acceso a servidor y montaje de carpetas en user2 (Windows).
- [**7. Copia de seguridad**](07-copias.md):
    - Preparar copia.
    - Pruebas de copia.
    - Desarrollo del script.
    - Permisos y primera ejecución.
    - Pruebas al sistema.
    - Programar copia de seguridad.
    - Retauración de copia.
    - Rotación de archivos de registro.
- [**8. Mantenimiento**](08-mantenimiento.md):
    - Comprobaciones de estado
    - Elaborar un informe de estado.
    - Probar el informe.
    - Configurar msmtp.
    - Prueba de envío por email.
    - Automatizar envío de informe por email.
    - Programar envío de informe.
    - Aviso urgente en caso de fallo de copia.
- [**9. Reenviar informes**](09-reenviar.md): se arregla problema de interrupción de envío de informe.
    - Descripción del problema.
    - Configurar reintentos.
    - Prueba de reintentos.

## Tecnologías usadas

### Sistema y administración
- Debian 13 sin entorno gráfico.
- Windows 11.
- Bash.
- PowerShell.
- OpenSSH.
- systemd.
- unattended-upgrades.
- logrotate.

### Red
- ifupdown.
- resolvconf.

### Almacenamiento
- parted.
- wipefs.
- ext4.
- chattr.

### Compartición de archivos
- Samba, protocolo SMB, smbpasswd, pdbedit, testparm, smbclient.
- cifs-utils.
- cmdkey y net use.

### Copias y monitorización
- rsync.
- msmtp.

## Licencia
El código y la documentación originales de este repositorio están bajo la licencia [MIT](LICENSE). El material de terceros que aparece en él (salidas de terminal, mensajes de programas, herramientas referenciadas, etc.) mantiene sus propias licencias. Algunas guías incluyen operaciones destructivas sobre discos. Revisa los comandos antes de ejecutarlos y úsalos bajo tu propia responsabilidad.