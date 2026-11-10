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
| B2.1 | PC-ADM | IP-VEN | VLAN 10 → 20 | Éxito | Éxito (4/4, 0 % de pérdida, TTL=127). El primer intento dio 3/4 por ARP |
| B2.2 | PC-VEN | IP-INV | VLAN 20 → 40 | Éxito | Éxito (4/4, 0 % de pérdida, TTL=127). El primer intento dio 3/4 por ARP |
| B2.3 | PC-ADM | 10.10.30.10 (SRV-TI) | VLAN 10 → 30 | Éxito | Éxito (4/4, 0 % de pérdida, TTL=127). El primer intento dio 3/4 por ARP |
| B2.4 | PC-VEN | 10.10.30.10 (SRV-TI) | VLAN 20 → 30 | Éxito | Éxito (4/4, 0 % de pérdida, TTL=127) en los dos intentos |
| B2.5 | PC-INV | 10.10.30.10 (SRV-TI) | VLAN 40 → 30 | Éxito ⚠ | Éxito (4/4, 0 % de pérdida, TTL=127). Hallazgo para la auditoría |
| B2.6 | PC-INV | IP-ADM | VLAN 40 → 10 | Éxito ⚠ | Éxito (4/4, 0 % de pérdida, TTL=127). Hallazgo para la auditoría |

Evidencia: `tests/v1-ping.txt`, sección B2.

### B3. Entre sedes (OSPF y enlace WAN)

| # | Origen | Destino | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| B3.1 | PC-SUC | 10.255.255.2 | Interfaz WAN de R-SUC | Éxito | Éxito (4/4, 0 % de pérdida, TTL=255) |
| B3.2 | PC-SUC | 10.255.255.1 | Cruce del enlace WAN hasta R-HQ | Éxito | Éxito (4/4, 0 % de pérdida, TTL=254) |
| B3.3 | PC-SUC | 10.10.30.10 (SRV-TI) | Ruta OSPF hacia la VLAN 30 | Éxito | Éxito (4/4, 0 % de pérdida, TTL=126) |
| B3.4 | PC-SUC | IP-ADM | Ruta OSPF hacia la VLAN 10 | Éxito | Éxito (4/4, 0 % de pérdida, TTL=126) |
| B3.5 | PC-ADM | IP-SUC | Ruta OSPF hacia la VLAN 50 (sentido inverso) | Éxito | Éxito (4/4, 0 % de pérdida, TTL=126) |
| B3.6 | PC-VEN | IP-SUC | Ventas central → Ventas sucursal | Éxito | Éxito (4/4, 0 % de pérdida, TTL=126) |
| B3.7 | PC-INV | IP-SUC | Invitados → sucursal | Éxito ⚠ | Éxito (4/4, 0 % de pérdida, TTL=126). Hallazgo para la auditoría |

Evidencia: `tests/v1-ping.txt`, sección B3.

### B4. Red de gestión (VLAN 99)

| # | Origen | Destino | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| B4.1 | R-HQ | 10.10.99.11 y 10.10.99.12 | Gestión de SW-HQ-1 y SW-HQ-2 (primer intento en `tests/v1-r-hq-router-on-a-stick.txt`, 3/5) | Éxito | Éxito (5/5 hacia .11 y .12 en la repetición). Los intentos anteriores dieron 3/5 por pérdida inicial (ARP) |
| B4.2 | R-SUC | 10.20.99.11 | Gestión de SW-SUC y trunk de la sucursal | Éxito | Éxito (5/5 en la repetición). El primer intento dio 3/5 por pérdida inicial (ARP) |
| B4.3 | PC-ADM | 10.20.99.11 | Gestión de la sucursal desde la central | Éxito | Éxito (4/4, 0 % de pérdida, TTL=253) |
| B4.4 | PC-SUC | 10.10.99.11 | Gestión de la central desde la sucursal | Éxito ⚠ | Éxito (4/4, 0 % de pérdida, TTL=253). Hallazgo para la auditoría |
| B4.5 | PC-INV | 10.10.99.12 | Gestión desde la red de invitados | Éxito ⚠ | Éxito (4/4, 0 % de pérdida, TTL=254). Hallazgo para la auditoría |

Evidencia: `tests/v1-ping.txt`, sección B4.

## Fase C — Servicios DNS y web en SRV-TI

Registros DNS del diseño (CLAUDE.md), ambos de tipo A hacia 10.10.30.10:
`srv-ti.horizonte.internal` (nombre del equipo) y `www.horizonte.internal` (servicio web).

### C0. Configuración en SRV-TI (pestaña Services)

- **DNS:** activar el servicio (en Server-PT viene apagado) y crear los dos registros
  como A Record, con el nombre completo (incluido el dominio). Comprobar que ambos
  aparecen en la tabla de registros.
- **HTTP:** comprobar que el servicio está activo. Editar `index.html` para que la
  página identifique al servidor (por ejemplo, "Comercial Horizonte S.A. (empresa
  ficticia) — SRV-TI"), de modo que en C1 y C4 se reconozca que responde este servidor.

Evidencia: `evidence/v1-c0-srv-ti-dns.png` (servicio DNS y registros),
`evidence/v1-c0-srv-ti-http.png` (servicios HTTP y HTTPS) y
`evidence/v1-c0-srv-ti-index.png` (contenido de `index.html`).

### Pruebas

