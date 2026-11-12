# Plan de aplicación de la red v2

> Proyecto de portafolio. Comercial Horizonte S.A. es una empresa ficticia y la red es
> una simulación en Cisco Packet Tracer.

Plan de aplicación de la red v2, en el orden de la decisión D28 (`CLAUDE.md`). Karla
escribe los comandos; este plan da las instrucciones, los riesgos, la verificación y la
forma de deshacer cada bloque. Las pruebas completas de la v2 (fases A, B y C y pruebas
nuevas de la D7) están en `docs/plan-pruebas-v2.md`.

## Reglas para todos los bloques

- Se trabaja solo en `diagrams/horizonte-v2.pkt`. La v1 no se toca.
- **Antes de cada bloque:** guardar el `.pkt` y hacer commit. Es el punto de
  restauración más seguro.
- **Durante el bloque:** no guardar la running-config en la startup-config hasta
  verificar. Si un equipo queda mal, reiniciarlo sin guardar lo devuelve a su último
  estado guardado. Excepción: las VLANs creadas en un switch quedan en su base de VLANs
  aunque se reinicie, pero eso no afecta a la red.
- **Al terminar cada bloque:** verificar, guardar la configuración en cada equipo
  modificado, guardar el `.pkt` y guardar las salidas en `tests/v2-bloqueN-<tema>.txt`.
- Todo se configura desde la consola (pestaña CLI), así que perder el acceso remoto
  nunca deja un equipo inaccesible. El único riesgo real de quedarse fuera es la
  contraseña de la consola, en el bloque 7.
- **TTL esperados al responder:** una PC responde con 128 y un router o un switch con
  255. Cada router en el camino resta 1.

---

## Bloque 1 — PC-TI

**Equipos y orden:** SW-HQ-1, luego PC-TI.

**Qué configurar**
1. En SW-HQ-1, asignar Fa0/11 a la VLAN 30 en modo acceso, igual que Fa0/10 (SRV-TI).
   Las protecciones del puerto van en el bloque 3.
2. Agregar una PC-PT llamada PC-TI y conectarla con cable directo de cobre: Fa0 de
   PC-TI a Fa0/11 de SW-HQ-1.
3. En PC-TI, IP estática: 10.10.30.11/24, gateway 10.10.30.1, DNS 10.10.30.10.

**Riesgo:** ninguno para lo que ya funciona. Si Fa0/11 queda en la VLAN 1, PC-TI no
llega a su gateway.

**Verificación**

| Prueba | Resultado esperado |
|---|---|
| `show vlan brief` en SW-HQ-1 | Fa0/10 y Fa0/11 en la VLAN 30 |
| PC-TI → 10.10.30.1 | Éxito, TTL=255 |
| PC-TI → 10.10.30.10 (SRV-TI, misma VLAN) | Éxito, TTL=128 |
| PC-TI → IP de PC-ADM | Éxito, TTL=127 |
| PC-TI → 10.10.99.11 y 10.10.99.12 | Éxito, TTL=254 |
| PC-TI → 10.20.99.11 | Éxito, TTL=253 |
| PC-TI: `nslookup www.horizonte.internal` | Dirección 10.10.30.10 |
| PC-TI: Telnet a 10.10.99.11 | Se conecta. Es la evidencia del "antes" para el bloque 7 |

**Cómo deshacer:** borrar PC-TI y devolver Fa0/11 a su configuración por defecto.

---

## Bloque 2 — Configuración base y operación (D12, D24, D25, D26 y D27)

**Equipos y orden:** SRV-TI, R-HQ, R-SUC, SW-HQ-1, SW-HQ-2 y SW-SUC. SRV-TI va primero
porque NTP y syslog dependen de él.

