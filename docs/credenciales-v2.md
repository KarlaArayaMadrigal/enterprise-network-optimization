# Política de credenciales — Red v2

> Proyecto de portafolio. Comercial Horizonte S.A. es una empresa ficticia y la red es
> una simulación en Cisco Packet Tracer.

Este documento describe qué credenciales tiene la red v2 y cómo se guardan. No contiene
ninguna contraseña.

## Contraseñas exclusivas del laboratorio

Todas las contraseñas de la v2 se crearon solo para esta simulación y no se usan en
ningún otro sistema ni cuenta. Por eso se pueden publicar las configuraciones completas
sin riesgo para nada real.

La v1 usa a propósito la contraseña débil `cisco123`, en texto plano y repetida en todos
los equipos. Es el estado "antes" que documenta la auditoría (`docs/auditoria-v1.md`,
SEC-01 a SEC-03).

## Credenciales de la v2

| Credencial | Equipos | Cómo se guarda en la configuración | ¿Reversible? |
|---|---|---|---|
| Usuario local de administración `karaya` (privilegio 15 en los routers; en los switches, 1 al abrir el `.pkt`: ver nota) | Los 5 | Hash MD5 (tipo 5). Packet Tracer no ofrece scrypt | No |
| `enable secret`, distinto en cada equipo | Los 5 | Hash MD5 (tipo 5) | No |
| Clave de autenticación OSPF (MD5) | R-HQ y R-SUC | **Texto plano** en Packet Tracer: `service password-encryption` no la cifra (en IOS real, tipo 7) | **Sí: se lee directamente** |

- **Hash (tipo 5):** no se puede descifrar, pero sí atacar por fuerza bruta. Por eso las
  contraseñas son robustas.
- **Sal fija (Packet Tracer):** todos los hashes usan la sal `$1$mERr$`, así que la misma
  contraseña da el mismo hash en todos los equipos. Por eso la configuración revela que
  `karaya` tiene la misma contraseña en los cinco. En IOS real la sal es aleatoria.
- **Privilegio en los 2960 (Packet Tracer):** el switch no guarda `privilege 15` en el
  usuario, así que al abrir el `.pkt` `karaya` entra con privilegio 1 y sube a 15 con
  `enable` y el `enable secret` de cada switch. Karla lo aceptó el 2026-11-23 (diferencia
  con la D9). En IOS real, el privilegio se guarda con el usuario.
- **Tipo 7 (en IOS real):** es un cifrado reversible que solo evita que se lea la clave
  de un vistazo. Cualquiera que tenga la configuración puede recuperarla en segundos. En
  Packet Tracer, la clave de OSPF ni siquiera llega a tipo 7: queda en texto plano.
- **Claves RSA de SSH:** no aparecen en la running-config.
- No hay contraseñas de línea en la consola ni en las VTY: usan los usuarios locales.

## Dónde aparecen

- **`configurations/v2/`:** las running-config se exportan completas, sin omitir nada,
  porque son la evidencia de la v2. Contienen los hashes y la clave de OSPF en texto
  plano.
- **`diagrams/horizonte-v2.pkt`:** contiene la configuración completa de los equipos,
  así que quien abra el archivo tiene acceso a la misma información.
- **Fuera del repositorio:** las contraseñas en texto plano se guardan en un gestor de
  contraseñas, fuera del repositorio. Nunca se guardan en un archivo dentro de la
  carpeta del proyecto, aunque esté en `.gitignore`.

## Credenciales expuestas durante el bloque 7

Algunas credenciales del laboratorio se escribieron en los chats de trabajo con Claude
(Claude Code y Claude en claude.ai). Todas eran exclusivas del laboratorio.

| Credencial expuesta | Dónde | Estado |
|---|---|---|
| `proyecto1` y `Packettracer1` (usuario `KarlaA`) | Claude Code | Reemplazadas en el momento; el usuario se borró |
| Última contraseña de `KarlaA` | claude.ai | El usuario se borró de los cinco equipos |
| `CLAVE_TEMPORAL` (usuario de prueba `PRUEBA`) | Claude Code | El usuario se borró |
| `enable secret` de R-SUC (con un texto de más al final) | claude.ai | Cambiado (hash `3lAm…`) |
| `<clave-ospf>` (clave de OSPF del bloque 6, el marcador) | Claude Code | Cambiada |
| Clave de OSPF vigente | Claude Code | En uso. Es pública de todos modos: Packet Tracer la guarda en texto plano y aparece en `configurations/v2/` |

La contraseña de `karaya` y los `enable secret` vigentes de los cinco equipos no se
escribieron en ningún chat. Detalle en `tests/v2-bloque7-acceso-seguro.txt`.

## En una red real

Esto no se haría en una red real. Las configuraciones con credenciales, aunque tengan
hash, no se publicarían ni se guardarían en un repositorio compartido sin control de
acceso. Las claves con cifrado reversible se tratarían como contraseñas en texto plano,
y las contraseñas se guardarían en un gestor de contraseñas o en un servidor de
autenticación centralizado.
