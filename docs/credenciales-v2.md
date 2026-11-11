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
| Usuario local de administración (privilegio 15) | Los 5 | Hash (scrypt, tipo 9, o MD5, tipo 5, según lo que acepte Packet Tracer) | No |
| `enable secret`, distinto en cada equipo | Los 5 | Hash (mismo criterio) | No |
| Clave de autenticación OSPF (MD5) | R-HQ y R-SUC | Cifrado tipo 7 de `service password-encryption` | **Sí** |

- **Hash (tipos 5 y 9):** no se puede descifrar, pero sí atacar por fuerza bruta. Por
  eso las contraseñas son robustas.
- **Tipo 7:** es un cifrado reversible que solo evita que se lea la clave de un vistazo.
  Cualquiera que tenga la configuración puede recuperar la clave de OSPF en segundos.
- **Claves RSA de SSH:** no aparecen en la running-config.
- No hay contraseñas de línea en la consola ni en las VTY: usan los usuarios locales.

## Dónde aparecen

- **`configurations/v2/`:** las running-config se exportan completas, sin omitir nada,
  porque son la evidencia de la v2. Contienen los hashes y la clave de OSPF en tipo 7.
- **`diagrams/horizonte-v2.pkt`:** contiene la configuración completa de los equipos,
  así que quien abra el archivo tiene acceso a la misma información.
- **Fuera del repositorio:** las contraseñas en texto plano se guardan en un gestor de
  contraseñas, fuera del repositorio. Nunca se guardan en un archivo dentro de la
  carpeta del proyecto, aunque esté en `.gitignore`.

## En una red real

Esto no se haría en una red real. Las configuraciones con credenciales, aunque tengan
hash, no se publicarían ni se guardarían en un repositorio compartido sin control de
acceso. Las claves con cifrado reversible se tratarían como contraseñas en texto plano,
y las contraseñas se guardarían en un gestor de contraseñas o en un servidor de
autenticación centralizado.
