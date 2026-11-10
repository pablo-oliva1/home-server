# Soluciones

Posibles soluciones ante fallos que han ido apareciendo con el uso cotidiano del proyecto.

- Error 13 Permission denied.

## Error 13 Permission denied

Al entrar a la carpeta *Personal* de user1 aparece el siguiente aviso:

> **No se pudo acceder a «Personal»**
> <br>error 13 (Permission denied) opening credential
> <br>file /root/.smb-user1

### Ver estado de montajes automáticos en user1

```bash
systemctl list-units --type=automount --all --no-pager

  UNIT                        LOAD   ACTIVE SUB     DESCRIPTION                 
  home-user1-Documentos-Se… loaded active running home-user1-Documentos-Ser…
● home-user1-Documentos-Se… loaded failed failed  home-user1-Documentos-Ser…
  proc-sys-fs-binfmt_misc.au… loaded active running Arbitrary Executable File F…

Legend: LOAD   → Reflects whether the unit definition was properly loaded.
        ACTIVE → The high-level unit activation state, i.e. generalization of SUB.
        SUB    → The low-level unit activation state, values depend on unit type.

3 loaded units listed.
To show all installed unit files use 'systemctl list-unit-files'.
```

NO carga nombres completos, se repite con `--full`:

```bash
❯ systemctl list-units --type=automount --all --no-pager --full
  UNIT                                                LOAD   ACTIVE SUB     DESCRIPTION                                                  
  home-user1-Documentos-Servidor-Comun.automount    loaded active running home-user1-Documentos-Servidor-Comun.automount
● home-user1-Documentos-Servidor-Personal.automount loaded failed failed  home-user1-Documentos-Servidor-Personal.automount
  proc-sys-fs-binfmt_misc.automount                   loaded active running Arbitrary Executable File Formats File System Automount Point

Legend: LOAD   → Reflects whether the unit definition was properly loaded.
        ACTIVE → The high-level unit activation state, i.e. generalization of SUB.
        SUB    → The low-level unit activation state, values depend on unit type.

3 loaded units listed.
To show all installed unit files use 'systemctl list-unit-files'.
```

Se identica problema con `automount`.

### Ver estado de automount en user1

```bash
❯ sudo systemctl status home-user1-Documentos-Servidor-Personal.automount --no-pager

[sudo] contraseña para user1: 
× home-user1-Documentos-Servidor-Personal.automount
     Loaded: loaded (/etc/fstab; generated)
     Active: failed (Result: mount-start-limit-hit) since Tue 2026-11-10 12:16:51 CET; 2h 30min ago
   Duration: 15min 31.770s
   Triggers: ● home-user1-Documentos-Servidor-Personal.mount
      Where: /home/user1/Documentos/Servidor/Personal
       Docs: man:fstab(5)
             man:systemd-fstab-generator(8)

nov 10 12:13:25 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:13:52 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:15:22 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:16:50 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Got automount request for /home/user1/Documentos/…vfsd-recent)
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.automount: Failed with result 'mount-start-limit-hit'.
Hint: Some lines were ellipsized, use -l to show in full.
[ble: exit 3]
```

Se identifica un fallo relacionado con el limites de montajes. systemd limia por defecto cuántas veces puede fallar una unidad en poco tiempo, y al superarse el límite `.mount` y `.automount` quedan marcados como fallidos.

### Ver estado de mount en user1

```bash
❯ sudo systemctl status home-user1-Documentos-Servidor-Personal.mount --no-pager

× home-user1-Documentos-Servidor-Personal.mount - /home/user1/Documentos/Servidor/Personal
     Loaded: loaded (/etc/fstab; generated)
     Active: failed (Result: exit-code) since Tue 2026-11-10 12:16:51 CET; 2h 39min ago
 Invocation: 230e15fcdba343018ce1dee6e0a49ff0
TriggeredBy: × home-user1-Documentos-Servidor-Personal.automount
      Where: /home/user1/Documentos/Servidor/Personal
       What: //192.168.1.250/user1
       Docs: man:fstab(5)
             man:systemd-fstab-generator(8)

nov 10 12:16:51 debian mount[38980]: This can happen if the server does not support the SMB version you are trying to use.
nov 10 12:16:51 debian mount[38980]: The default SMB version recently changed from SMB1 to SMB2.1 and above. Try mounting with vers=1.0.
nov 10 12:16:51 debian mount[38980]: mount error(112): Host is down
nov 10 12:16:51 debian mount[38980]: Refer to the mount.cifs(8) manual page (e.g. man mount.cifs) and kernel log messages (dmesg)
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.mount: Mount process exited, code=exited, status=32/n/a
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.mount: Failed with result 'exit-code'.
nov 10 12:16:51 debian systemd[1]: Failed to mount home-user1-Documentos-Servidor-Personal.mount - /home/user1/Documentos/Servidor/Personal.
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.mount: Start request repeated too quickly.
nov 10 12:16:51 debian systemd[1]: home-user1-Documentos-Servidor-Personal.mount: Failed with result 'exit-code'.
nov 10 12:16:51 debian systemd[1]: Failed to mount home-user1-Documentos-Servidor-Personal.mount - /home/user1/Documentos/Servidor/Personal.
[ble: exit 3]
```

