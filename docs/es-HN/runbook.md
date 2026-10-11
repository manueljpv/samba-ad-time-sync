# Runbook: sincronización de hora en un dominio Samba AD

[← README](../../README.es-HN.md) · [Revertir](rollback.md) · [Lecciones aprendidas](lessons-learned.md)

> **Aviso: datos ficticios.** Todos los nombres, direcciones y valores de este documento son ficticios (dominio `example.local`, rangos reservados para documentación por la RFC 5737). Adáptelos a su entorno antes de usarlos, y haga cada cambio con una copia de respaldo y una estación de prueba.

## 1. Escenario

- Dominio `example.local`, con un controlador `dc01.example.local` (`192.0.2.10`): Samba AD DC compilado desde el código fuente en `/usr/local/samba`, Rocky Linux 10.x, máquina virtual (KVM).
- Estaciones Windows en OU de computadoras organizadas por departamento.
- Redes que consultan el servidor de hora: `192.0.2.0/24` (red principal, con servidores y algunas estaciones), `198.51.100.0/24` y `203.0.113.0/24` (redes de estaciones). Las redes de celulares, visitantes e IoT no tienen máquinas del dominio y **no se habilitaron**, a propósito.

## 2. Problema

Dispositivos del dominio con la hora desfasada por minutos: unos adelantados, otros atrasados.

## 3. Diagnóstico

Haga el diagnóstico antes de cambiar cualquier cosa, y anote los valores.

**En el servidor** (solo lectura):

```bash
date; timedatectl                         # hora y zona horaria
chronyc tracking; chronyc sources -v      # sincronización y fuentes
grep -v -E '^\s*(#|$)' /etc/chrony.conf   # configuración en uso
samba -b | grep -i NTP_SIGND_SOCKET_DIR   # ruta del socket de firma
ls -ld /ruta/del/ntp_signd                # permisos del directorio
```

**En una estación Windows** (PowerShell como administrador, solo lectura):

```
w32tm /query /source
w32tm /query /status
w32tm /stripchart /computer:dc01.example.local /samples:3 /dataonly
w32tm /query /configuration
nltest /dsgetdc:example.local /TIMESERV
```

Lo que encontramos:

- El servidor `dc01` tenía la hora correcta (diferencia de cerca de 0.1 ms contra servidores NTP públicos) y `chronyd` activo, **pero sin atender clientes**: el `chrony.conf` no tenía `allow` ni `ntpsigndsocket`.
- Estaciones medidas antes de la corrección:

| Estación | Diferencia con el servidor | Fuente de hora | Última sincronización |
|---|---|---|---|
| WS-B | cerca de 1.75 s | Local CMOS Clock | nunca |
| WS-C | cerca de 11.2 s | Local CMOS Clock | nunca |
| WS-A | cerca de 128 s (2 min 8 s) | Local CMOS Clock | nunca |

- La configuración de los clientes ya era correcta (`Type: NT5DS`, cliente NTP habilitado), y el dominio ya anunciaba a `dc01` como servidor de hora (marcas `TIMESERV` y `GTIMESERV` en `nltest`).
- **Causa:** el servidor no respondía hora **firmada** (MS-SNTP) a los clientes Windows. Sin una respuesta válida, cada reloj seguía su propio hardware y se fue desviando.
- La ruta UDP 123 entre las redes se validó con `w32tm /stripchart`.

## 4. Cambios en el servidor (dc01)

Todos reversibles (vea [rollback.md](rollback.md)). Samba no necesitó ninguna opción nueva en `smb.conf`. Haga un paso a la vez y revise el resultado antes del siguiente.

**Paso 1. Dar al usuario `chrony` acceso al directorio del socket de firma** (comandos de shell):

```bash
chgrp chrony /usr/local/samba/var/lib/ntp_signd
chmod 750 /usr/local/samba/var/lib/ntp_signd
```

Resultado esperado: `drwxr-x--- root chrony`. El socket que está dentro sigue como `root:root srwxrwxrwx`.

**Paso 2. Copia de respaldo de la configuración de chrony** (comando de shell):

```bash
cp -a /etc/chrony.conf /etc/chrony.conf.bak-YYYYMMDD
```

**Paso 3. Agregar al final de `/etc/chrony.conf`** las líneas de [configs/chrony.conf.example](../../configs/chrony.conf.example). Son **contenido de archivo**, no comandos: no las escriba en la terminal. Las líneas originales quedan intactas.

```
# Servidor de hora del dominio example.local
allow 192.0.2.0/24
allow 198.51.100.0/24
allow 203.0.113.0/24
ntpsigndsocket /usr/local/samba/var/lib/ntp_signd
```

**Paso 4. Reiniciar `chronyd`** (comando de shell):

```bash
systemctl restart chronyd
```

El registro pasa a mostrar `MS-SNTP authentication enabled` y `chronyd` empieza a escuchar en el UDP 123. Justo después del reinicio, `chronyc tracking` puede mostrar `Not synchronized` por uno o dos minutos: espere antes de probar cualquier cliente.

