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
4. Red optimizada (v2) y validación — siguiente.
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

### Conexiones (cable directo de cobre)

| Origen | Puerto | Destino | Puerto | Uso |
|---|---|---|---|---|
| R-HQ | G0/0/0 | SW-HQ-1 | G0/1 | Trunk router-on-a-stick |
| R-HQ | G0/0/1 | R-SUC | G0/0/1 | Enlace WAN |
| SW-HQ-1 | Fa0/1 | SW-HQ-2 | Fa0/1 | Enlace entre switches 1 |
| SW-HQ-1 | Fa0/2 | SW-HQ-2 | Fa0/2 | Enlace entre switches 2 |
| SW-HQ-1 | Fa0/10 | SRV-TI | Fa0 | VLAN 30 |
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

## Incidentes encontrados durante la construcción (para el Proyecto 2)

1. VLAN nativa distinta en los dos extremos de un trunk (SW-HQ-2 Fa0/1): STP bloqueó
   la VLAN 99. Resuelto devolviendo la VLAN nativa a 1.
2. Exclusión DHCP con rango entre dos redes (10.10.20.0 a 10.10.40.0) en R-HQ: habría
   dejado sin IP a Ventas. Resuelto con una exclusión por red.
3. Dominios públicos (.com) en los pools DHCP. Resuelto con `horizonte.internal`.
4. Router-on-a-stick con la interfaz física apagada y una subinterfaz en
   `administratively down` (R-HQ y R-SUC). Resuelto con `no shutdown`.
5. Red WAN sin anunciar en OSPF en R-HQ. Resuelto agregando el `network` de la /30.
