Entorno utilizado

- Sistema operativo: [Windows 11, versión 26H2]
- PHP: [8.5.7] (XAMPP)
- Laravel: [13.17]

Capturas

<img width="1917" height="630" alt="Captura de pantalla 2026-11-05 143902" src="https://github.com/user-attachments/assets/195060c2-4050-4cb0-869b-08ac78904922" />

<img width="1306" height="700" alt="image" src="https://github.com/user-attachments/assets/5cecfb08-56ed-4bc6-8a26-c2acd6766092" />


Respuestas

1. ¿Por qué la temperatura suele aparecer como "No disponible" en Windows?

Porque la temperatura se lee de la clase WMI `MSAcpi_ThermalZoneTemperature` (namespace `root/wmi`),
que tiene dos limitaciones:

- **Permisos:** solo se puede consultar con privilegios de administrador. Al ejecutar
  `php artisan serve` desde una terminal normal, la consulta falla, el `catch` del script la
  ignora y el valor queda en `null`, que la vista muestra como "No disponible".
- **Hardware:** aunque se ejecute como administrador, el dato depende de que el firmware
  (BIOS/ACPI) de la placa madre informe las zonas térmicas. Muchas placas y notebooks no lo hacen,
  y otras devuelven un valor fijo que no es la temperatura real de la CPU.


2. ¿Por qué el script de PowerShell nunca debe incluir datos que vengan del usuario?

Porque `shell_exec()` ejecuta comandos reales en el sistema operativo del servidor, con los mismos
permisos que el proceso de PHP. Si se armara el script con datos de un formulario o de la URL,
un atacante podría enviar algo como `; Remove-Item C:\ -Recurse` y ese texto se ejecutaría como
un comando más. Esto se llama **inyección de comandos** y permite tomar el control del servidor
(borrar archivos, robar datos, instalar programas).

Codificar el script en Base64 no protege contra esto: solo evita problemas con comillas y
caracteres especiales; el contenido se sigue ejecutando tal cual. Por eso el script del TP es
**fijo**: no tiene ninguna parte variable.

3. ¿Qué riesgo tiene dejar /api/metrics sin autenticación en un servidor real?

- **Exposición de información (reconocimiento):** cualquiera puede ver el nombre del equipo, el
  sistema operativo y la versión exacta de PHP. Con eso, un atacante busca vulnerabilidades
  conocidas para esas versiones. El uptime también indica si el servidor se reinició
  recientemente (por ejemplo, si se aplicaron actualizaciones).
- **Denegación de servicio:** cada pedido abre un proceso de PowerShell, que es costoso. Muchos
  pedidos simultáneos pueden saturar la CPU y la memoria del servidor. El `throttle:60,1` limita
  los pedidos por IP, pero no frena un ataque desde muchas IP distintas.

En producción se debería proteger con autenticación (por ejemplo `auth:sanctum`), restringir el
acceso a la red interna y guardar la lectura en caché unos segundos para no ejecutar PowerShell
en cada pedido.

4. Con polling cada 5 segundos, ¿cuántos pedidos por minuto recibe el servidor con 50 personas mirando?

Cada persona hace 60 / 5 = 12 pedidos por minuto.
Con 50 personas: 50 × 12 = 600 pedidos por minuto (10 por segundo).

Cada uno de esos pedidos ejecuta PowerShell, aunque los datos no hayan cambiado. Ese es el
problema que resuelven los WebSockets: el servidor envía los datos una sola vez por lectura y
todos los clientes conectados los reciben, sin pedidos repetidos.

5. Si una app solo necesita la temperatura, ¿qué datos recibe de más al llamar a /api/metrics?

Recibe todo lo demás: `hostname`, `os`, `php`, `disk` (`total_gb`, `free_gb`, `used_pct`),
`memory` (`total_mb`, `available_mb`, `used_pct`), `cpu_pct`, `uptime_s` y `at`.
De los 14 valores de la respuesta, solo usa 1.

Esto se llama **over-fetching**: el cliente descarga datos que no necesita, lo que desperdicia
ancho de banda y además expone información innecesariamente. Con REST, el endpoint siempre
devuelve la misma estructura completa. GraphQL resuelve este problema porque el cliente pide
exactamente los campos que quiere (por ejemplo, solo `cpu_temp_c`).
