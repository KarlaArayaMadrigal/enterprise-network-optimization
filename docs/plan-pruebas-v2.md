# Plan de pruebas — Red v2

> Proyecto de portafolio. Comercial Horizonte S.A. es una empresa ficticia y la red es
> una simulación en Cisco Packet Tracer.

Este plan valida la red v2 completa con las mismas pruebas de la v1 (fases A, B y C) y
con las pruebas nuevas de la decisión D7 (`CLAUDE.md`). Se ejecuta al terminar el
bloque 8 de `docs/plan-v2.md`, cuando IPv4 está completo. Las verificaciones de cada
bloque están en ese plan; aquí está la validación final. La columna "Obtenido" se
completa al ejecutar cada prueba.

## Antes de empezar

- Los bloques 1 a 8 de `docs/plan-v2.md` están terminados y verificados.
- Las cuatro PCs renovaron su IP después de los bloques 5 y 8.
- **Pérdida inicial:** igual que en la v1, el primer ping hacia un destino nuevo puede
  perder paquetes por ARP. Una prueba de éxito pasa si la repetición llega al 100 %.
- **Fallos esperados:** se marcan con ✖. Pasan si se cumplen las tres condiciones:
  1. **Ninguna respuesta desde la IP de destino**, en dos intentos. Un "Destination
     host unreachable" enviado por un router o un "Request timed out" cuentan como
     fallo; no se exige un texto concreto. Una sola respuesta del destino es un
     **defecto**.
  2. **Sube el contador de la línea deny final explícita de la ACL indicada.** Comparar
     `show access-lists` en el equipo indicado antes y después de la prueba. El deny
     implícito no tiene contador, por eso las tres ACLs de interfaz (gestión en R-HQ y
     en R-SUC, e Invitados en R-HQ) y la ACL de las VTY terminan con un deny explícito.
  3. **La prueba de control da éxito.** Es otra prueba de este plan que llega al mismo
     destino desde un origen permitido. Demuestra que el destino funciona y que el
     bloqueo viene de la ACL.
- Los pings desde las PCs se hacen en **Desktop > Command Prompt**.

## Fase A — DHCP

Igual que en la v1: cada pool excluye de .1 a .20. Además valida que DHCP snooping
(bloque 5) y la ACL de Invitados (bloque 8) dejan pasar el DHCP.

| # | PC | Puerto / VLAN | IP esperada | Gateway | DNS | IP obtenida | ¿OK? |
|---|---|---|---|---|---|---|---|
| A1 | PC-ADM | SW-HQ-2 Fa0/10 / VLAN 10 | 10.10.10.21-.254 | 10.10.10.1 | 10.10.30.10 | | |
| A2 | PC-VEN | SW-HQ-2 Fa0/11 / VLAN 20 | 10.10.20.21-.254 | 10.10.20.1 | 10.10.30.10 | | |
| A3 | PC-INV | SW-HQ-2 Fa0/12 / VLAN 40 | 10.10.40.21-.254 | 10.10.40.1 | 10.10.30.10 | | |
| A4 | PC-SUC | SW-SUC Fa0/10 / VLAN 50 | 10.20.50.21-.254 | 10.20.50.1 | 10.10.30.10 | | |
| A5 | PC-TI | SW-HQ-1 Fa0/11 / VLAN 30 | 10.10.30.11 (fija) | 10.10.30.1 | 10.10.30.10 | | |

En las cuatro PCs con DHCP, liberar y renovar la IP antes de anotar la que obtienen. La
A5 solo confirma la configuración estática de PC-TI.

**Comprobación en los equipos:**
- `show ip dhcp binding` en R-HQ (tres asignaciones) y en R-SUC (una).
- `show ip dhcp snooping binding` en SW-HQ-2 y SW-SUC, si Packet Tracer lo acepta.

## Fase B — Conectividad

`IP-ADM`, `IP-VEN`, `IP-INV` e `IP-SUC` son las direcciones obtenidas en la fase A. En la
columna "Contador / control", el contador es siempre el de la línea deny final de la
ACL indicada.

### B1. Gateway local

| # | Origen | Destino | Esperado | Obtenido |
|---|---|---|---|---|
| B1.1 | PC-ADM | 10.10.10.1 | Éxito, TTL=255 | |
| B1.2 | PC-VEN | 10.10.20.1 | Éxito, TTL=255 | |
| B1.3 | PC-INV | 10.10.40.1 | Éxito, TTL=255. La ACL de Invitados permite el eco hacia su gateway | |
| B1.4 | PC-SUC | 10.20.50.1 | Éxito, TTL=255 | |

### B2. Entre VLANs en la sede central

