# Proyecto: Diseño y optimización de una red empresarial

## Contexto

Proyecto de portafolio de Karla Araya (ingeniera informática, CCNA) para demostrar
conocimientos prácticos de redes y el uso de Claude Code en tareas de infraestructura.

La empresa **Comercial Horizonte S.A. es ficticia** y la red es una **simulación en
Cisco Packet Tracer**. Toda la documentación debe dejarlo claro. Nunca presentar este
proyecto como experiencia en una empresa real.

## Tu rol, Claude

- Eres asistente de Karla, no su sustituto. Ella ejecuta los comandos en Packet Tracer,
  hace las pruebas y toma las decisiones.
- No tienes acceso a Packet Tracer. Trabajas solo con los archivos de esta carpeta.
  Nunca digas que algo "funciona" si no hay una salida de comando o captura que lo pruebe.
- Explica en español y de forma clara. Cuando des comandos de Cisco IOS, explica qué
  hace cada uno y en qué modo se escribe (`#`, `(config)#`, `(config-if)#`, etc.).
- Cuando Karla pida solo instrucciones, no le des los comandos: dale pistas.
- Si una configuración no coincide con el diseño de este archivo, señálalo y pregunta
  antes de asumir cuál de los dos está mal.

## Registro de uso de IA (obligatorio)

Al terminar cada tarea, agrega una entrada al final de `docs/uso-de-ia.md` (créalo si
no existe) con este formato:

```
### AAAA-MM-DD — Título corto
- Qué pidió Karla:
- Qué propuso Claude:
- Qué verificó Karla y cómo:
- Qué se corrigió o se rechazó, y por qué:
```

Deja en blanco "Qué verificó Karla" y "Qué se corrigió": esos campos los completa ella.

## Estructura de carpetas

```
enterprise-network-optimization/
├── CLAUDE.md
├── README.md
├── conexiones.md
├── diagrams/          archivos .pkt y capturas de la topología
├── configurations/
│   ├── v1/            running-config exportada de cada equipo (red base)
│   └── v2/            running-config después de las mejoras
├── tests/             salidas de ping, show ip route, show ip ospf neighbor, etc.
├── evidence/          capturas de pantalla
└── docs/              informes, auditoría y uso-de-ia.md
```

## Fases del proyecto

1. Diseño — terminado.
2. Red base (v1) — terminada.
3. Auditoría con Claude Code de `configurations/v1/` — terminada
   (`docs/auditoria-v1.md` y `docs/comparacion-auditoria-v1.md`).
4. Red optimizada (v2) y validación — en curso (decisiones del grupo 1 tomadas).
5. Documentación final (README con comparación antes/después y uso de IA).

## Diseño de la red

### Equipos

| Equipo | Modelo | Rol |
|---|---|---|
| R-HQ | ISR4331 | Router sede central (router-on-a-stick, DHCP, OSPF) |
| R-SUC | ISR4331 | Router sucursal (router-on-a-stick, DHCP, OSPF) |
| SW-HQ-1 | 2960 | Switch de distribución, sede central |
| SW-HQ-2 | 2960 | Switch de acceso, sede central |
| SW-SUC | 2960 | Switch de acceso, sucursal |
| SRV-TI | Server-PT | Servidor DNS y web, IP fija |
| PC-ADM, PC-VEN, PC-INV | PC-PT | PCs de prueba, sede central |
| PC-SUC | PC-PT | PC de prueba, sucursal |
| PC-TI | PC-PT | Estación de administración, VLAN 30, IP fija (v2) |

### Conexiones (cable directo de cobre)

| Origen | Puerto | Destino | Puerto | Uso |
|---|---|---|---|---|
| R-HQ | G0/0/0 | SW-HQ-1 | G0/1 | Trunk router-on-a-stick |
| R-HQ | G0/0/1 | R-SUC | G0/0/1 | Enlace WAN |
| SW-HQ-1 | Fa0/1 | SW-HQ-2 | Fa0/1 | Enlace entre switches 1 |
| SW-HQ-1 | Fa0/2 | SW-HQ-2 | Fa0/2 | Enlace entre switches 2 |
| SW-HQ-1 | Fa0/10 | SRV-TI | Fa0 | VLAN 30 |
| SW-HQ-1 | Fa0/11 | PC-TI | Fa0 | VLAN 30 (v2) |
| SW-HQ-2 | Fa0/10 | PC-ADM | Fa0 | VLAN 10 |
| SW-HQ-2 | Fa0/11 | PC-VEN | Fa0 | VLAN 20 |
| SW-HQ-2 | Fa0/12 | PC-INV | Fa0 | VLAN 40 |
| R-SUC | G0/0/0 | SW-SUC | G0/1 | Trunk router-on-a-stick |
| SW-SUC | Fa0/10 | PC-SUC | Fa0 | VLAN 50 |