**Qué configurar**
1. **SRV-TI (pestaña Services):**
   - Activar NTP y Syslog.
   - Comprobar que DNS y HTTP siguen activos.
   - Apagar HTTPS y cualquier otro servicio activo que no esté en la D27 (revisar cada
     pestaña).
   - Borrar los archivos de ejemplo (`helloworld.html`, `image.html`, `copyrights.html`
     y el logo) sin tocar `index.html`.
2. **En los cinco equipos IOS:**
   - Marcas de tiempo con fecha, hora y milisegundos en los logs y en el debug (hoy
     están desactivadas: es el primer par de líneas de la configuración).
   - Servidor NTP 10.10.30.10. No hace falta configurar zona horaria: UTC es la que
     traen por defecto.
   - Envío de logs a 10.10.30.10 con nivel informativo.
   - Banner de acceso restringido (D12), sin la empresa, los modelos ni las IPs.
3. **Solo SW-HQ-1:** desactivar la búsqueda de nombres (OP-03). Los demás ya la tienen
   desactivada.
4. **Descripciones (D25):** en cada interfaz en uso, con el formato
   `<tipo> a <equipo> <puerto>`, según `conexiones.md`. Incluye las subinterfaces (por
   ejemplo, "VLAN 10 ADMINISTRACION") y la Fa0/11 de PC-TI. Los puertos sin uso se
   describen en el bloque 3.

**Riesgo:** bajo. Lo único que puede romper algo es apagar DNS o HTTP por error en
SRV-TI, o borrar `index.html`. Por comprobar: si el 2960 de Packet Tracer no acepta NTP,
anotarlo y seguir. No es un bloqueo.

**Verificación**

| Prueba | Resultado esperado |
|---|---|
| `show ntp status` y `show ntp associations` en cada equipo | Sincronizado con 10.10.30.10. Puede tardar uno o dos minutos |
| `show clock` | La hora de SRV-TI |
| `show logging` | Logging hacia 10.10.30.10 y marcas de tiempo activas |
| Entrar y salir del modo de configuración en cada equipo y mirar SRV-TI > Services > SYSLOG | Un mensaje de cada equipo, con hora |
| `show interfaces description` (y `show ip interface brief` en los routers) | Descripciones en todas las interfaces en uso |
| `show running-config` en SW-HQ-1 | Aparece la búsqueda de nombres desactivada |
| Repetir C1.1, C2.1 y C4.1 | Éxito: DNS y HTTP siguen activos |
| Captura de la pestaña de HTTP de SRV-TI | HTTPS apagado y sin archivos de ejemplo. Evidencia de OP-04 |

**Cómo deshacer:** cada línea se quita con su forma "no", y los servicios de SRV-TI se
reactivan desde Services. Los archivos de ejemplo borrados solo se recuperan desde el
`.pkt` guardado (no hacen falta).

---

## Bloque 3 — Capa 2: VLANs, trunks y puertos (D13, D14, D17, D18 y D20)

**Equipos y orden:**
1. Crear las VLANs 998 y 999 en los tres switches.
2. Trunk de SW-SUC hacia R-SUC.
3. Trunk de SW-HQ-1 hacia R-HQ.
4. Trunks entre SW-HQ-1 y SW-HQ-2 (los dos extremos seguidos).
5. Puertos de acceso.
6. Puertos sin uso, al final.

**Qué configurar**
1. **VLANs:** crear la 998 (SIN-USO) y la 999 (NATIVA) en SW-HQ-1, SW-HQ-2 y SW-SUC.
2. **Trunks hacia los routers** (SW-SUC Gi0/1 y SW-HQ-1 Gi0/1):
   - VLAN nativa 999.
   - Lista explícita de VLANs permitidas: SW-SUC 50, 99 y 999; SW-HQ-1 10, 20, 30, 40,
     99 y 999.
   - Desactivar la negociación (DTP).
   - Los routers no se tocan: ninguna subinterfaz usa la VLAN nativa.
