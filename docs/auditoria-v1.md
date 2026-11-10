# Auditoría de la red v1

> Proyecto de portafolio. Comercial Horizonte S.A. es una empresa ficticia y la red es
> una simulación en Cisco Packet Tracer. Los defectos de la v1 son intencionales: la
> red representa una instalación hecha "con prisa" por una pyme.

## Alcance y metodología

- **Qué se revisó:** las running-config exportadas de los cinco equipos IOS en
  `configurations/v1/` (R-HQ, R-SUC, SW-HQ-1, SW-HQ-2 y SW-SUC), la evidencia de
  `tests/` y `evidence/`, y los resultados del plan de pruebas
  (`docs/plan-pruebas-v1.md`). SRV-TI no tiene running-config; se revisó con las
  capturas de la fase C.
- **Contra qué se comparó:** el diseño de la red (CLAUDE.md) y buenas prácticas de
  seguridad y operación para equipos Cisco.
- **Quién hizo qué:** Claude Code revisó los archivos y redactó los hallazgos. Karla
  revisó la lista, la aprobó y obtuvo en Packet Tracer la evidencia complementaria
  (sección "Evidencia complementaria").
- **Qué no incluye:** los comandos de corrección. Las recomendaciones dicen qué cambiar
  en la v2; el cómo se prepara en la fase 4.

### Escala de severidad

| Severidad | Criterio |
|---|---|
| Alta | Permite acceso no autorizado a equipos o redes internas, o expone credenciales |
| Media | Debilita la red frente a errores o ataques en la LAN, o compromete su estabilidad |
| Baja | Afecta la operación, el diagnóstico o la consistencia, sin riesgo directo |
| Info | Observación sin impacto práctico, registrada para que la v2 quede limpia |

## Comparación con el diseño

**Una diferencia: IPv6 (DIS-01).** El diseño planifica prefijos IPv6 para las VLANs de
usuarios y el enlace WAN, y ninguna configuración tiene IPv6. La versión inicial de esta
auditoría decía "Sin diferencias", porque el plan de pruebas dejaba IPv6 fuera de
alcance; se corrigió después de la comparación con las debilidades intencionales
(`docs/comparacion-auditoria-v1.md`). Todo lo demás coincide con CLAUDE.md: las
subinterfaces y sus IPs, las exclusiones y los pools DHCP (DNS y dominio incluidos), los
router ID y las redes OSPF, los trunks, los puertos de acceso y sus VLANs, y las SVI de
gestión con su gateway. Salvo DIS-01, los hallazgos de este documento son de buenas prácticas, no errores de configuración: la red funciona,
como confirman las fases A, B y C del plan de pruebas.

La evidencia complementaria (`tests/v1-sw-stp-vlans-trunks.txt`) confirma además los
nombres de las VLANs y la asignación de los puertos de acceso en SW-HQ-2 y SW-SUC
(`show vlan brief`), incluida la VLAN 50 (VENTAS-SUCURSAL), y la de Fa0/10 de SW-HQ-1 en
la VLAN 30 (`show spanning-tree`).

## Resumen de hallazgos

