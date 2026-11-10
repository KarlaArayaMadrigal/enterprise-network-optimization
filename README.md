# Diseño y optimización de una red empresarial

> **Proyecto personal de portafolio, en construcción.**
> Comercial Horizonte S.A. es una **empresa ficticia** y la red es una **simulación en
> Cisco Packet Tracer**. Este proyecto no corresponde a experiencia en una empresa real.

Proyecto de Karla Araya (ingeniera informática, CCNA) para demostrar conocimientos
prácticos de redes y el uso de Claude Code como asistente en tareas de infraestructura.

## Qué es

Una red de dos sedes (central y sucursal) para una pyme ficticia, construida en dos
versiones:

- **v1:** red base, armada como la haría una pyme "con prisa": funciona, pero no aplica
  buenas prácticas. Es intencional.
- **v2:** la misma red después de una auditoría, con las mejoras aplicadas y validadas.

El objetivo final es comparar ambas versiones y documentar el proceso, incluido el uso
de IA.

## Tecnologías

- Router-on-a-stick con subinterfaces 802.1Q (VLANs 10, 20, 30, 40, 50 y 99)
- DHCP en los routers, con exclusiones por red
- OSPF de área única entre sedes
- Switches 2960 con trunks y VLAN de gestión
- Servidor DNS y web con IP fija

| Equipo | Rol |
|---|---|
| R-HQ, R-SUC (ISR4331) | Routers de sede central y sucursal: enrutamiento entre VLANs, DHCP y OSPF |
| SW-HQ-1, SW-HQ-2, SW-SUC (2960) | Switches de distribución y acceso |
| SRV-TI | Servidor DNS y web |
| PC-ADM, PC-VEN, PC-INV, PC-SUC | PCs de prueba |

## Estado

| Fase | Estado |
|---|---|
| 1. Diseño | Terminada |
| 2. Red base (v1) | En curso |
| 3. Auditoría de la v1 con Claude Code | Pendiente |
| 4. Red optimizada (v2) y validación | Pendiente |
| 5. Documentación final | Pendiente |

**Fase 2:** routers y switches configurados, con vecindad OSPF FULL entre sedes. Ya están
verificados el DHCP de las cuatro PCs y su conectividad con el gateway local
(`tests/dhcp-pcs.txt` y `tests/ping-v1.txt`). Faltan las pruebas entre VLANs, entre
sedes y de gestión, además de los servicios DNS y web de SRV-TI. El plan de pruebas
está en `docs/plan-pruebas-v1.md`.

## Aviso sobre credenciales

Las contraseñas que aparecen en `configurations/v1/` son **credenciales de laboratorio
débiles a propósito**. Forman parte de los defectos intencionales de la v1, no se usan
en ningún sistema real y se corregirán en la v2.

## Estructura

```
├── conexiones.md      tabla de conexiones física entre equipos
├── diagrams/          topología en Packet Tracer (.pkt) y notas
├── configurations/
│   ├── v1/            running-config de cada equipo (red base)
│   └── v2/            running-config después de las mejoras
├── tests/             salidas de comandos usadas como evidencia
├── evidence/          capturas de pantalla
└── docs/              plan de pruebas, auditoría y registro de uso de IA
```

## Uso de IA

Claude Code se usa como asistente: revisa configuraciones, propone planes y explica
conceptos. Los comandos en Packet Tracer, las pruebas y las decisiones son de Karla.
Cada tarea queda registrada en `docs/uso-de-ia.md`: qué se pidió, qué propuso la IA,
qué se verificó y qué se corrigió o rechazó.
