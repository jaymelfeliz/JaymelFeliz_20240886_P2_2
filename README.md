# Documentación Oficial del Laboratorio: VPN IPsec Site-to-Site (FortiGate a Cisco 7200)

[![GitHub release](https://img.shields.io/badge/Status-Completed-success.svg)](#) [![License](https://img.shields.io/badge/License-MIT-blue.svg)](#)

---

## 📹 Video Demostrativo

VIDEO: https://youtu.be/D_AajzrgN5M

> **Nota:** Haz clic en el enlace superior.

---

## 🎯 Propósito del Laboratorio

El objetivo principal de este laboratorio es implementar y validar un **túnel VPN IPsec Site-to-Site** para interconectar de forma segura la red LAN de una sede corporativa protegida por un **FortiGate VM** (FortiOS) con la red LAN de un servidor remoto gestionado por un **Router Cisco 7200** (IOS) en un entorno emulado en **GNS3**.

### Objetivos Específicos:
- Establecer conectividad de Capa 3 entre las interfaces WAN de ambos cortafuegos/routers.
- Negociar adecuadamente la **Fase 1 (IKEv1 ISAKMP)** y la **Fase 2 (Quick Mode / IPsec SA)** bajo restricciones específicas de algoritmos criptográficos.
- Configurar el enrutamiento estático adecuado a través de la interfaz de túnel VPN (`VPN_CISCO`).
- Garantizar que las políticas de seguridad (Firewall Policies) y las listas de control de acceso (ACLs) permitan el flujo de datos sin alterar las direcciones IP originales (NAT desactivado).
- Validar la comunicación end-to-end entre la **PC1** (`192.168.86.10`) y el **Servidor Web** (`172.20.24.2`).

---

## 🌐 Topología de Red

La arquitectura física/lógica interconecta dos sedes a través de un enlace WAN punto a punto en la subred `202.40.88.0/30`.
MAQUETACION

```text
       [ LAN SEDE A ]                                                                [ LAN SEDE B ]
    PC1 (VPCS) 192.168.86.10/25                                                  Servidor Web 172.20.24.2/28
            │                                                                              │
            │ (port3: 192.168.86.1)                                                        │ (Fa1/0: 172.20.24.1)
  ┌─────────┴─────────┐                                                          ┌─────────┴─────────┐
  │   FortiGate-A     │                                                          │   Cisco 7200      │
  │   (FortiOS VM)    │                                                          │   (c7200-advent)  │
  └─────────┬─────────┘                                                          └─────────┬─────────┘
            │ (port2: 202.40.88.1)                                                         │ (Fa2/0: 202.40.88.2)
            └───────────────────────[ ENLACE WAN 202.40.88.0/30 ]──────────────────────────┘
                                      Túnel IPsec: VPN_CISCO
```

---
LOGICO:
<img width="538" height="473" alt="image" src="https://github.com/user-attachments/assets/83244ddf-aed8-447a-bff3-175920d6774d" />

## 📊 Tabla de Direccionamiento

| Dispositivo | Interfaz | Dirección IP / Máscara | Rol / Descripción | Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **FortiGate-A** | `port2` | `202.40.88.1/30` | Interfaz WAN (Hacia Cisco) | N/A |
| **FortiGate-A** | `port3` | `192.168.86.1/25` | Interfaz LAN (Puerta de enlace PC1) | N/A |
| **FortiGate-A** | `VPN_CISCO` | Dynamic / P-t-P | Interfaz Virtual de Túnel IPsec | N/A |
| **Cisco 7200** | `Fa2/0` | `202.40.88.2/30` | Interfaz WAN (Hacia FortiGate) | N/A |
| **Cisco 7200** | `Fa1/0` | `172.20.24.1/28` | Interfaz LAN (Puerta de enlace Servidor)| N/A |
| **PC1 (VPCS)** | `eth0` | `192.168.86.10/25` | Cliente LAN Sede A | `192.168.86.1` |
| **Servidor** | `ens3` | `172.20.24.2/28` | Servidor Destino Sede B | `172.20.24.1` |

---

## ⚙ Funcionamiento de la Configuración

La VPN se basa en el protocolo **IKEv1** dividido en dos fases fundamentales:

1. **Fase 1 (ISAKMP / Main Mode):** Autentica ambos extremos y establece un canal cifrado seguro. Debido a las limitaciones de la licencia/imagen VM del FortiGate, la negociación se fijó en **DES-SHA256** con **Diffie-Hellman Group 14 (2048-bit)** y clave precompartida (`ClaveSegura2024`).
2. **Fase 2 (Quick Mode / IPsec SA):** Negocia los parámetros de encriptación del tráfico de datos (`DES` + `SHA1`) y los **Selectores de Tráfico (Proxy ID)**:
   * **Origen (Local):** `192.168.86.0/25`
   * **Destino (Remoto):** `172.20.24.0/28`
3. **Enrutamiento:** FortiGate dirige todo el tráfico destinado a la red `172.20.24.0/28` a través de la interfaz de software `VPN_CISCO`.
4. **Tráfico de Interés (Cisco):** El router Cisco utiliza la **ACL 101** vinculada a un `crypto map` en su interfaz `Fa2/0` para capturar el tráfico que regresa hacia `192.168.86.0/25` e inyectarlo en el túnel.

---

## 🛠️ Configuraciones Implementadas

### 1. FortiGate-A (GUI)
<img width="1598" height="868" alt="image" src="https://github.com/user-attachments/assets/e659c58c-8313-428d-8d1b-7fb17fe196dc" />

<img width="1610" height="877" alt="image" src="https://github.com/user-attachments/assets/13add52f-f7f2-4ff4-bb1b-190e80516039" />
<img width="1604" height="276" alt="image" src="https://github.com/user-attachments/assets/f7fcde36-6660-48b3-b838-4869ca7a59d6" />

### 2. Router Cisco 7200 (CLI)

```text
configure terminal

crypto isakmp policy 10
 encr des
 hash sha256
 authentication pre-share
 group 14
 exit

crypto isakmp key ClaveSegura2024 address 202.40.88.1

crypto ipsec transform-set TS_VPN esp-des esp-sha-hmac
 mode tunnel
 exit

crypto map CM_FORTI 10 ipsec-isakmp
 set peer 202.40.88.1
 set transform-set TS_VPN
 match address 101
 exit

interface FastEthernet2/0
 ip address 202.40.88.2 255.255.255.252
 crypto map CM_FORTI
 no shutdown
 exit

access-list 101 permit ip 172.20.24.0 0.0.0.15 192.168.86.0 0.0.0.127

end
write memory
```

---

## 🛡️ Políticas de Seguridad

Para garantizar que el tráfico no sea bloqueado ni alterado por el cortafuegos, se configuraron dos políticas esenciales en el FortiGate **sin NAT**:

```text
config firewall policy
    edit 1
        set name "LAN_to_VPN"
        set srcintf "port3"
        set dstintf "VPN_CISCO"
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 2
        set name "VPN_to_LAN"
        set srcintf "VPN_CISCO"
        set dstintf "port3"
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

---

## 🔬 Validación y Pruebas

### 1. Estado de Fase 1 (IKE SA) en Router Cisco
```text
Cisco7200# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
202.40.88.2     202.40.88.1     QM_IDLE           1001 ACTIVE
```
<img width="666" height="188" alt="image" src="https://github.com/user-attachments/assets/c98c51a9-4706-48c4-b2e5-c9b619e4ba85" />

### 2. Estado de Fase 2 (IPsec SA) en FortiGate
```text
# diagnose vpn tunnel list name VPN_CISCO
name=VPN_CISCO ver=1 serial=1 202.40.88.1:0->202.40.88.2:0
proxyid=VPN_CISCO proto=0 sa=1 ref=2 serial=1
  src: 0:192.168.86.0-192.168.86.127:0
  dst: 0:172.20.24.0-172.20.24.15:0
  SA: ref=3 options=30202 type=00
  dec: spi=1464a1c4 esp=des ah=sha1
  enc: spi=ae208eee esp=des ah=sha1
```
<img width="1119" height="582" alt="image" src="https://github.com/user-attachments/assets/e63622ed-e881-4d37-a8c8-6c3aa8f44f3e" />

### 3. Prueba ICMP y Traza desde la PC1
```text
PC1> ping 172.20.24.2
84 bytes from 172.20.24.2 icmp_seq=1 ttl=63 time=28.412 ms

PC1> trace 172.20.24.2
trace to 172.20.24.2, 8 hops max, press Ctrl+C to stop
 1   192.168.86.1   12.608 ms  14.907 ms  28.527 ms
 2   172.20.24.2    29.110 ms  26.402 ms  25.881 ms
```
---

## 🖼 Diagramas y Evidencias

```text
2026-10-02 16:16:14.289090 port3 in 192.168.86.10 -> 172.20.24.2: icmp: echo request
2026-10-02 16:16:14.289126 VPN_CISCO out 192.168.86.10 -> 172.20.24.2: icmp: echo request
```

---

## 📜 Running Configurations

Los RUNNING-CONFIG estan linkeados en el directorio.
---

## 📝 Scripts y Archivos Utilizados

### Script de Inicialización de Red en Servidor Linux (`scripts/server-init.sh`)
```bash
#!/bin/bash
sudo ip addr add 172.20.24.2/28 dev ens3
sudo ip link set ens3 up
sudo ip route replace default via 172.20.24.1 dev ens3
echo "Red configurada correctamente:"
ip route
```

### Script VPCS PC1 (`scripts/vpcs-setup.txt`)
```text
ip 192.168.86.10 255.255.255.128 192.168.86.1
save
```

---

## 🏁 Conclusión

La realización de este laboratorio permitió validar la interoperabilidad entre dos fabricantes líderes en la industria de redes (**Fortinet** y **Cisco**). Se demostró que es posible establecer una comunicación privada y cifrada exitosa a través de una VPN IPsec incluso cuando existen limitaciones de licenciamiento o versión que restringen los algoritmos criptográficos soportados.

El correcto diagnóstico paso a paso resultó fundamental para lograr una solución funcional y profesional.
