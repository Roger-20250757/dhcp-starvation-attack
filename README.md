# Ataque DHCP Starvation

**Autor:** Roger Rodriguez  
**Matrícula:** 20250757  
**Fecha:** Junio 2026 

**Link:**  https://youtu.be/DlGYlq84_wc

---

## Objetivo del laboratorio

Demostrar cómo un atacante puede agotar el pool de direcciones IP de un servidor
DHCP enviando múltiples solicitudes DHCP con MACs falsas y aleatorias, dejando
al servidor sin IPs disponibles para asignar a clientes legítimos, causando una
Denegación de Servicio (DoS) en la red.

---

## Objetivo del script

El script `dhcp_starvation.py` realiza las siguientes acciones:

1. Genera 200 MACs aleatorias únicas
2. Por cada MAC envía un DHCP Discover al servidor
3. El servidor reserva una IP para cada MAC falsa
4. El pool de IPs se agota y los clientes legítimos no pueden obtener dirección

---

## Parámetros usados

| Parámetro | Valor | Descripción |
|---|---|---|
| `INTERFAZ` | `eth0` | Interfaz de red del atacante |
| `PAQUETES` | `200` | Cantidad de DHCP Discovers falsos |
| `DELAY` | `0.05` | Tiempo entre paquetes (segundos) |
| `DST` | `255.255.255.255` | Broadcast de red |
| Puerto destino | `67` | Puerto del servidor DHCP |
| Puerto origen | `68` | Puerto del cliente DHCP |

---

## Requisitos para utilizar la herramienta

### Software
- Kali Linux
- Python 3.x
- Librería Scapy

### Instalación de dependencias
```bash
sudo apt update && sudo apt install python3-scapy -y
```

### Permisos
```bash
sudo python3 dhcp_starvation.py
```

---

## Documentación del funcionamiento del script

### ¿Cómo funciona DHCP Starvation?

Un servidor DHCP tiene un pool limitado de IPs para asignar. Cuando recibe
un DHCP Discover, reserva una IP para ese cliente identificado por su MAC.
Si un atacante envía miles de Discovers con MACs distintas, el servidor
reserva una IP por cada una hasta agotar el pool completo.

### Flujo del ataque

```
NORMAL:
PC1 --DISCOVER (MAC real)--> Servidor DHCP
PC1 <--OFFER (IP: 20.25.7.10)-- Servidor DHCP

CON ATAQUE:
Kali --DISCOVER (MAC: a9:e0:d9:f8:0e:70)--> Servidor  → reserva 20.25.7.2
Kali --DISCOVER (MAC: 32:35:2b:4d:ee:ca)--> Servidor  → reserva 20.25.7.3
Kali --DISCOVER (MAC: 5d:bf:f9:1c:3d:77)--> Servidor  → reserva 20.25.7.4
...x200 veces...
Pool agotado - PC1 no puede obtener IP
```

### Diagrama del ataque

```
ATTACKER (Kali)              SW-ACCESS-1           DHCP Server
20.25.7.100                                        20.25.7.1
      |                           |                     |
      |--DISCOVER MAC-1---------->|-------------------->|  reserva .2
      |--DISCOVER MAC-2---------->|-------------------->|  reserva .3
      |--DISCOVER MAC-3---------->|-------------------->|  reserva .4
      |           ...             |                     |
      |--DISCOVER MAC-200-------->|-------------------->|  reserva .201
      |                           |                     |
      |                      Pool agotado               |
      |                           |                     |
PC1   |--DISCOVER---------------->|-------------------->|  ERROR: no hay IPs
```

### Pasos del script

1. **Generación MAC:** Crea una MAC aleatoria única para cada paquete
2. **Construcción BOOTP:** Arma el frame con la MAC falsa como `chaddr`
3. **DHCP Discover:** Agrega la opción DHCP tipo 1 (discover)
4. **Envío:** Manda el paquete por broadcast cada 50ms
5. **Reporte:** Muestra progreso cada 20 paquetes con la MAC usada

---

## Documentación de la red

### Topología

