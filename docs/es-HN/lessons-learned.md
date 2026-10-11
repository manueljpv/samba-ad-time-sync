# Lecciones aprendidas

[← README](../../README.es-HN.md) · [Runbook](runbook.md) · [Revertir](rollback.md)

El resumen de las lecciones está en [ROADMAP.es-HN.md](../../ROADMAP.es-HN.md). Aquí queda el camino de la investigación, lo que no salió como se esperaba y lo que haríamos distinto.

## El camino de la investigación

1. **Primero el servidor.** Hora y zona horaria, qué servicio de hora está corriendo, fuentes y desfase (`chronyc`), versión y rutas de Samba, `smb.conf` y permisos del socket. Resultado: el servidor estaba bien, solo no atendía a los clientes.
2. **Red.** `w32tm /stripchart` desde una estación hasta el servidor: ¿responde? ¿cuál es el desfase? Esto muestra si el UDP 123 pasa y cuantifica el problema.
3. **Cliente.** `w32tm /query /source` y `/status` (fuente y última sincronización), `w32tm /query /configuration` (tipo y fuentes) y `nltest /dsgetdc:... /TIMESERV` (¿el dominio anuncia un servidor de hora?). Resultado: configuración correcta, pero sin sincronizar.
4. **Hipótesis y prueba mínima.** "Los clientes simplemente no volvieron a intentar después de que el servidor empezó a responder firmado." Confirmado con `w32tm /resync /rediscover` en **una** estación.
5. **Solo entonces la GPO**, por fases.

## Lo que no salió como se esperaba

- **Ruta genérica, instalación específica.** La receta inicial usaba `/var/lib/samba/ntp_signd`. En el Samba compilado desde el código fuente la ruta era `/usr/local/samba/var/lib/ntp_signd`. La investigación del servidor lo descubrió antes de cualquier cambio.
- **Comandos y contenido de archivo en el mismo bloque.** Las líneas `allow` y `ntpsigndsocket` se pegaron en la terminal y dieron `command not found`. Sin efecto secundario, pero mostró que cada bloque debe decir si es un comando o contenido de archivo.
- **`SpecialPollInterval` para acelerar la corrección.** La idea inicial era reducirlo, pero en modo `NT5DS` no se usa. Se descartó antes de aplicarla.
- **Esperar un salto de hora.** Después del `resync`, la diferencia siguió casi igual. Era la corrección gradual de Windows: solo medir de nuevo después de unos 30 minutos mostró el resultado.
- **Consulta en la ruta equivocada del registro.** Dos valores de la GPO (`NtpServer` y `Type`) están en `...\W32time\Parameters`, y no en `...\TimeProviders\NtpClient`.
- **`gpresult` ambiguo.** El nombre de la GPO aparece también en la lista de filtradas. Pasamos a confirmar con los valores del registro.

## Decisiones de riesgo que valieron la pena

- Restringir el alcance (filtro de seguridad en una estación) **antes** de vincular la GPO.
- Reiniciar el servicio Samba y el servidor para ver si el permiso del directorio del socket se mantiene.
- Hacer respaldo antes de editar y tener un comando de reversión para cada cambio.
- Medir antes y después con los mismos comandos.

## Lo que haríamos distinto

- Registrar las mediciones (valor y hora) en una tabla desde el inicio.
- Medir también la estación de la que alguien dijo "ya está correcta", antes de dar el asunto por cerrado.
- Preparar temprano un script de verificación (está en la [hoja de ruta](../../ROADMAP.es-HN.md)).