El orden va de lo que menos depende de otras cosas a lo que más depende: primero web
por IP (sin DNS), luego resolución de nombres, luego ping y web por nombre. La
conectividad IP hacia SRV-TI ya está probada (B2.3, B2.4 y B3.3).

| # | Origen | Qué hacer | Qué valida | Esperado | Obtenido |
|---|---|---|---|---|---|
| C1.1 | PC-ADM | Web Browser → `http://10.10.30.10` | Servicio HTTP, sin DNS | Carga la página de SRV-TI | Éxito: carga la página de SRV-TI (captura) |
| C2.1 | PC-ADM | `nslookup www.horizonte.internal` | DNS en la misma sede | Servidor 10.10.30.10, dirección 10.10.30.10 | Éxito: servidor 10.10.30.10, dirección 10.10.30.10 |
| C2.2 | PC-VEN | `nslookup srv-ti.horizonte.internal` | Segundo registro, otra VLAN | Servidor 10.10.30.10, dirección 10.10.30.10 | Éxito: servidor 10.10.30.10, dirección 10.10.30.10 |
| C2.3 | PC-SUC | `nslookup www.horizonte.internal` | DNS entre sedes | Servidor 10.10.30.10, dirección 10.10.30.10 | Éxito: servidor 10.10.30.10, dirección 10.10.30.10 |
| C2.4 | PC-ADM | `nslookup noexiste.horizonte.internal` | Prueba negativa: solo responde lo configurado | Error: nombre no encontrado | Éxito: `Non-existent domain` |
| C3.1 | PC-ADM | `ping srv-ti.horizonte.internal` | El cliente usa el DNS | `Pinging 10.10.30.10`, 4/4, TTL=127 | Éxito (`Pinging 10.10.30.10`, 4/4, 0 % de pérdida, TTL=127) |
| C3.2 | PC-SUC | `ping srv-ti.horizonte.internal` | Ídem, entre sedes | `Pinging 10.10.30.10`, 4/4, TTL=126 | Éxito (`Pinging 10.10.30.10`, 4/4, 0 % de pérdida, TTL=126) |
| C4.1 | PC-ADM | Web Browser → `http://www.horizonte.internal` | Web por nombre | La misma página que en C1.1 | Éxito: carga la página de SRV-TI (captura) |
| C4.2 | PC-VEN | Web Browser → `http://www.horizonte.internal` | Ídem, VLAN 20 | La misma página que en C1.1 | Éxito: carga la página de SRV-TI (captura) |
| C4.3 | PC-SUC | Web Browser → `http://www.horizonte.internal` | Web por nombre entre sedes | La misma página que en C1.1 | Éxito: carga la página de SRV-TI (captura) |
| C4.4 | PC-INV | Web Browser → `http://www.horizonte.internal` | Invitados accede a la web interna | Éxito ⚠ | Éxito: carga la página de SRV-TI (captura). Hallazgo para la auditoría |

Notas:
- `nslookup` y `ping` se ejecutan en **Desktop > Command Prompt**; el navegador está en
  **Desktop > Web Browser**.
- Usar siempre el nombre completo. No está comprobado que las PCs de Packet Tracer
  agreguen el dominio recibido por DHCP a un nombre corto; si se prueba `ping srv-ti`,
  se anota como observación, no como criterio de la prueba.
- Si `nslookup` no existe en la versión de Packet Tracer, C3 cubre la resolución de
  nombres de forma indirecta.
- La primera consulta puede fallar por ARP, igual que en los pings: se repite y la
  prueba pasa si la repetición tiene éxito.

Evidencia: salidas de texto (C2 y C3) en `tests/v1-dns-web.txt`. Capturas en `evidence/`:
`v1-c1-web-por-ip.png` (C1.1), `v1-c2-dns-nombre-inexistente.png` (C2.4) y
`v1-c4-web-por-nombre-pc-adm.png`, `-pc-ven.png`, `-pc-suc.png` y `-pc-inv.png`
(C4.1 a C4.4).

### Observaciones para la auditoría (fase C)

- **C4.4 ⚠:** la red de Invitados accede a la web interna de SRV-TI.
- **Valores por defecto sin revisar en SRV-TI (hallazgo menor):** HTTPS está activado
  aunque no forma parte del diseño, y siguen en el servidor los archivos de ejemplo de
  Packet Tracer (`helloworld.html`, `image.html`, `copyrights.html` y el logo).

### Cómo diagnosticar un fallo en la fase C

| Síntoma | Dónde revisar primero |
|---|---|
| Falla C1 (web por IP) | Servicio HTTP de SRV-TI. La red no es el problema: B2 y B3 pasaron |
| C1 pasa y C2 hace timeout | Servicio DNS apagado, o la PC tiene otro DNS (`ipconfig /all`) |
| C2 responde que el nombre no existe | Registro mal escrito: nombre completo, tipo A e IP |
| C2 pasa y C4 falla | URL mal escrita en el navegador o servicio HTTP apagado |
| C2.4 devuelve una IP | Hay un registro de más en la tabla DNS |

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

- IPv6: está solo planificado.

## Evidencia

- Guarda las salidas de texto en `tests/` (por ejemplo, `tests/v1-dhcp-pcs.txt` y
  `tests/v1-ping.txt`).
- Guarda las capturas de IP Configuration de cada PC en `evidence/`.
