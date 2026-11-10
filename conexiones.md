Equipos utilizados:

Router: ISR4331
Router 0: R-HQ
Router 1: R-SUC

Switch: 2960
Switch 0: SW-HQ-1
Switch 1: SW-HQ-2
Switch 2: SW-SUC

Servidor: Server-PT
Server 0: SRV-TI

PCs:
PC 0: PC-ADM
PC 1: PC-VEN
PC 2: PC-INV
PC 3: PC-SUC

Conexiones (cable directo de cobre):

SRV-TI a SW-HQ-1:
Puertos: Fa0 a Fa0/10

R-HQ a SW-HQ-1:
Puertos: G0/0/0 a G0/1

Enlaces entre SW-HQ-1 y SW-HQ-2 (2 cables, serán EtherChannel en la v2):
Cable 1: Fa0/1 a Fa0/1
Cable 2: Fa0/2 a Fa0/2

SW-HQ-2 a PCs:
PC-ADM: Fa0/10 a Fa0
PC-VEN: Fa0/11 a Fa0
PC-INV: Fa0/12 a Fa0

R-HQ a R-SUC (enlace WAN):
Puertos: G0/0/1 a G0/0/1

R-SUC a SW-SUC:
Puertos: G0/0/0 a G0/1

SW-SUC a PC-SUC:
Puertos: Fa0/10 a Fa0