```
                    Router-GW
                   20.25.7.1/24
                        |
                    SW-CORE
                   /         \
            SW-ACCESS-1    SW-ACCESS-2
            /       \            \
        ATTACKER    PC1          PC2
      20.25.7.100  20.25.7.10  20.25.7.20
```

### Interfaces y direccionamiento

| Dispositivo | Interfaz | IP | VLAN |
|---|---|---|---|
| Router-GW | e0/0 | 20.25.7.1/24 | 1 |
| ATTACKER | eth0 | 20.25.7.100/24 | 1 |
| PC1 | eth0 | 20.25.7.10/24 | 1 |
| PC2 | eth0 | 20.25.7.20/24 | 1 |

---

## Ejecución del ataque

```bash
# Clonar el repositorio
git clone https://github.com/tu-usuario/dhcp-starvation-attack

# Entrar al directorio
cd dhcp-starvation-attack

# Ejecutar el script
sudo python3 dhcp_starvation.py
```

### Resultado esperado
```
==================================================
 DHCP Starvation Attack
 Autor    : Roger Rodriguez
 Matricula: 20250757
 Interfaz : eth0
 Paquetes : 200
==================================================
[*] Iniciando ataque...
[+] Paquetes enviados: 20/200  | MAC: a9:e0:d9:f8:0e:70
[+] Paquetes enviados: 40/200  | MAC: 32:35:2b:4d:ee:ca
[+] Paquetes enviados: 60/200  | MAC: 5d:bf:f9:1c:3d:77
[+] Paquetes enviados: 80/200  | MAC: c1:9d:6c:ab:95:a5
[+] Paquetes enviados: 100/200 | MAC: c2:bf:ae:d0:9c:35
...
==================================================
[*] Ataque finalizado
[+] Enviados : 200
[-] Errores  : 0
==================================================
```

---

## Contra-medida

### Descripción
Configurar **DHCP Snooping Rate Limiting** en los puertos de acceso
para limitar la cantidad de paquetes DHCP por segundo que puede enviar
un cliente. Si supera el límite el puerto se deshabilita automáticamente.

### Implementación en SW-ACCESS-1 y SW-ACCESS-2
```
enable
configure terminal
interface e0/1
 ip dhcp snooping limit rate 10
interface e0/2
 ip dhcp snooping limit rate 10
end
write memory
```

### Verificación
```
SW-ACCESS-1# show ip dhcp snooping statistics
Packets Forwarded             = 0
Packets Dropped               = 0
Packets Dropped From untrusted ports = 0
```

### ¿Por qué funciona?
El rate limit de 10 pps (paquetes por segundo) permite el tráfico DHCP
normal de un cliente legítimo pero bloquea automáticamente el flood
masivo del atacante que supera ese límite.

---

## Diferencia entre DHCP Spoofing y DHCP Starvation

| Característica | DHCP Spoofing | DHCP Starvation |
|---|---|---|
| Objetivo | Redirigir tráfico | Agotar pool de IPs |
| Tipo de ataque | MitM | DoS |
| Paquetes usados | OFFER y ACK falsos | DISCOVER masivos |
| Impacto | Interceptar tráfico | Denegar servicio |
| Contra-medida | DHCP Snooping trust | DHCP Rate Limit |

---

## Capturas de pantalla

| Archivo | Descripción |
|---|---|
| <img width="864" height="528" alt="image" src="https://github.com/user-attachments/assets/6b76b241-0fef-4406-9357-08c0e0da0bca" /> | Topología en EVE-NG |
|<img width="866" height="790" alt="image" src="https://github.com/user-attachments/assets/e2840d9a-2063-4990-9f24-60b339847a34" />  | Script corriendo con 200 MACs aleatorias |
| <img width="951" height="312" alt="image" src="https://github.com/user-attachments/assets/958729e9-4453-4981-8ccc-9e001145e116" /> | Rate limit configurado en SW-ACCESS-1 y SW-ACCESS-2 |

---

## Referencias

- [DHCP Starvation Attack](https://www.geeksforgeeks.org/dhcp-starvation-attack/)
- [Cisco DHCP Snooping Rate Limit](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/12-2SX/configuration/guide/book/snoodhcp.html)
- [Scapy Documentation](https://scapy.readthedocs.io/)
