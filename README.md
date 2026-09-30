# Infraestructura 2 – VPN IPsec Site-to-Site

## Video demostrativo

[Ver video demostrativo](COLOCAR_AQUI_EL_ENLACE_DEL_VIDEO)

---

## Propósito del laboratorio

El propósito de esta práctica es implementar una infraestructura de red en GNS3 que permita establecer comunicación segura entre una red de usuarios y una red de servidores mediante una VPN IPsec Site-to-Site.

La infraestructura utiliza un router Cisco y un firewall FortiGate como peers de la VPN. El usuario pertenece a la red 10.8.27.0/25 y el servidor web a la red 172.8.27.0/28. Ambas redes se comunican a través del túnel VPN.

También se implementaron servicios y configuraciones como NAT, DHCP, HTTPS y direccionamiento de red. Finalmente, se realizaron pruebas para comprobar que el usuario puede comunicarse con el servidor cuando el tráfico de la VPN está habilitado y que la comunicación se interrumpe al deshabilitarlo.

---

## Topología

La infraestructura está compuesta por:

- 1 router Cisco R1.
- 1 firewall FortiGate.
- 1 segmento WAN/ISP.
- 1 switch para interconectar los peers con el ISP.
- 1 usuario en VLAN 10.
- 1 servidor web con HTTPS.
- 1 VPN IPsec Site-to-Site entre Cisco y FortiGate.

### Diagrama de la topología

![Topología de Infraestructura 2](Diagramas/Topologia-Infraestructura-2.png)

---

## Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP | Función |
|---|---|---|---|
| R1 | FastEthernet0/0 | 10.8.27.1/25 | Gateway de Usuarios |
| R1 | FastEthernet1/0 | 192.168.42.82/24 | WAN / ISP |
| Usuario | DHCP | 10.8.27.2/25 | Cliente VLAN 10 |
| FortiGate | port1 | 192.168.42.6/24 | WAN / ISP |
| FortiGate | port3 | 172.8.27.1/28 | Gateway de Servidores |
| Web Server | ens3 | 172.8.27.2/28 | Servidor HTTPS |

> Las direcciones 192.168.42.0/24 representan el segmento WAN/ISP dentro del entorno de laboratorio GNS3.

---

## VPN IPsec Site-to-Site

Se configuró una VPN IPsec Site-to-Site entre R1 y el FortiGate.

**Peer Cisco:** 192.168.42.82  
**Peer FortiGate:** 192.168.42.6

Red protegida del lado Cisco:

`10.8.27.0/25`

Red protegida del lado FortiGate:

`172.8.27.0/28`

La VPN permite que los dispositivos de ambas redes puedan comunicarse de forma segura a través del túnel IPsec.