| # | Origen | Destino | Esperado | Contador / control | Obtenido |
|---|---|---|---|---|---|
| B2.1 | PC-ADM | IP-VEN | Éxito, TTL=127 | — | |
| B2.2 | PC-VEN | IP-INV | ✖ Falla: el pedido llega a PC-INV, pero la respuesta se bloquea al entrar por la .40 | ACL de Invitados (R-HQ) / B1.3 | |
| B2.3 | PC-ADM | 10.10.30.10 | Éxito, TTL=127 | — | |
| B2.4 | PC-VEN | 10.10.30.10 | Éxito, TTL=127 | — | |
| B2.5 | PC-INV | 10.10.30.10 | ✖ Falla | ACL de Invitados (R-HQ) / B2.3 | |
| B2.6 | PC-INV | IP-ADM | ✖ Falla | ACL de Invitados (R-HQ) / B3.4 | |

### B3. Entre sedes

| # | Origen | Destino | Esperado | Contador / control | Obtenido |
|---|---|---|---|---|---|
| B3.1 | PC-SUC | 10.255.255.2 | Éxito, TTL=255 | — | |
| B3.2 | PC-SUC | 10.255.255.1 | Éxito, TTL=254 | — | |
| B3.3 | PC-SUC | 10.10.30.10 | Éxito, TTL=126 | — | |
| B3.4 | PC-SUC | IP-ADM | Éxito, TTL=126 | — | |
| B3.5 | PC-ADM | IP-SUC | Éxito, TTL=126 | — | |
| B3.6 | PC-VEN | IP-SUC | Éxito, TTL=126 | — | |
| B3.7 | PC-INV | IP-SUC | ✖ Falla | ACL de Invitados (R-HQ) / B3.6 | |

### B4. Red de gestión

| # | Origen | Destino | Esperado | Contador / control | Obtenido |
|---|---|---|---|---|---|
| B4.1 | R-HQ | 10.10.99.11 y 10.10.99.12 | Éxito. El tráfico del propio router no pasa por la ACL de salida | — | |
| B4.2 | R-SUC | 10.20.99.11 | Éxito, por el mismo motivo | — | |
| B4.3 | PC-ADM | 10.20.99.11 | ✖ Falla | ACL de gestión (R-SUC) / D1.3 | |
| B4.4 | PC-SUC | 10.10.99.11 | ✖ Falla | ACL de gestión (R-HQ) / D1.1 | |
| B4.5 | PC-INV | 10.10.99.12 | ✖ Falla. La bloquea primero la ACL de Invitados, antes de llegar a la de gestión | ACL de Invitados (R-HQ) / D1.2 | |
| B4.6 | R-SUC | 10.10.99.11 | Éxito. **Nueva:** confirma que el origen WAN está permitido (D4) | — | |

## Fase C — Servicios DNS y web en SRV-TI

Antes de empezar: captura de la pestaña Services de SRV-TI, con DNS, HTTP, NTP y Syslog
activos, HTTPS apagado y sin archivos de ejemplo.

| # | Origen | Qué hacer | Esperado | Obtenido |
|---|---|---|---|---|
| C1.1 | PC-ADM | Web Browser → `http://10.10.30.10` | Carga la página de SRV-TI | |
| C2.1 | PC-ADM | `nslookup www.horizonte.internal` | Servidor y dirección 10.10.30.10 | |
| C2.2 | PC-VEN | `nslookup srv-ti.horizonte.internal` | Servidor y dirección 10.10.30.10 | |
| C2.3 | PC-SUC | `nslookup www.horizonte.internal` | Servidor y dirección 10.10.30.10 | |
| C2.4 | PC-ADM | `nslookup noexiste.horizonte.internal` | `Non-existent domain` | |
| C3.1 | PC-ADM | `ping srv-ti.horizonte.internal` | `Pinging 10.10.30.10`, 4/4, TTL=127 | |
| C3.2 | PC-SUC | `ping srv-ti.horizonte.internal` | `Pinging 10.10.30.10`, 4/4, TTL=126 | |
| C4.1 | PC-ADM | Web Browser → `http://www.horizonte.internal` | La misma página que en C1.1 | |
| C4.2 | PC-VEN | Ídem | La misma página | |
| C4.3 | PC-SUC | Ídem | La misma página | |
| C4.4 | PC-INV | Ídem | ✖ No carga. El nombre se resuelve (D3.2), pero el HTTP se bloquea. Contador: ACL de Invitados (R-HQ). Control: C4.1 | |

## Fase D — Pruebas nuevas

### D1. Administración desde PC-TI (D7)