**Nota de red y seguridad:** en otros entornos, confirme que el UDP 123 esté abierto en el firewall del servidor y que SELinux (si está activo) permita que `chronyd` lea el directorio del socket. Mantenga el `allow` restringido a las redes que realmente tienen máquinas del dominio.

## 5. Validaciones en el servidor

- `chronyc tracking`: `Leap status: Normal`, estrato 2, diferencia de microsegundos a pocos milisegundos.
- **Reinicio del servicio `samba`:** el permiso del directorio `ntp_signd` se mantuvo (`root:chrony`), el socket se recreó y los puertos 53, 88, 389 y 445 volvieron a escuchar.
- **Reinicio del servidor:** `samba` y `chronyd` arrancaron solos, el permiso se mantuvo y chrony se sincronizó.
- En ambos casos, la estación de prueba se sincronizó con `Time Source Flags: 2 (Authenticated)` y `Last Sync Error: 0` (en su Windows, las etiquetas aparecen en el idioma del sistema).

Los reinicios afectan el dominio por unos segundos (servicio Samba) o minutos (servidor): inicio de sesión, DNS interno y GPO quedan no disponibles. Hágalo en un horario de poco movimiento.

## 6. Resultado en los clientes

- **WS-C** (estación de prueba): `w32tm /resync /rediscover` cambió la fuente a `dc01.example.local`. Para diferencias pequeñas, Windows corrige el reloj de forma **gradual** (slew), y no con un salto: la diferencia de 11.2 s bajó a cerca de 10 ms en aproximadamente 30 minutos.
- **WS-B** y otra estación (WS-D) se corrigieron **solas**, sin ningún comando, después de que el servidor empezó a responder de forma firmada.
- **WS-A** (la que estaba cerca de 128 s desfasada): el usuario informó que el reloj ya estaba correcto. **Medición técnica todavía pendiente.**

## 7. GPO "Time Sync - NT5DS"

Creada para uniformar la configuración de los clientes. Es la **segunda capa**: la primera es el servidor sano, y ninguna GPO lo sustituye.

Configuración del equipo > Directivas > Plantillas administrativas > Sistema > Servicio Hora de Windows > Proveedores de hora:

- **Habilitar el cliente NTP de Windows:** Habilitada.
- **Configurar el cliente NTP de Windows:** Habilitada, con `NtpServer = dc01.example.local,0x9` y `Type = NT5DS`. Los demás campos quedan en los valores predeterminados de la directiva (CrossSiteSyncFlags 2, ResolvePeerBackoffMinutes 15, ResolvePeerBackoffMaxTimes 7, SpecialPollInterval 1024, EventLogFlags 0).
- **Habilitar el servidor NTP de Windows:** No configurada (las estaciones no deben servir hora).

Dónde aparecen los valores en el registro de las estaciones: `HKLM\SOFTWARE\Policies\Microsoft\W32time\Parameters` (`NtpServer` y `Type`) y `HKLM\SOFTWARE\Policies\Microsoft\W32time\TimeProviders\NtpClient` (`Enabled` y los demás).

### Alcance y fases

Para limitar el riesgo, la GPO se aplicó por fases:

1. GPO creada **sin vínculo**, solo con las directivas configuradas.
2. Filtro de seguridad restringido a **una estación de prueba**, y solo entonces el vínculo en la raíz del dominio (no exigido).
3. Validación en la estación de prueba (`gpupdate /force`, `gpresult`, `w32tm /query /configuration`, `reg query`).
4. Filtro WMI `Workstations only (Windows Client)`, para no afectar a los servidores, y nueva validación.
5. Filtro de seguridad ampliado a `Domain Computers`, y validación en una segunda estación.

Como el vínculo está en la raíz, la herencia cubre todas las OU de computadoras. Las estaciones reciben la GPO en el ciclo normal de actualización (90 a 120 minutos), sin reiniciar.

## 8. Observaciones técnicas

- En modo `NT5DS`, el `SpecialPollInterval` **no se usa**. El intervalo de 1024 s sale de `MinPollInterval` y `MaxPollInterval`. Esos parámetros no se cambiaron: la directiva "Global Configuration Settings" define muchos valores a la vez y es riesgosa.
- Los valores `NtpServer` y `Type` de la directiva están en `...\W32time\Parameters`, y no en `...\TimeProviders\NtpClient`. Consultar la ruta equivocada da la impresión de que la GPO no llegó.
- Revisar solo el nombre de la GPO en `gpresult` no basta: aparece tanto en la lista de aplicadas como en la de filtradas. Confirme con los valores del registro después de un `gpupdate /force`.
- El servicio de hora de Windows viene con `StartType` `Manual` (inicio por disparador), que es el valor predeterminado.

## 9. Pendientes

1. **WS-A:** medir la diferencia y la fuente de hora con `w32tm /query /source` y `w32tm /stripchart /computer:dc01.example.local /samples:3 /dataonly`. Hoy solo existe el informe del usuario.
2. Revisar una estación de cada red de estaciones, en OU distintas, después del ciclo de actualización de las GPO.
3. Verificar si alguna OU de computadoras tiene bloqueada la herencia de GPO (ícono de exclamación en GPMC).
