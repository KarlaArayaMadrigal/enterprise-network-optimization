# Plan de pruebas de conectividad — Red v1

> Proyecto de portafolio. Comercial Horizonte S.A. es una empresa ficticia y la red es
> una simulación en Cisco Packet Tracer.

Este plan valida la red base (v1) completa: DHCP, enrutamiento entre VLANs en cada
sede, OSPF entre sedes y alcance a la red de gestión. La columna "Obtenido" se completa
al ejecutar cada prueba.

## Antes de empezar

- SRV-TI ya tiene su IP fija (10.10.30.10/24, gateway 10.10.30.1).
- Las cuatro PCs están en modo DHCP y ya recibieron dirección (fase A).
- **Pérdida inicial:** en el primer ping hacia un destino nuevo es normal que se pierdan
  uno o dos paquetes mientras se resuelve ARP. En ese caso repite el ping. La prueba
  pasa si la repetición llega al 100 %.
- Los pings desde las PCs se hacen en **Desktop > Command Prompt**.

## Fase A — DHCP

Cada pool excluye de .1 a .20, así que las PCs deben recibir una dirección entre .21 y
.254 de su red. Lo normal es que reciban la .21, pero cualquier dirección de ese rango es
válida. Anota la IP que reciba cada una, porque se usa en la fase B.

| # | PC | Puerto / VLAN | IP esperada | Máscara | Gateway | DNS | IP obtenida | ¿OK? |
|---|---|---|---|---|---|---|---|---|
| A1 | PC-ADM | SW-HQ-2 Fa0/10 / VLAN 10 | 10.10.10.21-.254  | 255.255.255.0 | 10.10.10.1 | 10.10.30.10 | 10.10.10.21 | Sí |
| A2 | PC-VEN | SW-HQ-2 Fa0/11 / VLAN 20 | 10.10.20.21-.254 | 255.255.255.0 | 10.10.20.1 | 10.10.30.10 | 10.10.20.21 | Sí |
| A3 | PC-INV | SW-HQ-2 Fa0/12 / VLAN 40 | 10.10.40.21-.254 | 255.255.255.0 | 10.10.40.1 | 10.10.30.10 | 10.10.40.21 | Sí |
| A4 | PC-SUC | SW-SUC Fa0/10 / VLAN 50 | 10.20.50.21-.254 | 255.255.255.0 | 10.20.50.1 | 10.10.30.10 | 10.20.50.21 | Sí |

Evidencia: `tests/v1-dhcp-pcs.txt` (asignaciones de R-HQ y R-SUC e `ipconfig /all` de las
cuatro PCs). En las cuatro coinciden la máscara, el gateway, el DNS y el dominio, y la
MAC de cada PC coincide con su asignación en el router.

**Comprobación en los routers:** revisa la tabla de asignaciones DHCP de R-HQ (tres
asignaciones) y la de R-SUC (una asignación). Cada IP debe corresponder a la MAC de la
PC correcta.

**Fallo típico:** si una PC recibe una dirección 169.254.x.x, no le llegó respuesta DHCP.
En ese caso, revisa en orden el puerto del switch, los trunks, la subinterfaz del router
y el pool.

## Fase B — Conectividad

En la tabla, `IP-ADM`, `IP-VEN`, `IP-INV` e `IP-SUC` son las direcciones obtenidas en la
fase A.

### B1. Gateway local (cada PC con su router)

| # | Origen | Destino | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| B1.1 | PC-ADM | 10.10.10.1 | Acceso, trunk y subinterfaz .10 | Éxito | Éxito (4/4, 0 % de pérdida) |
| B1.2 | PC-VEN | 10.10.20.1 | Acceso, trunk y subinterfaz .20 | Éxito | Éxito (4/4, 0 % de pérdida) |
| B1.3 | PC-INV | 10.10.40.1 | Acceso, trunk y subinterfaz .40 | Éxito | Éxito (4/4, 0 % de pérdida) |
| B1.4 | PC-SUC | 10.20.50.1 | Acceso, trunk y subinterfaz .50 | Éxito | Éxito (4/4, 0 % de pérdida) |

Evidencia: `tests/v1-ping.txt`, sección B1.

### B2. Entre VLANs en la sede central (router-on-a-stick en R-HQ)