3. **Trunks entre switches** (Fa0/1 y Fa0/2 en SW-HQ-1 y en SW-HQ-2): nativa 999,
   permitidas 10, 20, 40, 99 y 999, y sin negociación. **Cambiar la nativa en SW-HQ-1 e
   inmediatamente en SW-HQ-2.** Entre un extremo y el otro, STP bloquea la VLAN
   (incidente 1); es esperable y se resuelve al igualar los dos lados.
4. **Puertos de acceso** (SW-HQ-1 Fa0/10 y Fa0/11, SW-HQ-2 Fa0/10 a Fa0/12, SW-SUC
   Fa0/10):
   - PortFast y BPDU Guard.
   - Sin negociación.
   - CDP desactivado en el puerto.
   - Port-security con 1 MAC, aprendizaje sticky y violación con apagado del puerto.
   - Después, hacer un ping desde cada equipo para que el switch aprenda su MAC.
5. **Puertos sin uso:** modo acceso, VLAN 998, descripción "SIN USO" y apagados. Lista
   según SW-01, ya ajustada por PC-TI:
   - SW-HQ-1: Fa0/3-9, Fa0/12-24 y Gi0/2. **Fa0/11 ya no.**
   - SW-HQ-2: Fa0/3-9, Fa0/13-24, Gi0/1 y Gi0/2.
   - SW-SUC: Fa0/1-9, Fa0/11-24 y Gi0/2.
   - Pista: el modo de rango de interfaces ahorra trabajo, pero hay que revisar cada
     rango dos veces antes de apagar.

**Riesgo de cortar algo**
- Nativa distinta en los dos extremos: se corta la VLAN afectada entre los switches. Se
  evita cambiando los dos lados seguidos.
- Una VLAN que falte en la lista de permitidas: por ejemplo, sin la 99 se pierde la
  gestión (B4) y sin la 30 en Gi0/1 de SW-HQ-1 se pierde SRV-TI.
- Un puerto en uso apagado por error dentro de un rango.
- Una violación de port-security: el puerto queda en err-disabled.

**Verificación**

| Prueba | Resultado esperado |
|---|---|
| `show interfaces trunk` en los tres switches | Nativa 999, modo "on" y solo las VLANs de la D14 permitidas |
| `show interfaces <trunk> switchport` | Negotiation of Trunking: Off |
| `show vlan brief` | 998 con los puertos sin uso, 999 creada; los puertos de acceso en su VLAN |
| `show ip interface brief` | Los puertos sin uso en administratively down y ninguno en uso caído |
| `show interfaces <puerto sin uso> switchport` (uno por switch) | "Administrative Mode: static access" y "Access Mode VLAN: 998" |
| `show port-security` y `show port-security address` | Un puerto seguro por equipo, 0 violaciones, y MACs sticky que coinciden con las de `tests/v1-dhcp-pcs.txt` (y las de SRV-TI y PC-TI) |
| `show spanning-tree` en SW-HQ-1 y SW-HQ-2 | Los puertos de acceso como "P2p Edge". En la v1 eran "P2p" sin Edge (evidencia de SW-03). En Packet Tracer no aparece "Edge": la evidencia es `show spanning-tree interface <puerto> detail` ("The port is in the portfast mode") |
| `show cdp interface` en un puerto de acceso | El puerto no aparece |
| Repetir B1 completa, B2 completa, B4.1 y B4.2, y los pings del bloque 1 desde PC-TI | Todo igual que en la v1: las ⚠ siguen pasando porque todavía no hay ACLs |

**Cómo deshacer**
- Nativa: volver a la VLAN 1 en los dos extremos seguidos.
- VLANs permitidas: volver a "todas" con la forma "no".
- Puerto en err-disabled: corregir la causa, apagar y volver a encender el puerto.
- En el peor caso: reiniciar el switch sin guardar o restaurar el `.pkt`.

---

## Bloque 4 — STP y EtherChannel (D15 y D16)

