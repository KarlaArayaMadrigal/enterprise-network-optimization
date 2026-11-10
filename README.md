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
| 2. Red base (v1) | Terminada |
| 3. Auditoría de la v1 con Claude Code | Terminada |
| 4. Red optimizada (v2) y validación | Siguiente |
| 5. Documentación final | Pendiente |

**Fase 2:** la red base está configurada y verificada según `docs/plan-pruebas-v1.md`:
DHCP en las cuatro PCs (fase A), conectividad entre VLANs, entre sedes por OSPF y hacia
la red de gestión (fase B), y servicios DNS y web en SRV-TI (fase C). La evidencia está
en `tests/` y `evidence/`. Las pruebas confirman que la v1 funciona, pero también que no
tiene controles de acceso: la red de Invitados llega a las redes internas y a la gestión.
Esos hallazgos son el punto de partida de la auditoría (fase 3).

**Fase 3:** la auditoría de la v1 está en `docs/auditoria-v1.md`: 21 hallazgos (5 de
severidad alta, 8 media, 7 baja y 1 informativa), cada uno con su evidencia y una
recomendación para la v2. Después de la auditoría, en `docs/comparacion-auditoria-v1.md`
se comparó con las debilidades introducidas a propósito en la v1: encontró 11 de 12. A
partir de esa comparación se agregaron dos hallazgos (DIS-01, IPv6 no implementado, y
SW-06, CDP en los puertos de usuarios), marcados como tales en la auditoría, y SW-05
subió de Baja a Media.

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

El trabajo con IA tuvo dos etapas: la v1 la construyó Karla en la CLI de Packet Tracer,
con Claude (claude.ai) como guía paso a paso; después se incorporó Claude Code para las
pruebas, la documentación y la auditoría.

Claude Code se usa como asistente: revisa configuraciones, propone planes y explica
conceptos. Los comandos en Packet Tracer, las pruebas y las decisiones son de Karla.
Cada tarea queda registrada en `docs/uso-de-ia.md`: qué se pidió, qué propuso la IA,
qué se verificó y qué se corrigió o rechazó.
