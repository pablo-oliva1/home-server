# 3. Ajustes del sistema

Ajustes finales del servidor: activación de la instalación automática de paquetes de seguridad y modificación del comportamiento de la tapa.

**Apartados:**

- Actualizaciones automáticas.
- Modificar comportamiento de la tapa.

## Actualizaciones automáticas
El paquete `unattended-upgrades` descarga e instala automáticamente actualizaciones de seguridad en segundo plano.

Instalar desde el acceso por ssh:
```bash
sysadmin@servidor:~$ sudo apt update

sysadmin@servidor:~$ sudo apt install -y unattended-upgrades
```

Activar (o reconfigurar) unattended-upgrades:
```bash
sysadmin@servidor:~$ sudo dpkg-reconfigure unattended-upgrades
```

## Modificar comportamiento de la tapa

Al ser un portátil cuando se cierra la tapa se activa la suspensión del sistema.

Este tipo de comportamientos se gestiona con el servicio `systemd-logind`, que permite además gestionar identificadores de inicio de sesión, apagado y suspensión, cambio entre sesiones, etc. Esta serie de eventos están configurados en los archivos `logind.conf` y en el directorio `logind.conf.d`.

> Se puede encontrar más información con `man systemd-logind` y en `man logind.conf.d`.

Estos manuales nos permiten crear un archivo de configuración.

Crear carpeta y archivo:

```bash
sysadmin@servidor:~$ sudo mkdir -p /etc/systemd/logind.conf.d

sysadmin@servidor:~$ sudo nano /etc/systemd/logind.conf.d/tapa.conf
```

Contenido del archivo:
```bash
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```
- Handle Lid Switch (*Manejar Tapa Interruptor*): ignorar.
- HandleLidSwitchExternalPower (*Manejar Tapa Interruptor Energía Externa*): ignorar.
- HandleLidSwitchDocked (*Manejar Tapa Acoplado*): ignorar.

Aplicar:
```bash
sysadmin@servidor:~$ sudo reboot
```