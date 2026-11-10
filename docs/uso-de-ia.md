### 2026-11-09 — Revisión de R-SUC contra el diseño (v1)
- Qué pidió Karla: Comparar `configurations/v1/R-SUC.txt` con el diseño de CLAUDE.md y decir qué le falta a R-SUC para completar la v1, solo con instrucciones y sin comandos.
- Qué propuso Claude: La configuración exportada coincide con el diseño (subinterfaces, IPs, DHCP y OSPF). El archivo ya incluye contraseñas de enable y VTY, igual que R-HQ, y ya está exportado. Quedan por confirmar el guardado en startup-config y por probar el DHCP y la conectividad LAN de la sucursal. También señaló la diferencia de versión de IOS con R-HQ (15.4 frente a 16.6.4) y la falta de contraseña de consola, común a R-HQ y SW-SUC.
- Qué verificó Karla y cómo: Comparé la configuración exportada con el archivo CLAUDE.md
- Qué se corrigió o se rechazó, y por qué: Se realizó la exportación del archivo con la configuración

### 2026-11-09 — Falso positivo en la versión de IOS de R-SUC
- Qué pidió Karla: Informó que, según `show version`, R-HQ y R-SUC son ISR4331 con el mismo IOS XE 03.16.05.S / 15.5(3)S5, así que la diferencia de versión señalada en la entrada anterior era un falso positivo. Pidió una explicación del error, este registro y marcar R-SUC como completo en CLAUDE.md.
- Qué propuso Claude: Explicó que la conclusión salió de comparar la línea `version` de las configuraciones exportadas (15.4 en R-SUC y 16.6.4 en R-HQ). Esa línea no sirve para saber qué IOS ejecuta un equipo: ninguno de los dos valores coincide con lo que muestra `show version`. Para identificar modelo y versión hay que usar la salida de `show version`, no el archivo de configuración. Actualizó CLAUDE.md para dejar R-SUC en la lista de equipos listos.
- Qué verificó Karla y cómo: Utilizando el comando show version
- Qué se corrigió o se rechazó, y por qué: Las versiones son las mismas, fue un error de lectura.

### 2026-11-09 — IP fija de SRV-TI desde la interfaz gráfica
- Qué pidió Karla: Instrucciones paso a paso para configurar la IP fija de SRV-TI desde la interfaz gráfica de Packet Tracer, según CLAUDE.md.
- Qué propuso Claude: Revisó antes que SW-HQ-1 Fa0/10 estuviera en la VLAN 30 y que R-HQ tuviera la subinterfaz 10.10.30.1. Propuso configurar desde Desktop > IP Configuration en modo Static: IP 10.10.30.10, máscara 255.255.255.0, gateway 10.10.30.1 y DNS 10.10.30.10. El DNS del propio servidor no está definido en el diseño y se propuso como sugerencia. También propuso pruebas de ping hacia el gateway, desde R-HQ y desde R-SUC. Los servicios DNS y web quedaron fuera de esta tarea.
- Qué verificó Karla y cómo: Revisé la configuración del servidor manualmente y verifiqué la conectividad de los dos routers hacia el default gateway
- Qué se corrigió o se rechazó, y por qué: Realicé la configuración del servidor y utilicé el comando ping para verificar la conexion 

### 2026-11-09 — DHCP en las PCs y plan de pruebas de la v1
- Qué pidió Karla: Instrucciones para que PC-ADM, PC-VEN, PC-INV y PC-SUC obtengan IP por DHCP en Packet Tracer, y un plan de pruebas de conectividad para validar toda la v1, con IPs esperadas, pings entre VLANs y entre sedes, y resultados esperados.
- Qué propuso Claude: Revisó que los puertos de SW-HQ-2 y SW-SUC coincidieran con el diseño. Dio los pasos para pasar cada PC a DHCP desde Desktop > IP Configuration. Creó `docs/plan-pruebas-v1.md` con una fase A de DHCP (rango .21–.254 por pool, gateway y DNS esperados) y una fase B de pings en cuatro grupos: gateway local, entre VLANs de la central, entre sedes y gestión. Marcó como hallazgos para la auditoría las pruebas que en la v1 tendrán éxito por falta de ACLs (invitados hacia redes internas y acceso a gestión desde VLANs de usuarios). Dejó fuera las pruebas de DNS por nombre y del servicio web, porque esos servicios están pendientes.
- Qué verificó Karla y cómo: Revisé la configuración de cada pc desde Desktop
- Qué se corrigió o se rechazó, y por qué: Configuré cada pc manualmente para que recibiera una dirección IP mediante DHCP