| ID | Severidad | Hallazgo | Equipos |
|---|---|---|---|
| SEC-01 | Alta | Acceso remoto solo por Telnet, sin SSH | Los 5 |
| SEC-02 | Alta | `enable password` en vez de `enable secret`, con la misma contraseña en todos | Los 5 |
| SEC-03 | Alta | Contraseñas sin cifrar y VTY sin usuarios locales | Los 5 |
| SEG-01 | Alta | Sin ACLs: Invitados llega a las redes internas y a la sucursal | R-HQ, R-SUC |
| SEG-02 | Alta | La red de gestión se alcanza desde VLANs de usuarios | Los 5 |
| SEC-04 | Media | Consola sin contraseña y sin tiempo de inactividad | Los 5 |
| SW-01 | Media | Puertos sin uso activos y en VLAN 1 | Switches |
| SW-02 | Media | Trunks con VLAN nativa 1, todas las VLANs permitidas y DTP activo | Switches |
| SW-03 | Media | Puertos de acceso sin PortFast, BPDU Guard ni port-security | Switches |
| SW-04 | Media | STP sin puente raíz definido y enlaces redundantes sin EtherChannel | SW-HQ-1, SW-HQ-2 |
| RT-01 | Media | OSPF sin `passive-interface` en las interfaces LAN | R-HQ, R-SUC |
| RT-02 | Media | OSPF sin autenticación en el enlace WAN | R-HQ, R-SUC |
| SW-05 | Media | Sin DHCP snooping | Switches |
| SW-06 | Baja | CDP activo en los puertos de usuarios | Switches |
| DIS-01 | Baja | IPv6 planificado en el diseño y no implementado (agregado después de la comparación) | R-HQ, R-SUC |
| SEC-05 | Baja | Sin banner de aviso legal | Los 5 |
| OP-01 | Baja | Logs sin marca de tiempo, sin NTP y sin syslog | Los 5 |
| OP-02 | Baja | Interfaces sin `description` | Los 5 |
| OP-03 | Baja | SW-HQ-1 sin `no ip domain-lookup`, a diferencia del resto | SW-HQ-1 |
| OP-04 | Baja | SRV-TI con HTTPS activado y archivos de ejemplo | SRV-TI |
| INFO-01 | Info | `line vty 5 15` con `login` y sin contraseña | Switches |

Total: 5 de severidad alta, 8 media, 7 baja y 1 informativa (21 hallazgos).

## Hallazgos en detalle

### Seguridad de acceso a los equipos

#### SEC-01 — Acceso remoto solo por Telnet (Alta)

- **Evidencia:** `transport input telnet` en `line vty 0 4` de los cinco equipos
  (`R-HQ.txt:120-123`, `R-SUC.txt:93-96`, `SW-HQ-1.txt:90-93`, `SW-HQ-2.txt:94-97`,
  `SW-SUC.txt:89-92`). No hay `ip domain-name`, claves RSA ni usuarios locales, que son
  requisitos de SSH.
- **Riesgo:** Telnet envía usuario, contraseña y toda la sesión en texto plano.
  Cualquiera que capture tráfico en el camino obtiene las credenciales de
  administración.
- **Recomendación para la v2:** SSH versión 2 como único protocolo de las VTY, con
  usuarios locales (`login local`).

#### SEC-02 — `enable password` y contraseña repetida (Alta)

- **Evidencia:** `enable password cisco123` en los cinco equipos (`R-HQ.txt:11`,
  `R-SUC.txt:11`, `SW-HQ-1.txt:9`, `SW-HQ-2.txt:9`, `SW-SUC.txt:9`).
- **Riesgo:** `enable password` se guarda sin hash, o con un cifrado reversible si se
  activa `service password-encryption`. Como la contraseña es la misma en toda la red,
  conocerla da acceso privilegiado a todos los equipos. Además, es una contraseña débil.
- **Recomendación para la v2:** `enable secret` (con hash) y eliminar
  `enable password`. Contraseñas robustas y distintas por equipo, o por función.

#### SEC-03 — Contraseñas sin cifrar y VTY sin usuarios (Alta)

- **Evidencia:** `no service password-encryption` (línea 5 de los cinco archivos). Las
  VTY usan `password cisco123` con `login`, una contraseña de línea compartida y sin
  usuarios.
- **Riesgo:** cualquiera que vea la configuración (un respaldo, una captura de pantalla
  o un `show running-config`) obtiene las contraseñas. Sin usuarios individuales no se
  sabe quién se conectó ni quién hizo un cambio.
- **Recomendación para la v2:** usuarios locales con contraseña con hash, `login local`
  en las VTY y `service password-encryption` como protección mínima para las
  contraseñas que queden en texto plano.

#### SEC-04 — Consola sin contraseña ni tiempo de inactividad (Media)

- **Evidencia:** `line con 0` sin configuración en los cinco equipos (por ejemplo,
  `R-HQ.txt:116-117`). Ninguna línea tiene `exec-timeout`.
- **Riesgo:** quien tenga acceso físico al equipo entra sin credenciales, y una sesión
  abierta queda disponible indefinidamente. En un laboratorio el riesgo es bajo; en una
  oficina real, no.