**Equipos y orden:**
1. Rapid PVST+ en los tres switches.
2. Prioridades: primero SW-HQ-1 como raíz principal, luego SW-HQ-2 como secundaria.
3. EtherChannel entre SW-HQ-1 y SW-HQ-2.

**Qué configurar**
1. **Modo de STP:** Rapid PVST+ en SW-HQ-1, SW-HQ-2 y SW-SUC.
2. **Raíz:** SW-HQ-1 como raíz principal y SW-HQ-2 como secundaria, para todas las
   VLANs de la central (1, 10, 20, 30, 40, 99, 998 y 999). Como SW-HQ-1 ya es la raíz
   por su MAC, la topología no debería cambiar.
3. **EtherChannel:**
   - Apagar Fa0/1 y Fa0/2 en **los dos** switches.
   - Agregarlas al grupo 1 con LACP en modo activo en ambos lados.
   - Configurar en el Port-channel 1 lo mismo que tienen hoy los puertos físicos:
     trunk, nativa 999, las VLANs permitidas y sin negociación.
   - Encender los cuatro puertos.

**Riesgo de cortar algo**
- Mientras Fa0/1 y Fa0/2 están apagadas, SW-HQ-2 queda aislado: PC-ADM, PC-VEN y PC-INV
  pierden la red unos minutos. Es esperable.
- Si los puertos o el Port-channel tienen configuraciones distintas, Packet Tracer
  suspende los puertos. Se evita igualando todo antes de encenderlos.
- El cambio a Rapid PVST+ provoca una reconvergencia de unos segundos.

**Verificación**

| Prueba | Resultado esperado |
|---|---|
| `show etherchannel summary` en los dos switches | Po1 con las banderas S y U (capa 2 y en uso), protocolo LACP, Fa0/1 y Fa0/2 con la bandera P (agrupados) |
| `show spanning-tree` en SW-HQ-1 | Protocolo rstp, "This bridge is the root" con prioridad 24576 más el número de VLAN |
| `show spanning-tree` en SW-HQ-2 | Prioridad 28672 más el número de VLAN, Po1 como puerto raíz y **ningún puerto en BLK**. En la v1, Fa0/2 estaba en Altn BLK: es la evidencia del antes y el después de SW-04 |
| `show interfaces trunk` | Po1 aparece como trunk, con la nativa 999 |
| Repetir B1.1 a B1.3, B2 y B4.1 | Éxito |
| Opcional, prueba de redundancia: ping continuo de PC-ADM a SRV-TI y apagar Fa0/1 en SW-HQ-1 | El ping sigue, con como mucho una pérdida. Después, volver a encender Fa0/1 |

**Cómo deshacer:** apagar los cuatro puertos, sacarlos del grupo, borrar el
Port-channel 1, devolver a Fa0/1 y Fa0/2 la configuración de trunk del bloque 3 y
encenderlos. Para STP: volver al modo PVST y quitar las prioridades.

---

## Bloque 5 — DHCP snooping (D19)

**Equipos y orden:** SW-HQ-2, luego SW-SUC. En cada switch, la activación global va
**al final**.

**Qué configurar**
1. Marcar como confiable el puerto que va hacia el router: Port-channel 1 en SW-HQ-2 y
   Gi0/1 en SW-SUC.
2. Desactivar la inserción de la opción 82.
3. Indicar las VLANs: 10, 20 y 40 en SW-HQ-2, y la 50 en SW-SUC.
4. Activar DHCP snooping globalmente. Así, cuando empieza a funcionar, la confianza y la
   opción 82 ya están listas.

**Riesgo de cortar algo:** las PCs pueden quedarse sin IP. Las que ya tienen IP no se
ven afectadas hasta renovar, así que **hay que forzar la renovación para probar**. Si
Packet Tracer no acepta el Port-channel como puerto de confianza, **detenerse y
consultarlo**: no improvisar otro puerto de confianza, porque es una decisión de diseño.

**Verificación**

