# Laboratorio 2: Trunking 802.1Q y Native VLAN

| | |
|---|---|
| **Fecha** | 28 de octubre de 2026 |
| **Entorno** | Cisco Packet Tracer 9.0.1 |
| **Nivel** | CCNA - Switching |

## Objetivo

Configurar un enlace troncal entre dos switches, cambiar la VLAN nativa (Native VLAN) y analizar el comportamiento y los mensajes de error cuando hay un desajuste.

## Topología

- 2 switches Cisco 2960: **SW1** y **SW2**
- 4 PCs distribuidas en VLAN 10 y VLAN 20
- Enlace troncal: `GigabitEthernet0/1` (SW1) ↔ `GigabitEthernet0/1` (SW2)

## Configuración aplicada

- **VLAN nativa:** cambiada de la VLAN 1 (por defecto) a la **VLAN 99** en ambos switches.
- **Puerto troncal:** `Gig0/1` en `switchport mode trunk`.

### Comandos clave en SW1

```cisco
vlan 99
 name NATIVA
exit
interface gigabitEthernet 0/1
 switchport mode trunk
 switchport trunk native vlan 99
end
write memory
```

> En SW2 se aplica lo mismo (cambiando el hostname).

## Verificación

| Comando | Qué verifica |
|---|---|
| `show interfaces trunk` | Modo trunk y Native VLAN en cada extremo |
| `show vlan brief` | Que la VLAN 99 existe |

## Notas

- Si la Native VLAN no coincide en los dos extremos, el switch avisa con un mensaje de desajuste de CDP/STP (*native VLAN mismatch*).
