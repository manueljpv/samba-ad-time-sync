# Hoja de ruta

[Português](ROADMAP.md) · [English](ROADMAP.en.md) · **Español (Honduras)**

Mejoras previstas para el proyecto y lo que hemos aprendido hasta ahora. Son ideas, sin fechas garantizadas, y el orden no es definitivo.

## Corto plazo

Mejoras pequeñas, que caben en lo que ya existe.

- [ ] **Cerrar los pendientes de validación.** Medir WS-A con `w32tm /query /source` y `w32tm /stripchart` (hoy solo existe el informe del usuario), revisar una estación de cada red de estaciones en OU distintas después del ciclo de actualización de las GPO, y verificar si alguna OU de computadoras tiene bloqueada la herencia de GPO.
- [ ] **Script de verificación del servidor.** Reunir en un solo script los comandos de validación (chrony sincronizado, firma habilitada, UDP 123 escuchando, grupo y modo del directorio `ntp_signd`) y devolver "ok" o "falla" con el motivo. Solo entra al repositorio después de probarse como script.
- [ ] **Alerta de desfase.** Monitorear `chronyc tracking` (`Leap status` y desfase) y el permiso del directorio del socket, con aviso si algo cambia. Sin esto, un problema solo aparece cuando alguien nota la hora equivocada.
- [ ] **Tiempo de corrección.** Evaluar con cuidado los parámetros de sondeo de `w32time` (`MinPollInterval` y `MaxPollInterval`) para estaciones que tardan en corregirse. No se cambiaron porque la política que los define modifica muchos valores a la vez.
- [ ] **GPO por script.** Evaluar crear y configurar la GPO por script (el módulo `GroupPolicy` de PowerShell) contra un Samba AD, para que la implementación sea repetible. Necesita pruebas, porque no todas las funciones de GPO se comportan igual con Samba.
- [ ] **Verificación automática de la documentación.** Revisar enlaces y formato de los archivos Markdown en cada cambio (GitHub Actions).

## Largo plazo

Cambios mayores, que requieren un proyecto propio.

- [ ] **Redundancia del servidor de hora.** Hoy hay un único servidor de hora. Evaluar un segundo controlador con chrony, y una fuente alternativa para cuando se caiga el internet (por ejemplo, un reloj de referencia local), sopesando el riesgo de servir una hora sin referencia externa.
- [ ] **Equipos fuera del dominio.** Cámaras, impresoras y celulares quedaron fuera a propósito. Evaluar atenderlos de forma controlada, por red, con reglas de firewall y sin ampliar el `allow` más de lo necesario.
- [ ] **Historial y panel.** Registrar el desfase del servidor y de las estaciones a lo largo del tiempo (por ejemplo, con Prometheus y Grafana, o Zabbix) para ver tendencias y estaciones problemáticas.
- [ ] **Verificación masiva de la flota.** Un informe que liste la fuente y el desfase de cada estación sin tener que abrir máquina por máquina. El mecanismo aún está por definir (por ejemplo, un script de inicio por GPO que guarde el resultado en una carpeta compartida).
- [ ] **Implementación con Ansible.** Un playbook idempotente que aplique el permiso del directorio, el `chrony.conf` y el reinicio del servicio en uno o más controladores Samba AD.
- [ ] **Varios sitios.** Adaptar el diseño a entornos con más de un sitio y más de un controlador.

## Lo que aprendimos

- **El problema no estaba donde parecía.** El servidor tenía la hora correcta, y la causa era que no respondía hora firmada a los clientes. Medir antes de cambiar evitó "arreglar" lo que no era.
- **Lea el servidor antes de aplicar una receta.** En un Samba compilado desde el código fuente, las rutas cambian (`/usr/local/samba`, y no `/var/lib/samba`). Confirme la ruta del socket con `samba -b | grep -i NTP_SIGND_SOCKET_DIR`.
- **Separe lo que es comando de lo que es contenido de archivo.** Pegar en la terminal líneas que pertenecen al `chrony.conf` produce errores confusos. Cada bloque debe decir adónde va.
- **Un paso a la vez, con respaldo y con forma de deshacer.** Cada cambio fue pequeño, validado y reversible antes del siguiente.
- **Windows corrige la hora poco a poco.** Para diferencias pequeñas acelera o frena el reloj en lugar de dar un salto: 11 s tardaron cerca de 30 minutos. Mida de nuevo antes de concluir que falló.
- **Confirme la GPO por sus valores, no por su nombre.** `gpresult` muestra el nombre de la GPO incluso cuando fue filtrada. Los valores `NtpServer` y `Type` están en `...\W32time\Parameters`.
- **En modo NT5DS, `SpecialPollInterval` no se usa.** El intervalo de sondeo sale de otros parámetros.
- **Pruebe el reinicio antes de confiar.** Reiniciar el servicio Samba y el servidor completo mostró que el permiso del directorio del socket se mantiene.
- **Restrinja antes de vincular.** En la GPO, limite el alcance (filtro de seguridad en una estación) antes de crear el vínculo, y amplíelo por fases.
- **Un informe no es una medición.** Cuando alguien dice que "la hora ya está correcta", regístrelo como informe y mida después.
- **Más vale prevenir que lamentar.** Las estaciones se corrigieron solas cuando se arregló el servidor, pero la GPO queda como segunda capa para que nadie se quede sin fuente de hora.