| Prueba | Resultado esperado |
|---|---|
| `show ip dhcp snooping` en los dos switches | Activo, VLANs correctas, inserción de la opción 82 desactivada y solo el puerto hacia el router como confiable |
| En PC-ADM, PC-VEN, PC-INV y PC-SUC: liberar y renovar la IP (Command Prompt o IP Configuration: Static y luego DHCP) | Cada una recibe una IP de su pool con el gateway y el DNS correctos. Repetir la **fase A completa** |
| `show ip dhcp snooping binding`, si Packet Tracer lo acepta | Una entrada por PC |
| `show ip dhcp binding` en R-HQ y R-SUC | Las asignaciones renovadas |

**Cómo deshacer:** desactivar DHCP snooping globalmente. Con eso basta para volver al
comportamiento anterior.

---

## Bloque 6 — OSPF (D21 y D22)

**Equipos y orden:**
1. Pasivo por defecto en R-HQ y verificar la vecindad.
2. Pasivo por defecto en R-SUC y verificar.
3. Autenticación en G0/0/1 de R-HQ e **inmediatamente** en R-SUC.

**Qué configurar**
1. **Pasivo por defecto (D21):** en el proceso OSPF 1, todas las interfaces pasivas por
   defecto y G0/0/1 como excepción. Hacerlo seguido: en cuanto se activa el pasivo por
   defecto, G0/0/1 también queda pasiva y la vecindad cae. Las sentencias `network` no
   se tocan.
2. **Autenticación MD5 (D22):** en la interfaz G0/0/1 de cada router, activar la
   autenticación por message-digest y definir la clave con el **mismo número de clave
   (1) y la misma clave** en los dos routers. La clave es una de las credenciales de
   `docs/credenciales-v2.md`.

**Riesgo de cortar algo:** la conectividad entre sedes (B3) se corta mientras la
vecindad está caída. Puede tardar hasta 40 segundos en caer y unos segundos en volver.

**Verificación**

| Prueba | Resultado esperado |
|---|---|
| `show ip ospf neighbor` en los dos routers | Un vecino en FULL (2.2.2.2 en R-HQ, 1.1.1.1 en R-SUC) por G0/0/1 |
| `show ip protocols` | Pasivas por defecto, salvo G0/0/1 |
| `show ip ospf interface G0/0/1` | Autenticación message-digest activa, clave 1 |
| `show ip ospf interface` en una subinterfaz LAN | Pasiva, sin vecinos ni hellos |
| `show ip route ospf` | R-HQ sigue aprendiendo 10.20.50.0 y 10.20.99.0; R-SUC, las 10.10.x.0 |
| Repetir la **B3 completa** | Éxito (la B3.7 todavía pasa) |

**Cómo deshacer:** quitar la autenticación en los dos routers (con la forma "no") y
quitar el pasivo por defecto.

---

## Bloque 7 — Acceso seguro: credenciales, SSH y consola (D9, D10 y D11)

**Equipos y orden:**
1. SW-HQ-2 como piloto, completo y verificado.
2. Luego SW-HQ-1, SW-SUC, R-HQ y R-SUC.
3. En cada equipo, la consola va al final.

**Qué configurar (en cada equipo)**
1. **Credenciales:**
   - Crear el usuario local con privilegio 15 y contraseña con hash. Probar primero con
     scrypt; si Packet Tracer no lo acepta, usar el hash normal y anotarlo para
     corregir la tabla de `docs/credenciales-v2.md`.
   - Configurar `enable secret`, distinto en cada equipo, y **borrar el
     `enable password`**.
   - Activar el cifrado de contraseñas del servicio.
2. **SSH:**
   - Definir el dominio `horizonte.internal`.
   - Generar las claves RSA de 2048 bits.
   - Fijar la versión 2.
