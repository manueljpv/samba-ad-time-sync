# Cómo revertir

[← README](../../README.es-HN.md) · [Runbook](runbook.md) · [Lecciones aprendidas](lessons-learned.md)

Cada parte del proyecto se puede deshacer por separado. El orden recomendado es **primero la GPO y después el servidor**: así las estaciones nunca quedan apuntando a un servidor que dejó de responder de forma firmada.

> Datos ficticios: `dc01.example.local`, `example.local` y los rangos `192.0.2.0/24`, `198.51.100.0/24` y `203.0.113.0/24`.

## 1. GPO "Time Sync - NT5DS"

Elija el nivel de reversión, del más liviano al más completo:

| Qué hacer | Efecto |
|---|---|
| Quitar `Domain Computers` del filtro de seguridad | La GPO vuelve a valer solo para las máquinas que queden en el filtro (por ejemplo, la estación de prueba) |
| Desmarcar **Vínculo habilitado** en el vínculo de la raíz | La GPO deja de valer para todas, pero sigue existiendo |
| Eliminar el vínculo y, si quiere, la propia GPO | Eliminación definitiva |

En las estaciones, los valores de la directiva desaparecen en el siguiente ciclo de actualización (90 a 120 minutos) o con `gpupdate /force`, y pasa a valer la configuración **local**, que ya era `NT5DS`. No se cambió ninguna configuración local en las estaciones durante el proyecto: todo venía de la GPO.

Para confirmar en la estación:

```
gpupdate /force
reg query "HKLM\SOFTWARE\Policies\Microsoft\W32time\Parameters"   # los valores de directiva deben haber desaparecido
w32tm /query /configuration                                       # Type: NT5DS (Local)
```

## 2. Servidor (dc01)

> **Atención:** revertir el servidor devuelve el problema original. Las estaciones dejan de recibir hora firmada y vuelven a quedarse sin fuente de hora del dominio.

Comandos de shell, en el servidor:

```bash
cp -a /etc/chrony.conf.bak-YYYYMMDD /etc/chrony.conf    # restaura la copia hecha antes del cambio
systemctl restart chronyd
chgrp root /usr/local/samba/var/lib/ntp_signd           # opcional: devuelve el grupo original del directorio
```

Para confirmar:

```bash
grep -v -E '^\s*(#|$)' /etc/chrony.conf    # sin allow ni ntpsigndsocket
ss -ulpn | grep ':123'                     # chronyd deja de escuchar en el UDP 123
ls -ld /usr/local/samba/var/lib/ntp_signd  # drwxr-x--- root root
```

`smb.conf` y Samba no se modificaron, así que no hay nada que deshacer en ellos y el servicio Samba no necesita reiniciarse.

## 3. Sincronización manual en una estación

`w32tm /resync` no cambia ninguna configuración: solo pide una sincronización. No hay nada que revertir en ese paso.