Se identifica:
- Durante un segundo a las 12:16:51 se hicieron repetidos intentos de montaje.
- Sugiere que el servidor puede no soportar la versión de SMB que se está intentando usar.
- Sugiere como causa un cambio de SMB1 a SMB2 e intentar un montaje con SMB1. Se descarta que el fallo se deba a esto porque la carpeta `Común` se monta sin problemas.
- Informa que el host está caído.
- Da referencias a manuales de cifs y dmesg.
- Tras varios intentos de montaje se disparó el límite de intentos de montaje establecidos en la configuración y se detuvo el sistema.

### Pruebas de conectividad servidor-user1

Se realizan las siguientes pruebas de conectividad mediante ping:

| Ruta | Tiempo |
|------|--------|
| user1 a router | promedio 3 ms |
| Servidor a router | promedio 2 ms |
| user1 a servidor | promedio 230 ms |
| servidor a user1 | promedio 72 ms |

Se repite la prueba de user1 a servidor, se comprueba que se produce "diente de sierra": 243, 266, 519, 162, 281, 306, 401...

Repasando el checklist de problemas de red:
- Si el problema se debiera a saturación del canal se reflejaría en los promedios de user1-router y servidor-router.
- Tampoco es un problema de saturación de la línea porque las pruebas no salen a internet.
- Si se debiera a dispositivos en segundo plano el patrón no sería tan consistente (user1-router, servidor-router, user1-router).
- Tampoco se debe a problemas de enrutamiento porque no hay salida a internet.
- Se descarta también el sobrecalentamiento del router porque no hay retrasos en todas las direcciones.

Una opción que hay que descartar es que el modo de ahorro de energía "duerma" la tarjeta de red y tarde en recibir conexiones.

### Pruebas desactivando ahorro de energía en servidor

Puede encontrarse info en `sudo iw help`:

```
dev <devname> set power_save <on|off>
		Set power save state to on or off.

dev <devname> get power_save 
		Retrieve power save state.
```

Se instalar herramienta de configuración de tarjetas de red (`man iw`):

```bash
sysadmin@servidor:~$ sudo apt install -y iw
```

Se comprueba estado de ahorro de energía:

```bash
sysadmin@servidor:~$ sudo iw dev wlp2s0 get power_save
Power save: on
```

Se desactiva ahorro de energía:

```bash
sysadmin@servidor:~$ sudo iw dev wlp2s0 set power_save off

sysadmin@servidor:~$ sudo iw dev wlp2s0 get power_save
Power save: off
```

Se repiten pruebas de conectividad:

| Ruta | Tiempo |
|------|--------|
| user1 a router | promedio 2 ms |
| Servidor a router | promedio 6 ms |
| user1 a servidor | promedio 14 ms |
| servidor a user1 | promedio 55 ms |

Comparativa:

| Ruta | Timpo antes | Tiempo después |
|------|-------------|----------------|
| user1 a router | promedio 3 ms | promedio 2 ms |
| Servidor a router | promedio 2 ms | promedio 6 ms |
| user1 a servidor | promedio 230 ms | promedio 14 ms |
| servidor a user1 | promedio 72 ms | promedio 55 ms |

### Aplicar desactivado persistente de power_save en servidor

Hacemos una copia del archivo de configuración de red antes de modificarlo, y se comprueba:

```bash
sysadmin@servidor:~$ sudo cp /etc/network/interfaces /etc/network/interfaces.bak-ahorro

sysadmin@servidor:~$ ls -l /etc/network/
total 32
drwxr-xr-x 2 root root 4096 oct 29 00:16 if-down.d
drwxr-xr-x 2 root root 4096 oct 28 22:40 if-post-down.d
drwxr-xr-x 2 root root 4096 oct 28 22:40 if-pre-up.d
drwxr-xr-x 2 root root 4096 oct 29 00:16 if-up.d
-rw------- 1 root root  454 oct 29 00:58 interfaces
-rw------- 1 root root  365 oct 29 00:11 interfaces.bak
-rw------- 1 root root  454 nov 10 17:11 interfaces.bak-ahorro
drwxr-xr-x 2 root root 4096 ene  8  2026 interfaces.d
```

Se edita `interfaces`:

```bash
sysadmin@servidor:~$ sudo nano /etc/network/interfaces
```

Y al final de la última edición se aplica:

```ini
post-up /usr/sbin/iw dev wlp2s0 set power_save off || true
```

Se puede comprobar cambio reiniciando el servidor y comprobando el estado de `power_save`. Además es útil aplicar el mismo procedimiento en el PC de user1.

### Desbloquear carpeta personal en user1

Tras los fallos en montaje systemd marcó la unidad como fallida al superar los límites de intentos y mientras sigan marcadas no se vuele a intentar montar.

En `man systemctl` se menciona la opción `reset-failed [PATTERN]` para reiniciar el estado de unidades específicas tras algún fallo (en este caso para `.mount` y `.automount`):

```bash
❯ sudo systemctl reset-failed home-user1-Documentos-Servidor-Personal.mount home-user1-Documentos-Servidor-Personal.automount
```

Y en el mismo manual encontramos `start [PATTERN]` para iniciar unidades:

```bash
❯ sudo systemctl start home-user1-Documentos-Servidor-Personal.automount
```

Tras esto se entrar:

```bash
❯ ls -l ~/Documentos/Servidor/Personal
```