3. **VTY (todas las que existan, todas igual; INFO-01):**
   - Comprobar primero cuántas líneas VTY tiene el equipo, con `show line` o en la
     running-config. En la v1, los routers tenían solo las líneas 0 a 4 y los switches,
     de 0 a 15. Configurar todos los rangos que existan; si queda uno sin configurar,
     repite la incoherencia de INFO-01.
   - Autenticación con usuarios locales.
   - **Primero, permitir SSH y Telnet a la vez**, para probar SSH sin perder Telnet.
   - Tiempo de inactividad de 10 minutos.
   - **Borrar las contraseñas de línea viejas** (`cisco123`). Si quedan,
     `service password-encryption` las deja en tipo 7 y contradicen
     `docs/credenciales-v2.md`.
4. **Probar SSH desde PC-TI.** Cuando funcione:
   - Dejar las VTY solo con SSH.
   - Crear una ACL estándar que permita 10.10.30.0/24 y termine con un **deny final
     explícito** (obligatorio, para que la D3.1 de `docs/plan-pruebas-v2.md` tenga
     contador).
   - Aplicarla como `access-class` de entrada en las VTY.
   - Por comprobar en Packet Tracer: si el contador de una ACL usada como
     `access-class` sube en `show access-lists`. Si no sube, se documenta como
     limitación del simulador.
5. **Consola, al final y solo después de entrar por SSH con el usuario** (lo que
   confirma que la contraseña funciona): autenticación con usuarios locales y 10
   minutos de inactividad. **No guardar la configuración hasta salir de la consola y
   volver a entrar con el usuario.**

**Riesgo de perder el acceso**
- **Consola:** si se configura el inicio de sesión local con una contraseña mal escrita,
  el equipo queda inaccesible. Por eso va al final, después de probar el usuario por
  SSH, y **no se guarda la configuración hasta confirmar que se puede volver a entrar
  por la consola**. Si la consola queda inaccesible y la configuración todavía no se
  guardó, basta con apagar y encender el equipo desde la pestaña Physical: arranca con
  la última configuración guardada. Ojo: eso también deshace todo lo que no se haya
  guardado del bloque 7 en ese equipo, que habrá que repetir. Restaurar el `.pkt`
  guardado antes del bloque queda como último recurso, si la configuración ya se había
  guardado.
- **Acceso remoto:** si se quita Telnet antes de que SSH funcione, se pierde el acceso
  remoto, aunque la consola sigue disponible.

**Verificación**

| Prueba | Resultado esperado |
|---|---|
| `show ip ssh` | SSH habilitado, versión 2.0 |
| PC-TI, desde el Command Prompt, con el cliente SSH y el usuario: a 10.10.99.11, 10.10.99.12, 10.20.99.11, 10.10.30.1 (R-HQ) y 10.20.50.1 (R-SUC) | Pide contraseña, entra y `enable` acepta el secret |
| `show ssh`, con la sesión abierta | La sesión de PC-TI con la versión 2.0 |
| PC-TI: Telnet a los mismos equipos | Rechazado. Prueba nueva de la D7 |
| PC-ADM: SSH a 10.10.99.12 | Rechazado por la `access-class`. Todavía no hay ACLs: el paquete llega, pero el equipo rechaza la conexión |
| `show access-lists` en SW-HQ-2, antes y después del SSH desde PC-ADM | Sube el contador del deny final de la ACL de las VTY. Si no sube, anotarlo como limitación del simulador |
| `show line` y `show running-config` | Sin `enable password`, sin contraseñas de línea, secrets con hash, todas las líneas VTY que existan configuradas igual y con `access-class` |
| Salir de la consola y volver a entrar | Pide usuario y contraseña |
| Copiar `show ip ssh` y las sesiones a `tests/` | Evidencia de SEC-01 a SEC-04 e INFO-01 |

**Cómo deshacer:** volver a permitir Telnet en las VTY, quitar la `access-class` y quitar
el inicio de sesión de la consola. Si la consola quedó inaccesible: apagar y encender el
equipo desde la pestaña Physical (si no se guardó) o, como último recurso, restaurar el
`.pkt`.

