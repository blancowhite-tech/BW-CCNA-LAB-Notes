# Laboratorio 2: Trunking 802.1Q y Native VLAN

| | |
|---|---|
| **Fecha** | 28 de octubre de 2026 |
| **Entorno** | Cisco Packet Tracer 9.0.1 |
| **Nivel** | CCNA - Network Access |

## Objetivo

Configurar un enlace troncal entre dos switches, cambiar la VLAN nativa (Native VLAN) y analizar cómo reaccionan CDP y Spanning Tree (STP) ante un desajuste (*Native VLAN mismatch*).

## Topología

- 2 switches Cisco 2960: **SW1** (izquierda) y **SW2** (derecha)
- PC0 y PC1 (VLAN 10) → SW1
- PC2 y PC3 (VLAN 20) → SW2
- Enlace troncal: `GigabitEthernet0/1` (SW1) ↔ `GigabitEthernet0/1` (SW2)

```
 PC0 ──┐                              ┌── PC2
       ├── SW1 ═══ trunk (Gi0/1) ═══ SW2 ─┤
 PC1 ──┘     native VLAN 99             └── PC3
 (VLAN 10)                            (VLAN 20)
```

### Direccionamiento (solo capa 2, sin gateway)

| Dispositivo | VLAN | IP |
|---|---|---|
| PC0 | 10 | 192.168.10.10/24 |
| PC1 | 10 | 192.168.10.11/24 |
| PC2 | 20 | 192.168.20.10/24 |
| PC3 | 20 | 192.168.20.11/24 |

## Configuración paso a paso

### 1. Base (SW1 y SW2)

```cisco
enable
configure terminal
hostname SW1
vlan 10
 name DATOS
exit
vlan 20
 name VOZ
exit
interface fastEthernet 0/10
 switchport mode access
 switchport access vlan 10
exit
interface fastEthernet 0/20
 switchport mode access
 switchport access vlan 20
exit
```

> En SW2 se repite lo mismo cambiando el hostname.

### 2. Trunk (SW1 y SW2)

```cisco
interface gigabitEthernet 0/1
 switchport mode trunk
end
```

### 3. Cambio de la Native VLAN (el reto)

Por defecto la nativa es la VLAN 1. Se crea la VLAN 99 y se asigna como nativa **en ambos extremos**.

```cisco
configure terminal
vlan 99
 name NATIVA
exit
interface gigabitEthernet 0/1
 switchport trunk native vlan 99
end
write memory
```

## Verificación del funcionamiento correcto

`show interfaces trunk` en SW1:

```text
Port        Mode         Encapsulation  Status        Native vlan
Gig0/1      on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gig0/1      1-1005

Port        Vlans allowed and active in management domain
Gig0/1      1,10,20,99

Port        Vlans in spanning tree forwarding state and not pruned
Gig0/1      1,10,20,99
```

| Campo | Lectura |
|---|---|
| `Mode on` | Puerto forzado a trunk con `switchport mode trunk` (no negocia DTP) |
| `Status trunking` | El enlace está operativo |
| `Native vlan 99` | La VLAN sin etiquetar es la 99; debe coincidir en ambos extremos |
| `Vlans allowed on trunk` | Por defecto **todas** (1-1005) |
| `allowed and active` | Solo las que existen en el switch: 1, 10, 20, 99 |

## Troubleshooting: Native VLAN mismatch

Se cambió a propósito la Native VLAN de SW2 de vuelta a la 1, dejando SW1 en la 99.

### 1. Mensajes observados

**CDP** detecta la inconsistencia:

```text
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on GigabitEthernet0/1 (1), with SW1 GigabitEthernet0/1 (99).
```

SW2 (nativa 1) recibe de SW1 (nativa 99) una VLAN distinta de la suya.

**STP** compara el PVID de las BPDU recibidas y, al no coincidir, bloquea **solo las VLANs afectadas** (la 1 y la 99), no todo el puerto. Mensajes típicos de IOS:

```text
%SPANTREE-2-RECV_PVID_ERR: Received 802.1Q BPDU on trunk GigabitEthernet0/1 VLAN1 with inconsistent peer vlan id 99.
%SPANTREE-2-BLOCK_PVID_LOCAL: Blocking GigabitEthernet0/1 on VLAN0001. Inconsistent local vlan.
%SPANTREE-2-BLOCK_PVID_PEER: Blocking GigabitEthernet0/1 on VLAN0099. Inconsistent peer vlan.
```

> Nota: el texto exacto puede variar según versión de IOS / Packet Tracer. Sustituir por la captura real del lab si difiere.

Al corregirlo (nativa 99 en SW2), STP desbloquea:

```text
%SPANTREE-2-UNBLOCK_CONSIST_PORT: Unblocking GigabitEthernet0/1 on VLAN0099. Port consistency restored.
%SPANTREE-2-UNBLOCK_CONSIST_PORT: Unblocking GigabitEthernet0/1 on VLAN0001. Port consistency restored.
```

### 2. Efecto sobre el tráfico

- **Tráfico etiquetado (VLAN 10 y 20):** no se ve afectado; sigue llevando su etiqueta 802.1Q.
- **Tráfico sin etiquetar (la nativa):** SW1 lo envía como VLAN 99 y SW2 lo asigna a la VLAN 1. Se mezclan dominios de broadcast y se pierde conectividad en esas VLANs.

## Lecciones aprendidas (examen)

1. **Native VLAN:** la VLAN cuyo tráfico viaja **sin etiquetar** por un trunk 802.1Q. Por defecto, la 1.
2. **Debe coincidir** en ambos extremos. Si no, CDP avisa y STP bloquea la VLAN afectada.
3. **Aislamiento del fallo:** el desajuste no rompe las VLANs etiquetadas de datos.
4. **Buena práctica de seguridad:** usar una VLAN nativa dedicada (p. ej. 99) que no lleve usuarios ni datos, para dificultar ataques de VLAN hopping por doble etiqueta. Además, restringir las VLANs del trunk con `switchport trunk allowed vlan`.
5. **Sintaxis:** `switchport trunk native vlan <número>`.
6. **Diagnóstico:** `show interfaces trunk`, `show spanning-tree vlan 1` y los logs de CDP.

## Mejoras / siguientes pasos

- En esta topología no hay dos PCs de la misma VLAN en distintos switches, así que los pings de datos no demuestran que el trunk funcione. Poner una PC de cada VLAN en cada switch.
- Probar `switchport trunk allowed vlan 10,20,99` y verificar el efecto.
- Siguiente lab: **Inter-VLAN Routing** (router-on-a-stick).

---
*Blancowhite Tech - Octubre 2026*