- **Recomendación para la v2:** autenticación en la consola (usuarios locales) y
  `exec-timeout` en la consola y en las VTY.

#### SEC-05 — Sin banner de aviso legal (Baja)

- **Evidencia:** ningún equipo tiene `banner motd` ni `banner login`.
- **Riesgo:** no se advierte que el acceso está restringido, un aviso que en muchas
  políticas se exige para poder actuar ante un acceso no autorizado.
- **Recomendación para la v2:** un banner de acceso restringido en todos los equipos,
  sin datos que identifiquen el modelo ni la red.

### Segmentación y control de tráfico

#### SEG-01 — Sin ACLs entre VLANs ni entre sedes (Alta)

- **Evidencia:** ninguna configuración tiene `access-list` ni `ip access-group`. Las
  pruebas lo confirman: la red de Invitados (VLAN 40) llega a SRV-TI (B2.5), a
  Administración (B2.6), a la sucursal (B3.7) y a la web interna (C4.4).
- **Riesgo:** un dispositivo de un invitado tiene alcance a toda la red interna de las
  dos sedes, incluidos el servidor y el área de Administración.
- **Recomendación para la v2:** ACLs en R-HQ que limiten la VLAN 40 a lo que un
  invitado necesita (el gateway, el DHCP y lo que se decida para el DNS; ver la
  pregunta abierta) y bloqueen el resto de las redes internas y la sucursal. Revisar
  también qué tráfico necesita cada VLAN de usuarios hacia las demás.

#### SEG-02 — Gestión accesible desde VLANs de usuarios (Alta)

- **Evidencia:** PC-ADM llega a la gestión de la sucursal (B4.3), PC-SUC a la de la
  central (B4.4) y PC-INV a SW-HQ-2 (B4.5). Las VTY no tienen `access-class`, así que
  cualquier equipo que alcance la IP de gestión puede intentar conectarse.
- **Riesgo:** sumado a SEC-01 a SEC-03, cualquier PC de usuario, incluidas las de
  Invitados, puede abrir una sesión Telnet a los switches y probar la contraseña, que
  es la misma en toda la red.
- **Recomendación para la v2:** limitar el acceso a las redes de gestión (VLAN 99 de
  cada sede) con ACLs en los routers, y restringir las VTY con `access-class` a los
  orígenes autorizados (por ejemplo, la red de TI).

### Switching

#### SW-01 — Puertos sin uso activos y en VLAN 1 (Media)

- **Evidencia:** los puertos sin configuración no tienen `shutdown`, así que quedan
  activos en la VLAN 1:
  - SW-HQ-1: Fa0/3-9, Fa0/11-24 y Gi0/2.
  - SW-HQ-2: Fa0/3-9, Fa0/13-24, Gi0/1 y Gi0/2.
  - SW-SUC: Fa0/1-9, Fa0/11-24 y Gi0/2.
  `show vlan brief` de SW-HQ-2 y SW-SUC (`tests/v1-sw-stp-vlans-trunks.txt`) muestra
  los puertos sin uso en la VLAN 1, incluidos Gi0/1 y Gi0/2 de SW-HQ-2. La salida de
  SW-HQ-1 en `tests/v1-sw-hq-1-vlans-trunks.txt` es de la construcción (Fa0/10 todavía
  aparece en la VLAN 1), así que no se usa como evidencia del estado actual.
- **Riesgo:** cualquiera que conecte un equipo a una toma libre obtiene acceso a la red.
  Con DTP activo (SW-02), el puerto podría incluso negociar un trunk.
- **Recomendación para la v2:** apagar los puertos sin uso y asignarlos a una VLAN sin
  uso y sin enrutamiento (VLAN "agujero negro"), en modo acceso.

#### SW-02 — Trunks con VLAN nativa 1, todas las VLANs permitidas y DTP (Media)

- **Evidencia:** `switchport mode trunk` sin más opciones en `SW-HQ-1.txt:19-23` y
  `:71-72`, `SW-HQ-2.txt:20-24` y `SW-SUC.txt:70-71`. En SW-HQ-1,
  `show interfaces trunk` muestra VLAN nativa 1 y VLANs permitidas 1-1005
  (`tests/v1-sw-hq-1-vlans-trunks.txt:35-41`, salida de la construcción). En SW-HQ-2
  (Fa0/1 y Fa0/2) y SW-SUC (Gi0/1) se ve lo mismo en las salidas actuales
  (`tests/v1-sw-stp-vlans-trunks.txt`).