---

## Bloque 8 — ACLs (D23)

**Equipos y orden:**
1. ACL de gestión en R-SUC (la más simple) y verificar.
2. ACL de gestión en R-HQ y verificar.
3. ACL de Invitados en R-HQ y verificar.

En cada caso, primero se escribe la ACL completa y **después** se aplica a la interfaz.

**Qué configurar**
1. **Gestión (R-SUC y R-HQ):**
   - ACL extendida con nombre que permita los orígenes 10.10.30.0/24 y 10.255.255.0/30
     hacia cualquier destino y termine con un **deny final explícito** de todo lo demás
     (obligatorio). El deny implícito no tiene contador en `show access-lists`, y los
     fallos esperados de `docs/plan-pruebas-v2.md` se comprueban con el contador de esa
     línea.
   - Aplicarla **de salida** en G0/0/0.99.
2. **Invitados (R-HQ):** ACL extendida con nombre, en este orden:
   1. DHCP del cliente (UDP 68) al servidor (UDP 67), con cualquier origen y destino,
      para que pase el Discover inicial de 0.0.0.0 a 255.255.255.255.
   2. Eco ICMP solo hacia 10.10.40.1.
   3. UDP 53 solo hacia 10.10.30.10.
   4. Deny final explícito de todo lo demás (obligatorio, por el mismo motivo).

   Aplicarla **de entrada** en G0/0/0.40.

**Riesgo de cortar algo**
- El deny implícito bloquea todo lo que no se permitió.
- Si la regla de DHCP queda mal, PC-INV se queda sin IP.
- Si la ACL se aplica en la dirección equivocada, el efecto es otro.
- NTP depende de la ACL de gestión: las respuestas de SRV-TI hacia los switches salen
  por la .99 y pasan solo si la red de TI está permitida. Syslog no pasa por esa ACL
  (los mensajes entran por la .99, no salen), así que si falla, la ACL quedó aplicada en
  la dirección equivocada.
- No hay riesgo de perder el acceso: la consola no pasa por las ACLs, y PC-TI está
  permitida.

**Verificación (después de cada ACL)**

| Prueba | Resultado esperado |
|---|---|
| `show ip interface G0/0/0.99` y `show ip interface G0/0/0.40` | "Outgoing access list is …" en la .99 e "Inbound access list is …" en la .40 |
| `show access-lists` antes y después de las pruebas | En cada fallo esperado sube el contador del deny final explícito; en las pruebas permitidas suben los de las líneas permit (DHCP, eco y DNS en Invitados; orígenes de TI y WAN en gestión). Es la mejor evidencia de que la ACL actúa |
| Después de la ACL de R-SUC: B4.2 (R-SUC → SW-SUC) y PC-TI → 10.20.99.11 por ping y SSH | Éxito |
| Después de la ACL de R-SUC: B4.3 (PC-ADM → 10.20.99.11) | **Falla** |
| Después de la ACL de R-SUC: **NTP y syslog de SW-SUC** (`show ntp status` en SW-SUC; generar un mensaje en SW-SUC y buscarlo en SYSLOG de SRV-TI) | Sigue sincronizado con 10.10.30.10 y el mensaje llega. Si NTP falla, falta permitir la red de TI en la ACL. Si falla syslog, la ACL quedó aplicada en la dirección equivocada |
| Después de la ACL de R-HQ (gestión): B4.1 y PC-TI → 10.10.99.11 y .12 | Éxito |
| Después de la ACL de R-HQ (gestión): ping de R-SUC a 10.10.99.11 | Éxito (origen WAN) |
| Después de la ACL de R-HQ (gestión): B4.4 y B4.5 | **Fallan** |
| Después de la ACL de R-HQ (gestión): **NTP y syslog de SW-HQ-1 y SW-HQ-2** (la misma comprobación) | Mismo resultado y mismo diagnóstico que en SW-SUC |
| Después de la ACL de Invitados: renovar la IP de PC-INV y repetir la A3 | Obtiene IP |
| Después de la ACL de Invitados: B1.3 | Éxito |
| Después de la ACL de Invitados: `nslookup www.horizonte.internal` desde PC-INV | Responde 10.10.30.10 |
| Después de la ACL de Invitados: B2.2, B2.5, B2.6, B3.7 y C4.4 | **Fallan**. En la B2.2, la respuesta de PC-INV se bloquea al entrar por la .40 |
| Después de la ACL de Invitados: B2.1, B2.3, B2.4, B3.1 a B3.6 y C1 a C4.3 | Éxito |