### VLANs y direccionamiento

Convención: el número de VLAN coincide con el tercer octeto de la red.

| VLAN | Nombre | Sede | Red IPv4 | Gateway | IPv6 (planificado) |
|---|---|---|---|---|---|
| 10 | ADMINISTRACION | Central | 10.10.10.0/24 | 10.10.10.1 | 2001:db8:acad:10::/64 |
| 20 | VENTAS | Central | 10.10.20.0/24 | 10.10.20.1 | 2001:db8:acad:20::/64 |
| 30 | TI | Central | 10.10.30.0/24 | 10.10.30.1 | 2001:db8:acad:30::/64 |
| 40 | INVITADOS | Central | 10.10.40.0/24 | 10.10.40.1 | 2001:db8:acad:40::/64 |
| 99 | GESTION | Central | 10.10.99.0/24 | 10.10.99.1 | — |
| 50 | VENTAS-SUCURSAL | Sucursal | 10.20.50.0/24 | 10.20.50.1 | 2001:db8:acad:50::/64 |
| 99 | GESTION | Sucursal | 10.20.99.0/24 | 10.20.99.1 | — |
| — | Enlace WAN | — | 10.255.255.0/30 | R-HQ .1 / R-SUC .2 | 2001:db8:acad:ff::/64 |

### IPs fijas

| Equipo | IP |
|---|---|
| SRV-TI | 10.10.30.10/24, gateway 10.10.30.1 |
| SW-HQ-1 (Vlan99) | 10.10.99.11/24, gateway 10.10.99.1 |
| SW-HQ-2 (Vlan99) | 10.10.99.12/24, gateway 10.10.99.1 |
| SW-SUC (Vlan99) | 10.20.99.11/24, gateway 10.20.99.1 |
| PC-TI (v2) | 10.10.30.11/24, gateway 10.10.30.1, DNS 10.10.30.10 |

### DHCP

- Se excluyen de .1 a .20 en cada red con pool (reservadas para equipos fijos).
- DNS en todos los pools: 10.10.30.10 (SRV-TI). Dominio: `horizonte.internal`.
- Registros DNS en SRV-TI (tipo A, ambos hacia 10.10.30.10):
  `srv-ti.horizonte.internal` (equipo) y `www.horizonte.internal` (servicio web).

| Pool | Router | Red | Gateway |
|---|---|---|---|
| ADMINISTRACION | R-HQ | 10.10.10.0/24 | 10.10.10.1 |
| VENTAS | R-HQ | 10.10.20.0/24 | 10.10.20.1 |
| INVITADOS | R-HQ | 10.10.40.0/24 | 10.10.40.1 |
| VENTAS-SUCURSAL | R-SUC | 10.20.50.0/24 | 10.20.50.1 |

Las VLANs 30 y 99 no tienen pool: sus equipos usan IP fija.

### OSPF

Proceso 1, área 0. Router ID: R-HQ = 1.1.1.1, R-SUC = 2.2.2.2.
Cada router anuncia sus redes LAN y la red WAN 10.255.255.0/30.

## Estado actual de la v1

- Listos: SW-HQ-1, SW-HQ-2, SW-SUC, R-HQ, R-SUC.
- SRV-TI: IP fija (10.10.30.10/24) y servicios DNS y web configurados y verificados
  (fase C del plan de pruebas: `tests/v1-dns-web.txt` y capturas en `evidence/`).
- PCs: las cuatro reciben IP por DHCP (fase A del plan de pruebas, con evidencia en
  `tests/v1-dhcp-pcs.txt`). La fase B del plan de pruebas (B1 a B4) está verificada
  (`tests/v1-ping.txt`).
- Sin pendientes: la v1 está completa y verificada (fases A, B y C del plan de pruebas).
  La auditoría (fase 3) registró 21 hallazgos en `docs/auditoria-v1.md`, que son la
  base de la v2.

La v1 representa una red armada "con prisa" por una pyme: funciona, pero no aplica
buenas prácticas. Es intencional y se corrige en la v2. No corrijas la v1 por tu
cuenta; los hallazgos se documentan en la auditoría de la fase 3.

## Decisiones de diseño de la v2

Tomadas por Karla el 2026-11-11, a partir de la auditoría (`docs/auditoria-v1.md`).
Detalle de las opciones en `docs/uso-de-ia.md`.