| # | Origen | Qué hacer | Esperado | Obtenido |
|---|---|---|---|---|
| D1.1 | PC-TI | Ping a 10.10.99.11 | Éxito, TTL=254 | |
| D1.2 | PC-TI | Ping a 10.10.99.12 | Éxito, TTL=254 | |
| D1.3 | PC-TI | Ping a 10.20.99.11 | Éxito, TTL=253 | |
| D1.4 | PC-TI | SSH con el usuario a 10.10.99.11 (SW-HQ-1) | Pide contraseña, entra y `enable` acepta el secret | |
| D1.5 | PC-TI | SSH a 10.10.99.12 (SW-HQ-2) | Ídem | |
| D1.6 | PC-TI | SSH a 10.20.99.11 (SW-SUC) | Ídem | |
| D1.7 | PC-TI | SSH a 10.10.30.1 (R-HQ) | Ídem | |
| D1.8 | PC-TI | SSH a 10.20.50.1 (R-SUC) | Ídem | |

### D2. Telnet desactivado (D7)

Se prueba desde PC-TI porque es el único origen permitido por las ACLs y las
`access-class`. Así, el rechazo solo puede deberse a que Telnet está desactivado.

| # | Origen | Qué hacer | Esperado | Obtenido |
|---|---|---|---|---|
| D2.1 | PC-TI | Telnet a 10.10.99.11 | ✖ Rechazo inmediato, no un timeout. El texto exacto se anota al ejecutarla | |
| D2.2 | PC-TI | Telnet a 10.10.99.12 | ✖ Ídem | |
| D2.3 | PC-TI | Telnet a 10.20.99.11 | ✖ Ídem | |
| D2.4 | PC-TI | Telnet a 10.10.30.1 | ✖ Ídem | |
| D2.5 | PC-TI | Telnet a 10.20.50.1 | ✖ Ídem | |

### D3. Acceso no autorizado e Invitados (D7)

| # | Origen | Qué hacer | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| D3.1 | PC-ADM | SSH a 10.10.10.1 (su propio gateway) | `access-class` de las VTY: en ese camino no hay ACL de interfaz | ✖ Rechazado por el router. Contador: ACL de las VTY (R-HQ). Control: D1.7 | |
| D3.2 | PC-INV | `nslookup www.horizonte.internal` | Invitados resuelve DNS (D5) | Servidor y dirección 10.10.30.10 | |
| D3.3 | PC-INV | SSH a 10.10.40.1 | La ACL de Invitados bloquea la gestión del propio gateway | ✖ Falla. Contador: ACL de Invitados (R-HQ). Control: D1.7 | |

El resto de la D7 ya está cubierto por otras pruebas: "Invitados obtiene IP por DHCP" es
la A3 y "no carga la web interna" es la C4.4.

En la D3.1, el contador es el del deny final de la ACL estándar de las VTY (bloque 7 de
`docs/plan-v2.md`). Por comprobar en Packet Tracer: si el contador de una ACL usada como
`access-class` sube en `show access-lists`. Si no sube, se documenta como limitación del
simulador y la D3.1 se valida con las condiciones 1 y 3.

### D4. Protecciones de capa 2

Demuestran en funcionamiento SW-03 y SW-04, que de otro modo solo se verían con comandos
`show`.

| # | Qué hacer | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|
| D4.1 | Desconectar PC-INV de SW-HQ-2 Fa0/12 y conectar en ese puerto otra PC; hacer un ping desde ella | Port-security (SW-03) | El puerto queda en err-disabled y `show port-security interface Fa0/12` muestra 1 violación. Después, volver a conectar PC-INV, rehabilitar el puerto y repetir la B1.3 | |
| D4.2 | Ping continuo de PC-ADM a SRV-TI y apagar Fa0/1 en SW-HQ-1 | Redundancia del EtherChannel (SW-04) | El ping sigue, con como mucho una pérdida, y `show etherchannel summary` muestra Po1 en uso con un solo puerto. Después, volver a encender Fa0/1 y comprobar que vuelve a agruparse | |

## Fase E — Evidencia de configuración por hallazgo

Salidas de los comandos `show` que demuestran que cada hallazgo de
`docs/auditoria-v1.md` quedó corregido. Son la comparación del antes y el después para
el README.