### 2026-11-09 — Verificación de DHCP (fase A del plan de pruebas)
- Qué pidió Karla: Revisar contra el diseño las asignaciones DHCP de R-HQ y R-SUC y la configuración IP de las cuatro PCs, y guardar la evidencia en `tests/dhcp-pcs.txt`.
- Qué propuso Claude: Comparó cada salida con el diseño. Las cuatro PCs recibieron la .21 de su red, con la máscara, el gateway, el DNS (10.10.30.10) y el dominio (horizonte.internal) correctos, y sus MAC coinciden con las asignaciones de los routers. Identificó por la MAC una salida que llegó sin nombre (PC-SUC). Guardó las salidas y un resumen en `tests/dhcp-pcs.txt`. Karla confirmó que las salidas de las PCs son de `ipconfig /all`. Claude completó después la tabla de la fase A en `docs/plan-pruebas-v1.md`, a pedido de Karla.
- Qué verificó Karla y cómo: Utilicé los comandos show ip dhcp binding` en R-HQ y R-SUC y ipconfig /all en las cuatro PCs. Comparé manualmente cada MAC de las tablas de los routers con la Physical Address de cada PC: las cuatro coinciden. Confirmé que la salida sin nombre corresponde a PC-SUC (MAC 0030.F2CE.822A, IP 10.20.50.21). Comprobé que todas recibieron la .21, lo que demuestra que las exclusiones de .1 a .20 funcionan.
- Qué se corrigió o se rechazó, y por qué: No hubo errores en el análisis de Claude. Aprobé que cambiara los títulos de las secciones en `tests/dhcp-pcs.txt`

### 2026-11-09 — Pruebas B1: gateway local
- Qué pidió Karla: Revisar las salidas de ping de cada PC hacia su gateway, guardarlas en `tests/ping-v1.txt` y marcar la B1 en el plan de pruebas.
- Qué propuso Claude: Confirmó que las cuatro pruebas tuvieron éxito (4/4, 0 % de pérdida). Explicó que el TTL=255 indica que respondió directamente el router, sin saltos intermedios. Guardó las salidas en `tests/ping-v1.txt` y marcó la B1 en `docs/plan-pruebas-v1.md`. Dejó listas las pruebas B2 con las IPs reales de las PCs, y anticipó un TTL de 127 y la posible pérdida del primer paquete.
- Qué verificó Karla y cómo: Desde el Command Prompt de cada PC hice ping a su gateway (PC-ADM → 10.10.10.1, PC-VEN → 10.10.20.1, PC-INV → 10.10.40.1, PC-SUC → 10.20.50.1). Las cuatro respondieron 4/4 sin pérdida. Revisé en las salidas que el TTL fuera 255, como explicó Claude, lo que confirma que respondió directamente el router.
- Qué se corrigió o se rechazó, y por qué: Nada; el análisis de Claude coincidió con las salidas.

### 2026-11-09 — README inicial y .gitignore
- Qué pidió Karla: Un README.md inicial que diga que es un proyecto personal de portafolio, que la empresa es ficticia, que la red es una simulación en Packet Tracer, que está en construcción con las fases 1 y 2 terminadas, y que las contraseñas de `configurations/v1/` son credenciales de laboratorio débiles a propósito que se corregirán en la v2. También un .gitignore que excluya `.claude/settings.local.json`.
- Qué propuso Claude: Antes de escribir, señaló que según CLAUDE.md y la evidencia de `tests/` la fase 2 sigue en curso (faltan las pruebas B2, B3 y B4 y los servicios DNS y web de SRV-TI), y preguntó cómo presentarla. Karla eligió "fase 2 en curso". Creó el README con el aviso de proyecto ficticio, el objetivo v1/v2, las tecnologías, el estado por fase (con lo ya verificado y lo pendiente), el aviso sobre credenciales, la estructura de carpetas y la sección de uso de IA. Creó el .gitignore con `.claude/settings.local.json`.
- Qué verificó Karla y cómo: Leí el README completo antes de publicarlo. Confirmé que el aviso de proyecto ficticio y simulado aparece al inicio, que el estado de las fases coincide con la evidencia de tests/ y que el aviso sobre las credenciales de laboratorio es claro. Revisé que no aparecieran mi correo ni otros datos personales.
- Qué se corrigió o se rechazó, y por qué: Mi pedido decía que la fase 2 estaba terminada, pero Claude detectó que faltaban las pruebas B2 a B4 y los servicios de SRV-TI, y preguntó antes de escribir. Elegí "fase 2 en curso" para que el README refleje el estado real. Decidí mantener mi nombre en el README porque es un proyecto de portafolio.