### Grupo 1 (diseño y pruebas)

- **Alcance (D1):** se corrigen los 21 hallazgos. IPv6 (DIS-01) es el último paso, y
  solo cuando IPv4 esté validado.
- **IPv6 (D2):** implementación completa según los prefijos del diseño, con ACLs IPv6
  equivalentes a las de IPv4 (incluido el acceso a las VTY), para que no se pueda
  saltar el aislamiento por IPv6. Las ACLs IPv6 deben permitir el descubrimiento de
  vecinos y de routers. El método de asignación (SLAAC o DHCPv6 sin estado) y las IPv6
  fijas de SRV-TI y PC-TI se deciden en ese paso.
- **Administración (D3):** PC-TI, en la VLAN 30, es la estación de administración.
- **Gestión (D4):** a las redes de gestión (VLAN 99 de ambas sedes) solo llegan la red
  de TI (10.10.30.0/24) y los routers (incluida la red WAN 10.255.255.0/30 como
  origen). Todo lo demás se bloquea. Riesgo aceptado: al permitir toda la red de TI,
  si alguien compromete SRV-TI, llega a la gestión. Consecuencia aceptada: la gestión
  de una sede no llega a la de la otra (por ejemplo, SW-HQ-1 no llega a SW-SUC). Los
  switches solo necesitan llegar a SRV-TI, para NTP y syslog, y ese tráfico no se
  filtra.
- **Invitados (D5):** la VLAN 40 solo puede usar DHCP (incluido el broadcast inicial),
  ping (eco ICMP) hacia su gateway (10.10.40.1) y DNS (UDP 53) hacia SRV-TI. Todo lo
  demás se bloquea, incluida la sucursal.
- **Tráfico interno (D6):** libre entre Administración, Ventas, TI y la sucursal.
- **Pruebas (D7):** en la v2 son fallos esperados B2.2, B2.5, B2.6, B3.7, B4.3, B4.4,
  B4.5 y C4.4. Un éxito en cualquiera de ellas es un defecto. El resto de las fases A,
  B y C debe seguir al 100 %. La B2.2 (PC-VEN hacia PC-INV) se agregó al preparar el
  plan de aplicación: la respuesta de PC-INV se bloquea en la ACL de entrada de la .40,
  así que Invitados queda aislado en los dos sentidos. Pruebas nuevas: PC-TI llega a la
  gestión y entra por SSH; Telnet falla (probado desde PC-TI, para que el fallo no se
  deba a una ACL); Invitados obtiene IP por DHCP y resuelve DNS, pero no carga la web
  interna.
- **Archivo de la v2 (D8):** `diagrams/horizonte-v2.pkt`, copia de la v1. La v1 se
  conserva en la versión del cierre de la fase 2.

### Grupo 2 (configuración)

Valores por defecto aceptados por Karla el 2026-11-11, con los ajustes por el grupo 1.

**Acceso a los equipos**
- **Credenciales (D9):** un usuario personal con privilegio 15 y `enable secret`
  distinto en cada equipo, todas con hash (scrypt si Packet Tracer lo acepta; si no,
  MD5) y `service password-encryption`. Contraseñas exclusivas del laboratorio. Las
  configuraciones de `configurations/v2/` se exportan completas, sin omitir nada.
  Política en `docs/credenciales-v2.md`.
- **SSH (D10):** versión 2, dominio `horizonte.internal`, RSA de 2048 bits, todas las
  líneas VTY que tenga cada equipo (en la v1, 0 a 4 en los routers y 0 a 15 en los
  switches) solo con SSH y `access-class` limitado a 10.10.30.0/24. En el paso de IPv6,
  el equivalente IPv6 en las VTY.
- **Consola (D11):** usuarios locales y 10 minutos de inactividad en la consola y en
  las VTY.
- **Banner (D12):** acceso restringido, sin nombre de la empresa, modelos ni IPs.

**Switching**
- **VLANs (D13):** nativa 999 (NATIVA) y sin uso 998 (SIN-USO), en los tres switches.
- **Trunks (D14):** DTP desactivado y VLANs permitidas:
  R-HQ↔SW-HQ-1: 10, 20, 30, 40, 99 y 999. SW-HQ-1↔SW-HQ-2: 10, 20, 40, 99 y 999.
  R-SUC↔SW-SUC: 50, 99 y 999.
- **EtherChannel (D15):** SW-HQ-1↔SW-HQ-2 (Fa0/1 y Fa0/2), LACP en modo activo en
  ambos extremos, Port-channel 1.