- **Riesgo:** la VLAN nativa 1 coincide con la VLAN de los puertos sin uso (SW-01), lo
  que facilita ataques de doble etiquetado. Permitir todas las VLANs extiende por todos
  los trunks VLANs que no deberían pasar. DTP activo permite negociar trunks no
  previstos.
- **Recomendación para la v2:** una VLAN nativa sin uso, **igual en ambos extremos**
  de cada trunk (el incidente 1 de la construcción fue justo una nativa distinta en
  cada extremo), una lista explícita de VLANs permitidas por trunk y
  `switchport nonegotiate`. En los trunks hacia los routers, revisar el efecto de la
  VLAN nativa en la configuración de las subinterfaces.

#### SW-03 — Puertos de acceso sin protecciones (Media)

- **Evidencia:** los puertos de PCs y servidor solo tienen `switchport access vlan` y
  `switchport mode access` (`SW-HQ-1.txt:39-41`, `SW-HQ-2.txt:40-50`,
  `SW-SUC.txt:38-40`). En `show spanning-tree`, Fa0/10 de SW-HQ-1 y Fa0/10, Fa0/11 y
  Fa0/12 de SW-HQ-2 aparecen como `P2p` sin `Edge`, lo que confirma que no tienen
  PortFast (`tests/v1-sw-stp-vlans-trunks.txt`).
- **Riesgo:** sin BPDU Guard, un switch conectado por error o a propósito participa en
  STP y puede alterar la topología. Sin PortFast, los equipos esperan la convergencia de
  STP al conectarse. Sin port-security, cualquier equipo puede reemplazar al autorizado.
- **Recomendación para la v2:** PortFast y BPDU Guard en todos los puertos de acceso,
  port-security con límite de MAC donde corresponda (al menos en el puerto de SRV-TI)
  y `switchport nonegotiate`.

#### SW-04 — STP sin puente raíz definido (Media)

- **Evidencia:** solo `spanning-tree mode pvst` (`SW-HQ-1.txt:16-17`,
  `SW-HQ-2.txt:17-18`), sin prioridades configuradas, y dos enlaces entre SW-HQ-1 y
  SW-HQ-2 (Fa0/1 y Fa0/2) sin EtherChannel. `show spanning-tree`
  (`tests/v1-sw-stp-vlans-trunks.txt`) muestra a SW-HQ-1 como puente raíz en las seis
  VLANs, con la prioridad por defecto, elegido por tener la MAC más baja
  (000D.BD84.508B frente a 0030.F29A.CABC). SW-HQ-2 Fa0/2 está en `Altn BLK` en todas
  las VLANs. El protocolo es `ieee` (PVST clásico).
- **Riesgo:** la raíz es la correcta por casualidad, no por diseño: si cambia el
  hardware, puede cambiar la raíz y el tráfico tomaría caminos no previstos. Uno de los
  dos enlaces entre SW-HQ-1 y SW-HQ-2 está bloqueado, así que hay redundancia pero se
  usa la mitad del ancho de banda. PVST clásico tarda de 30 a 50 segundos en converger.
- **Severidad:** se mantiene en Media. Aunque la raíz es hoy la correcta, el enlace
  bloqueado y la convergencia lenta de PVST son riesgos actuales.
- **Recomendación para la v2:** SW-HQ-1 como puente raíz de todas las VLANs de la
  central, Rapid PVST+ y agrupar Fa0/1 y Fa0/2 en un EtherChannel (LACP).

#### SW-05 — Sin DHCP snooping (Media)

- **Evidencia:** ningún switch tiene `ip dhcp snooping`.
- **Riesgo:** un servidor DHCP no autorizado en una VLAN de usuarios podría entregar
  otro gateway u otro DNS y desviar el tráfico.
- **Recomendación para la v2:** DHCP snooping en las VLANs con pool, con los trunks
  hacia los routers como puertos de confianza. Comprobar antes qué soporta Packet
  Tracer en el 2960.
