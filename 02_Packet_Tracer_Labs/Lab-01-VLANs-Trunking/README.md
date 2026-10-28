# Laboratorio 1: Configuración básica de VLANs y Trunking

| | |
|---|---|
| **Fecha** | 28 de octubre de 2026 |
| **Entorno** | Cisco Packet Tracer 9.0.1 (Kali Linux VM) |
| **Nivel** | CCNA - Switching |

## Objetivo

Crear dos VLANs (Datos y Voz), asignar puertos de acceso, configurar un enlace troncal (trunk) entre dos switches y verificar la conectividad intra-VLAN y el aislamiento inter-VLAN.

## Topología

- 2 switches Cisco 2960: **SW1** (izquierda) y **SW2** (derecha)
- 4 PCs:
  - PC0 (VLAN 10) → SW1
  - PC1 (VLAN 10) → SW1
  - PC2 (VLAN 20) → SW2
  - PC3 (VLAN 20) → SW2
- Enlace troncal: `GigabitEthernet0/1` (SW1) ↔ `GigabitEthernet0/1` (SW2)

```
 PC0 ──┐                          ┌── PC2
       ├── SW1 ═══ trunk ═══ SW2 ─┤
 PC1 ──┘     Gi0/1      Gi0/1     └── PC3
  (VLAN 10)                         (VLAN 20)
```

## Direccionamiento IP

| Dispositivo | VLAN | Dirección IP | Máscara | Gateway |
|---|---|---|---|---|
| PC0 | 10 (DATOS) | 192.168.10.10 | 255.255.255.0 | N/A |
| PC1 | 10 (DATOS) | 192.168.10.11 | 255.255.255.0 | N/A |
| PC2 | 20 (VOZ) | 192.168.20.10 | 255.255.255.0 | N/A |
| PC3 | 20 (VOZ) | 192.168.20.11 | 255.255.255.0 | N/A |

## Configuración (plantilla para SW1 y SW2)

> En SW2 solo cambia el `hostname`.

```cisco
enable
configure terminal
hostname SW1

! Creación de VLANs
vlan 10
 name DATOS
exit
vlan 20
 name VOZ
exit

! Asignación de puertos de acceso
interface fastEthernet 0/10
 switchport mode access
 switchport access vlan 10
exit
interface fastEthernet 0/20
 switchport mode access
 switchport access vlan 20
exit

! Enlace troncal
interface gigabitEthernet 0/1
 switchport mode trunk
exit

! Guardar configuración
end
write memory
```

## Comandos de verificación

| Comando | Qué verifica |
|---|---|
| `show vlan brief` | Creación de VLANs y puertos de acceso asignados |
| `show interfaces trunk` | Que Gi0/1 está en modo trunking y qué VLANs permite |
| `ping` desde el Command Prompt de las PCs | Conectividad |

## Resultados

### Ping intra-VLAN

- `PC0 → 192.168.10.11 (PC1)`: **exitoso**.
- Se puede requerir repetir el ping: el primer paquete se pierde mientras ARP resuelve la MAC.

### Ping inter-VLAN

- `PC0 → 192.168.20.10 (PC2)`: **fallido** (`Destination host unreachable` / `Request timed out`).
- Es el resultado esperado: sin un router o switch L3 que haga *inter-VLAN routing*, las VLANs están aisladas (además son subredes distintas).

## Lecciones aprendidas

- **Trunk:** para que varias VLANs crucen entre switches, el puerto debe estar en modo trunk. En modo access solo pasa el tráfico de la VLAN asignada a ese puerto (VLAN 1 por defecto).
- **Puertos:** usar puertos Gigabit para los troncales evita cuellos de botella.
- **ARP en Packet Tracer:** el primer ping suele fallar mientras se resuelven las MACs. Es normal.

## Mejoras / siguientes pasos

- En esta topología PC0 y PC1 están en el mismo switch, por lo que su ping **no atraviesa el trunk**. Para probarlo de verdad, poner una PC de cada VLAN en cada switch (p. ej. PC1 → SW2 en VLAN 10) y repetir el ping.
- Cambiar la VLAN nativa y restringir VLANs permitidas: `switchport trunk native vlan 99` / `switchport trunk allowed vlan 10,20`.
- Siguiente lab: **Inter-VLAN Routing** (router-on-a-stick).