| Hallazgo | Comando o prueba en la v2 | Resultado esperado |
|---|---|---|
| SEC-01 | `show ip ssh` y fases D1 y D2 | SSH 2.0 y Telnet rechazado |
| SEC-02 | `show running-config` en los 5 equipos | Sin `enable password`; `enable secret` con hash, distinto en cada equipo |
| SEC-03 | `show running-config` | Usuario local con hash, cifrado de contraseñas activo, VTY con usuarios locales |
| SEC-04 | Salir y volver a entrar a la consola; `show running-config` | Pide usuario; inactividad de 10 minutos en la consola y las VTY |
| SEC-05 | Entrar a cualquier equipo | Aparece el banner |
| SEG-01 | B2.2, B2.5, B2.6, B3.7, C4.4, D3.3 y `show access-lists` en R-HQ | Fallan y sube el contador del deny final de la ACL de Invitados |
| SEG-02 | B4.3 a B4.5, D3.1 y `show access-lists` en los dos routers | Fallan y sube el contador del deny final de cada ACL de gestión y de la ACL de las VTY (D3.1); D1 da éxito |
| SW-01 | `show vlan brief` y `show ip interface brief` en los 3 switches | Puertos sin uso en la VLAN 998 y en administratively down |
| SW-02 | `show interfaces trunk` y `show interfaces <trunk> switchport` | Nativa 999, VLANs permitidas de la D14, negociación desactivada |
| SW-03 | `show spanning-tree` (puertos "P2p Edge"), `show port-security` y D4.1 | PortFast, BPDU Guard y port-security en todos los puertos de acceso; violación demostrada |
| SW-04 | `show spanning-tree`, `show etherchannel summary` y D4.2 | rstp, SW-HQ-1 raíz por prioridad, Po1 (SU) con LACP, ningún puerto en BLK; redundancia demostrada |
| SW-05 | `show ip dhcp snooping` | Activo en las VLANs de la D19, opción 82 desactivada, solo el puerto hacia el router como confiable |
| SW-06 | `show cdp interface` en SW-HQ-2 Fa0/12 | El puerto no aparece |
| RT-01 | `show ip protocols` | Pasivas por defecto, salvo G0/0/1 |
| RT-02 | `show ip ospf interface G0/0/1` y `show ip ospf neighbor` | Autenticación message-digest y vecino en FULL |
| DIS-01 | Fase F (bloque 9) | Pendiente |
| OP-01 | `show ntp status`, `show logging` y la pestaña SYSLOG de SRV-TI | Sincronizado con 10.10.30.10, marcas de tiempo y mensajes de los 5 equipos |
| OP-02 | `show interfaces description` | Descripciones en todas las interfaces en uso |
| OP-03 | `show running-config` en SW-HQ-1 | Búsqueda de nombres desactivada |
| OP-04 | Captura de la pestaña Services de SRV-TI | HTTPS apagado y sin archivos de ejemplo |
| INFO-01 | `show line` y `show running-config` | Todas las líneas VTY con la misma configuración |

## Fase F — IPv6 (bloque 9)

Se define cuando se detalle el bloque 9 de `docs/plan-v2.md`. Repetirá en IPv6 las
pruebas de conectividad de la fase B (incluidos los fallos esperados de Invitados) y el
acceso a las VTY.

## Cómo diagnosticar un fallo inesperado en la v2

Además de la tabla de diagnóstico de `docs/plan-pruebas-v1.md`:

| Síntoma | Dónde revisar primero |
|---|---|
| Una PC no obtiene IP (fase A) | `show ip dhcp snooping`: el puerto hacia el router tiene que ser confiable y la opción 82 tiene que estar desactivada. En PC-INV, revisar además la regla de DHCP de la ACL de Invitados |
| Una PC sin conectividad en su VLAN | `show port-security` y `show ip interface brief`: el puerto puede estar en err-disabled |
| Se corta una VLAN entre SW-HQ-1 y SW-HQ-2 | `show interfaces trunk` y `show etherchannel summary`: la nativa y las VLANs permitidas tienen que coincidir en los dos lados y en Po1 |
| Falla una prueba de éxito y sube un contador de una ACL | La ACL bloquea de más: revisar el orden de las líneas y la dirección en la que está aplicada |
| Un fallo esperado da éxito | La ACL no está aplicada o está en la dirección equivocada: `show ip interface` en la subinterfaz |
| Un fallo esperado falla, pero el contador del deny final no sube | El bloqueo viene de otro lado (otra línea de la ACL, la ACL equivocada o un problema de red): revisar la prueba de control |
| SSH desde PC-TI falla | `show ip ssh`, la `access-class` de las VTY y que todas las líneas VTY estén configuradas igual |
| Falla la B3 | `show ip ospf neighbor`: la autenticación tiene que coincidir en los dos routers y G0/0/1 no puede quedar pasiva |

## Evidencia

- Salidas de texto en `tests/`, con el prefijo `v2-`, siguiendo los nombres de la v1:
  - `v2-dhcp-pcs.txt` (fase A)
  - `v2-ping.txt` (fase B)
  - `v2-dns-web.txt` (fase C)
  - `v2-acceso.txt` (fase D)
  - `v2-config-show.txt` (fase E)
- Capturas en `evidence/` con el prefijo `v2-`, por ejemplo
  `v2-c4-web-por-nombre-pc-inv.png`.