- **Severidad:** subió de Baja a Media después de la comparación. Según la escala de
  esta auditoría, debilita la red frente a ataques en la LAN, y la VLAN 40 conecta
  equipos que no se controlan.

#### SW-06 — CDP activo en los puertos de usuarios (Baja)

- **Evidencia:** ninguna configuración tiene `no cdp run` ni `no cdp enable`, así que
  CDP está activo por defecto en todos los puertos. `show cdp interface` en SW-HQ-2
  Fa0/12 (PC-INV, VLAN 40) lo confirma: envía anuncios cada 60 segundos
  (`tests/v1-sw-hq-2-cdp.txt`). Agregado después de la comparación.
- **Riesgo:** cualquier equipo conectado a un puerto de usuario, incluido Invitados,
  recibe el nombre del switch, su modelo, su versión de IOS y su IP de gestión. No da
  acceso por sí solo, pero facilita preparar un ataque contra la gestión (SEG-02,
  SEC-01).
- **Recomendación para la v2:** desactivar CDP en los puertos de usuarios y mantenerlo
  solo entre equipos de red (trunks y enlaces hacia los routers).

### Enrutamiento

#### RT-01 — OSPF sin `passive-interface` (Media)

- **Evidencia:** `router ospf 1` sin `passive-interface` (`R-HQ.txt:96-104`,
  `R-SUC.txt:72-77`). Las sentencias `network` activan OSPF en todas las subinterfaces
  LAN.
- **Riesgo:** los routers envían hellos de OSPF a las VLANs de usuarios e Invitados. Un
  equipo con software de enrutamiento en esas VLANs podría formar vecindad e inyectar
  rutas falsas.
- **Recomendación para la v2:** todas las interfaces pasivas por defecto, con OSPF
  activo solo en el enlace WAN (G0/0/1). Las redes LAN se siguen anunciando.

#### RT-02 — OSPF sin autenticación (Media)

- **Evidencia:** no hay autenticación OSPF ni en el proceso ni en G0/0/1 de ningún
  router.
- **Riesgo:** sin autenticación, los routers aceptan como vecino a cualquier router que
  hable OSPF en el segmento. El riesgo en el enlace WAN es menor que en las LAN (RT-01),
  pero es una buena práctica básica.
- **Recomendación para la v2:** autenticación OSPF en el enlace WAN, con el método más
  robusto que soporte Packet Tracer (MD5 como mínimo).

### Operación y consistencia

#### OP-01 — Logs sin marca de tiempo, sin NTP y sin syslog (Baja)

- **Evidencia:** `no service timestamps log datetime msec` y
  `no service timestamps debug datetime msec` (líneas 3-4 de los cinco archivos). No
  hay `ntp server` ni `logging host`.
- **Riesgo:** sin marcas de tiempo ni relojes sincronizados, no se puede reconstruir
  qué pasó ni en qué orden durante un incidente. Los logs solo quedan en el buffer de
  cada equipo.
- **Recomendación para la v2:** marcas de tiempo en los logs, NTP y syslog hacia
  SRV-TI, que tiene ambos servicios disponibles en Packet Tracer.

#### OP-02 — Interfaces sin `description` (Baja)

- **Evidencia:** ninguna interfaz de los cinco equipos tiene `description`.
- **Riesgo:** para saber a dónde va cada puerto hay que consultar `conexiones.md`. Así
  es más fácil equivocarse al hacer cambios o al diagnosticar.
- **Recomendación para la v2:** `description` en los trunks, el enlace WAN, las
  subinterfaces y los puertos de acceso en uso, según `conexiones.md`.

#### OP-03 — SW-HQ-1 sin `no ip domain-lookup` (Baja)

- **Evidencia:** `SW-HQ-2.txt:13`, `SW-SUC.txt:13`, `R-HQ.txt:50` y `R-SUC.txt:38`
  tienen `no ip domain-lookup`; `SW-HQ-1.txt` no.
- **Riesgo:** en SW-HQ-1, un comando mal escrito en EXEC se interpreta como un nombre de
  equipo y bloquea la consola mientras intenta resolverlo. Es una inconsistencia entre
  equipos configurados "a mano".