- **STP (D16):** Rapid PVST+ en los tres switches. SW-HQ-1 raíz principal y SW-HQ-2
  secundaria en todas las VLANs de la central.
- **Puertos de acceso (D17):** PortFast, BPDU Guard y port-security (1 MAC, sticky,
  violación `shutdown`) en todos, incluido el de PC-TI (SW-HQ-1 Fa0/11).
- **Puertos sin uso (D18):** apagados, en la VLAN 998, modo acceso, descripción
  "SIN USO". SW-HQ-1 Fa0/11 ya no está sin uso.
- **DHCP snooping (D19):** solo en los switches de acceso: SW-HQ-2 (VLANs 10, 20 y 40)
  y SW-SUC (VLAN 50). El único puerto de confianza es el que va hacia el router
  (Port-channel 1 en SW-HQ-2, Gi0/1 en SW-SUC). Opción 82 desactivada.
- **CDP (D20):** desactivado en los puertos de usuarios (incluido PC-TI) y activo
  entre equipos de red.

**Enrutamiento**
- **OSPF pasivo (D21):** todas las interfaces pasivas por defecto, salvo G0/0/1. Se
  mantienen las sentencias `network`.
- **Autenticación OSPF (D22):** MD5 en G0/0/1 de los dos routers.
- **ACLs (D23):** extendidas con nombre.
  - Invitados, de entrada en R-HQ G0/0/0.40: DHCP del cliente (UDP 68) al servidor
    (UDP 67) con cualquier origen y destino, para incluir el Discover inicial (origen
    0.0.0.0, destino 255.255.255.255); ping (eco ICMP) solo hacia 10.10.40.1; UDP 53
    hacia 10.10.30.10. Todo lo demás se bloquea.
  - Gestión, de salida en G0/0/0.99 de cada router: orígenes 10.10.30.0/24 y
    10.255.255.0/30. Todo lo demás se bloquea.

**Operación**
- **NTP, syslog y marcas de tiempo (D24):** SRV-TI como servidor NTP y syslog de los
  cinco equipos, nivel informativo, marcas de tiempo con milisegundos, zona UTC.
- **Descripciones (D25):** formato `<tipo> a <equipo> <puerto>`, según `conexiones.md`.
- **Búsqueda de nombres (D26):** desactivada en los cinco equipos.
- **SRV-TI (D27):** solo DNS, HTTP, NTP y syslog activos. HTTPS y el resto apagados,
  sin archivos de ejemplo.
- **Orden (D28):** 1. PC-TI. 2. Configuración base y operación. 3. Capa 2. 4. STP y
  EtherChannel. 5. DHCP snooping (repetir la fase A). 6. OSPF. 7. SSH (probar desde
  PC-TI antes de quitar Telnet). 8. ACLs. 9. IPv6. Pruebas después de cada bloque.

**Precauciones al aplicar**
- VLAN nativa: cambiar los dos extremos de cada trunk seguidos (incidente 1).
- EtherChannel: armarlo con Fa0/1 y Fa0/2 apagados.
- OSPF: el pasivo por defecto y la autenticación cortan la vecindad hasta completar
  los dos lados; verificarla al terminar.
- DHCP snooping: la opción 82 y el snooping en switches encadenados pueden dejar sin
  IP a las PCs.
- ACL de Invitados: si la regla de DHCP se limita al gateway, el Discover inicial
  (0.0.0.0 a 255.255.255.255) se bloquea y PC-INV no obtiene IP. Después de aplicarla,
  liberar y renovar la IP de PC-INV y repetir la A3.
- Port-security: si se cambia una PC, el puerto queda en err-disabled.

**Por comprobar en Packet Tracer:** scrypt en las contraseñas, cliente NTP en el 2960,
el Port-channel como puerto de confianza de DHCP snooping y SHA en OSPF (se usa MD5).

## Incidentes encontrados durante la construcción (para el Proyecto 2)

1. VLAN nativa distinta en los dos extremos de un trunk (SW-HQ-2 Fa0/1): STP bloqueó
   la VLAN 99. Resuelto devolviendo la VLAN nativa a 1.
2. Exclusión DHCP con rango entre dos redes (10.10.20.0 a 10.10.40.0) en R-HQ: habría
   dejado sin IP a Ventas. Resuelto con una exclusión por red.
3. Dominios públicos (.com) en los pools DHCP. Resuelto con `horizonte.internal`.
4. Router-on-a-stick con la interfaz física apagada y una subinterfaz en
   `administratively down` (R-HQ y R-SUC). Resuelto con `no shutdown`.
5. Red WAN sin anunciar en OSPF en R-HQ. Resuelto agregando el `network` de la /30.