| # | Origen | Destino | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| B2.1 | PC-ADM | IP-VEN | VLAN 10 → 20 | Éxito | |
| B2.2 | PC-VEN | IP-INV | VLAN 20 → 40 | Éxito | |
| B2.3 | PC-ADM | 10.10.30.10 (SRV-TI) | VLAN 10 → 30 | Éxito | |
| B2.4 | PC-VEN | 10.10.30.10 (SRV-TI) | VLAN 20 → 30 | Éxito | |
| B2.5 | PC-INV | 10.10.30.10 (SRV-TI) | VLAN 40 → 30 | Éxito ⚠ | |
| B2.6 | PC-INV | IP-ADM | VLAN 40 → 10 | Éxito ⚠ | |

### B3. Entre sedes (OSPF y enlace WAN)

| # | Origen | Destino | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| B3.1 | PC-SUC | 10.255.255.2 | Interfaz WAN de R-SUC | Éxito | |
| B3.2 | PC-SUC | 10.255.255.1 | Cruce del enlace WAN hasta R-HQ | Éxito | |
| B3.3 | PC-SUC | 10.10.30.10 (SRV-TI) | Ruta OSPF hacia la VLAN 30 | Éxito | |
| B3.4 | PC-SUC | IP-ADM | Ruta OSPF hacia la VLAN 10 | Éxito | |
| B3.5 | PC-ADM | IP-SUC | Ruta OSPF hacia la VLAN 50 (sentido inverso) | Éxito | |
| B3.6 | PC-VEN | IP-SUC | Ventas central → Ventas sucursal | Éxito | |
| B3.7 | PC-INV | IP-SUC | Invitados → sucursal | Éxito ⚠ | |

### B4. Red de gestión (VLAN 99)

| # | Origen | Destino | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| B4.1 | R-HQ | 10.10.99.11 y 10.10.99.12 | Gestión de SW-HQ-1 y SW-HQ-2 (primer intento en `tests/v1-r-hq-router-on-a-stick.txt`, 3/5; falta repetirla al 100 %) | Éxito | |
| B4.2 | R-SUC | 10.20.99.11 | Gestión de SW-SUC y trunk de la sucursal | Éxito | |
| B4.3 | PC-ADM | 10.20.99.11 | Gestión de la sucursal desde la central | Éxito | |
| B4.4 | PC-SUC | 10.10.99.11 | Gestión de la central desde la sucursal | Éxito ⚠ | |
| B4.5 | PC-INV | 10.10.99.12 | Gestión desde la red de invitados | Éxito ⚠ | |

## Pruebas marcadas con ⚠

La v1 no tiene ACLs, así que **se espera que estas pruebas tengan éxito**, aunque en una
red bien diseñada deberían fallar: invitados no debería llegar a las redes internas, y
la gestión no debería ser accesible desde las VLANs de usuarios. Si pasan, es el
comportamiento esperado de la v1. Anótalas como hallazgos para la auditoría (fase 3);
se corrigen en la v2.

Si alguna de estas pruebas **falla**, hay algo que no está en el diseño. Investígalo
antes de seguir.

## Cómo diagnosticar un fallo

| Síntoma | Dónde revisar primero |
|---|---|
| Falla B1 (gateway local) | Puerto de acceso y su VLAN, trunks entre switches, que la VLAN exista en cada switch del camino y subinterfaz del router. La base de VLANs no aparece en las running-config exportadas, así que hay que revisarla en el equipo. |
| B1 pasa y B2 falla | Subinterfaz de la VLAN destino en R-HQ, o gateway y máscara del equipo destino |
| B2 pasa y B3 falla | Vecindad OSPF y redes anunciadas en cada router (incidente 5 de CLAUDE.md) |
| Falla solo hacia SRV-TI | IP, máscara y gateway configurados en el servidor |

## Fuera de este plan (pendiente)

- Resolución de nombres con el DNS de SRV-TI y acceso al servicio web: los servicios DNS
  y web todavía no están configurados.
- IPv6: está solo planificado.

## Evidencia

- Guarda las salidas de texto en `tests/` (por ejemplo, `tests/v1-dhcp-pcs.txt` y
  `tests/v1-ping.txt`).
- Guarda las capturas de IP Configuration de cada PC en `evidence/`.