- **Recomendación para la v2:** la misma configuración base en todos los equipos. Con
  SRV-TI como DNS, decidir si los equipos deben resolver nombres (con `ip name-server`)
  o mantener la búsqueda desactivada.

#### OP-04 — SRV-TI con HTTPS activado y archivos de ejemplo (Baja)

- **Evidencia:** `evidence/v1-c0-srv-ti-http.png`: HTTPS en On y los archivos
  `helloworld.html`, `image.html`, `copyrights.html` y `cscoptlogo177x111.jpg`.
- **Riesgo:** servicios y contenido que nadie usa ni revisa amplían lo que el servidor
  expone. Son valores por defecto que se dejaron sin revisar.
- **Recomendación para la v2:** desactivar los servicios que no forman parte del diseño
  y eliminar los archivos de ejemplo.

#### INFO-01 — `line vty 5 15` con `login` y sin contraseña (Info)

- **Evidencia:** `SW-HQ-1.txt:94-95`, `SW-HQ-2.txt:98-99` y `SW-SUC.txt:93-94`.
- **Observación:** con `login` y sin contraseña, IOS rechaza las conexiones en esas
  líneas, así que no hay un acceso abierto. Pero la configuración es incoherente con las
  líneas 0-4.
- **Recomendación para la v2:** aplicar a todas las VTY (0-15) la misma configuración
  de acceso.

### Diseño

#### DIS-01 — IPv6 planificado y no implementado (Baja)

- **Evidencia:** CLAUDE.md asigna prefijos IPv6 a las VLANs 10, 20, 30, 40 y 50 y al
  enlace WAN (columna "IPv6 (planificado)"). Ninguna running-config tiene direcciones
  IPv6, y los dos routers tienen `no ipv6 cef` (`R-HQ.txt:37`, `R-SUC.txt:25`).
  Agregado después de la comparación.
- **Riesgo:** no es un riesgo de seguridad; es una brecha entre el diseño y lo
  construido. La documentación promete algo que la red no tiene.
- **Recomendación para la v2:** implementar IPv6 según los prefijos del diseño, o dejar
  registrado en el diseño que se pospone. Es una decisión de la fase 4.

## Descartado

- **Línea `version` de las configuraciones** (16.6.4 en R-HQ y 15.4 en R-SUC): no indica
  el IOS que ejecuta el equipo. Ya se registró como falso positivo; según
  `show version`, los dos routers tienen el mismo IOS (`docs/uso-de-ia.md`, entrada del
  2026-11-09).
- **Elementos correctos en la v1** que no requieren cambio: G0/0/2 apagada en los dos
  routers, `Vlan1` apagada en los tres switches, exclusiones DHCP por red, dominio
  `horizonte.internal` y router ID fijos en OSPF.

## Evidencia complementaria

Las running-config no incluyen la base de VLANs ni el estado de STP. Karla obtuvo estas
salidas en Packet Tracer, en modo EXEC privilegiado (`#`); están en
`tests/v1-sw-stp-vlans-trunks.txt`.

| Comando | Equipos | Para qué | Resultado |
|---|---|---|---|
| `show vlan brief` | SW-HQ-2, SW-SUC | Confirmar la base de VLANs y sus nombres | Obtenida: coincide con el diseño, incluida la VLAN 50; confirma SW-01 |
| `show spanning-tree` | SW-HQ-1, SW-HQ-2 | Confirmar el puente raíz y el puerto bloqueado | Obtenida: confirma SW-04 y aporta evidencia a SW-03 |
| `show interfaces trunk` | SW-HQ-2, SW-SUC | Completar la evidencia de SW-02 | Obtenida: confirma SW-02 |

## Pregunta abierta para la fase 4

El pool INVITADOS entrega el DNS interno (10.10.30.10), como define el diseño. Si en la
v2 se aísla la VLAN 40 (SEG-01), hay que decidir si Invitados sigue usando SRV-TI como
DNS, lo que exige permitir ese tráfico en la ACL, o si usa otro servidor. No se registra
como hallazgo porque es una decisión de diseño.