**Cómo deshacer:** quitar la aplicación de la ACL en la interfaz. El efecto se revierte
de inmediato; la definición de la ACL puede quedar para corregirla.

---

## Bloque 9 — IPv6 (D2): esquema

Se detalla cuando IPv4 esté validado, después de decidir el método de asignación (SLAAC
o DHCPv6 sin estado) y las IPv6 fijas de SRV-TI y PC-TI. Contenido previsto:

- **Routers:** activar el enrutamiento IPv6 y poner direcciones en las subinterfaces de
  usuarios y en el enlace WAN, según los prefijos del diseño. La VLAN 99 no tiene IPv6.
- **OSPFv3:** con los mismos router ID y el mismo criterio de pasivo. Hay que comprobar
  si Packet Tracer soporta autenticación en OSPFv3; si no, se documenta la diferencia
  con RT-02.
- **ACLs IPv6:** equivalentes a las de IPv4 para Invitados (incluido el descubrimiento
  de vecinos y de routers) y para las VTY. **Se aplican en el mismo bloque, antes de
  validar**, porque en cuanto los routers anuncian prefijos, Invitados ya tiene IPv6.
- **Verificación:** direcciones en las PCs, ping IPv6 entre VLANs y entre sedes, y las
  mismas pruebas de aislamiento de Invitados repetidas en IPv6.

---

## Exportación y pruebas finales

**Cuándo exportar las running-config a `configurations/v2/`**

1. **Al terminar el bloque 8**, cuando IPv4 esté completo y validado:
   - Guardar la configuración en cada equipo.
   - Exportar las cinco running-config completas, sin omitir nada (D9), con el mismo
     método que en la v1.
   - Hacer commit.
   - Así queda la v2 en IPv4 aunque el bloque 9 se retrase.
2. **Al terminar el bloque 9:** volver a exportar los dos routers (los switches no
   cambian) y hacer otro commit. El historial de git conserva la versión de solo IPv4.
3. **SRV-TI** no tiene running-config: capturas de la pestaña Services (DNS, HTTP, NTP,
   Syslog y los servicios apagados) en `evidence/v2-...png`.

**Qué pruebas repetir (con `docs/plan-pruebas-v2.md`)**

- **Fase A completa**, renovando la IP en las cuatro PCs.
- **Fase B completa:**
  - Fallos esperados: B2.2, B2.5, B2.6, B3.7, B4.3, B4.4 y B4.5.
  - El resto debe dar éxito.
- **Fase C completa:** C4.4 falla; el resto da éxito.
- **Pruebas nuevas de la D7:**
  - PC-TI llega a la gestión y entra por SSH.
  - Telnet rechazado desde PC-TI.
  - Invitados obtiene IP, resuelve DNS y no carga la web interna.
- **Comandos `show` del antes y el después**, para el README:
  - `show spanning-tree`, `show interfaces trunk`, `show vlan brief` y
    `show etherchannel summary`
  - `show port-security`, `show ip dhcp snooping` y `show ip ssh`
  - `show ip ospf neighbor` y `show ip ospf interface`
  - `show access-lists` y `show ntp status`
- **NTP y syslog:** `show ntp status` en los cinco equipos y un mensaje de cada uno en
  SYSLOG de SRV-TI, como comprobación final después de todos los bloques.
