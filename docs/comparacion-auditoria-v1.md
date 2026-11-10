# Comparación de la auditoría con las debilidades intencionales de la v1

> Proyecto de portafolio. Comercial Horizonte S.A. es una empresa ficticia y la red es
> una simulación en Cisco Packet Tracer.

La v1 la configuró Karla directamente en la CLI de Packet Tracer, con Claude (claude.ai)
como guía paso a paso; Claude Code se incorporó después, para las pruebas, la
documentación y la auditoría. Las debilidades de la v1 se definieron intencionalmente
durante esa construcción. Esta tabla se armó después de la auditoría, a partir de esas
notas, para medir su cobertura; no es una lista independiente previa. Las observaciones
propias de Karla anteriores a la auditoría están en docs/uso-de-ia.md (entradas
«Pruebas pendientes de la fase B y resultados esperados de la B2» y «Pruebas B4: red de
gestión»).

## Debilidades introducidas a propósito

| Debilidad | ¿La encontró la auditoría? |
|---|---|
| Acceso remoto por Telnet | Sí (SEC-01) |
| `enable password`, sin cifrar y repetida | Sí (SEC-02, SEC-03) |
| `no service password-encryption` | Sí (SEC-03) |
| VLAN nativa 1 y todas las VLANs permitidas en los trunks | Sí (SW-02) |
| Sin port security | Sí (SW-03) |
| Puertos sin uso activos | Sí (SW-01) |
| Dos enlaces entre switches sin EtherChannel | Sí (SW-04) |
| OSPF sin interfaces pasivas | Sí (RT-01) |
| Interfaces sin descripción | Sí (OP-02) |
| Consola sin contraseña | Sí (SEC-04) |
| `line vty 5 15` sin contraseña | Sí (INFO-01) |
| IPv6 planificado en el diseño pero no implementado | No (se agregó después como DIS-01) |

Cobertura: 11 de 12.

## Errores no intencionales que encontró la auditoría

| ID | Hallazgo | Verificación |
|---|---|---|
| OP-03 | SW-HQ-1 sin `no ip domain-lookup`, a diferencia del resto | Karla lo verificó con `show running-config` en SW-HQ-1 |
| OP-04 | SRV-TI con HTTPS activado y archivos de ejemplo de Packet Tracer | Captura `evidence/v1-c0-srv-ti-http.png` |

## Hallazgos que no se habían sembrado

| ID | Hallazgo | Nota |
|---|---|---|
| SEG-01 | Sin ACLs: Invitados llega a las redes internas y a la sucursal | Anticipado en el plan de pruebas (⚠) y observado por Karla en la B2 antes de la auditoría |
| SEG-02 | Gestión accesible desde VLANs de usuarios | Anticipado en el plan de pruebas (⚠) y observado por Karla en la B4 antes de la auditoría |
| RT-02 | OSPF sin autenticación | |
| SW-05 | Sin DHCP snooping | |
| SEC-05 | Sin banner de aviso legal | |
| OP-01 | Logs sin marca de tiempo, sin NTP y sin syslog | |
| SW-06 | CDP activo en los puertos de usuarios | Agregado después de la comparación, a partir de una pregunta abierta |

Partes de hallazgos sembrados que van más allá de la debilidad original:
- SEC-03: VTY con contraseña compartida y sin usuarios locales.
- SEC-04: sin `exec-timeout` en la consola ni en las VTY.
- SW-02: DTP activo (falta `switchport nonegotiate`).
- SW-03: puertos de acceso sin PortFast ni BPDU Guard.
- SW-04: STP sin puente raíz definido y PVST en lugar de Rapid PVST+.

## Decisiones sobre las preguntas abiertas

- **IPv6:** se registra como hallazgo DIS-01, de severidad Baja, marcado como agregado
  después de la comparación. Es una brecha entre el diseño y lo construido, no un riesgo
  de seguridad. Se corrigió en la auditoría la frase "Sin diferencias" de la comparación
  con el diseño. La cobertura de esta comparación sigue siendo 11 de 12, porque la
  auditoría original no lo encontró.
- **CDP:** se registra como hallazgo SW-06, de severidad Baja. `show cdp interface` en
  SW-HQ-2 Fa0/12 (Invitados) confirmó que está activo (`tests/v1-sw-hq-2-cdp.txt`).
  Recomendación: desactivarlo en los puertos de usuarios y mantenerlo solo entre equipos
  de red.
- **SW-05:** sube de Baja a Media. Según la escala de la auditoría, debilita la red
  frente a ataques en la LAN, y la VLAN 40 conecta equipos que no se controlan.
