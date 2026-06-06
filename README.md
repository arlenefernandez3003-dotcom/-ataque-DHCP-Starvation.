# Ataque #4 — DHCP Starvation

**Instituto Tecnológico de las Américas (ITLA)**  
**Carrera:** Seguridad de la Información  
**Materia:** Seguridad de Redes  
**Matrícula:** 20250730  
**Entorno:** PNetLab (red virtualizada académica)

-----

## Objetivo

Demostrar el ataque DHCP Starvation, que agota el pool de direcciones IP de un servidor DHCP legítimo enviando masivamente solicitudes con MACs falsas, dejando a los clientes reales sin conectividad. Se aplica y verifica la contramedida mediante **DHCP Snooping** en el switch Cisco.

-----

## Topología

```
[Kali Linux - Atacante]          [Servidor DHCP - Ubuntu]
     20.25.7.30/24                     20.25.7.1/24
           │                                 │
           └──────────[Cisco Switch 2960]────┘
                         Fa0/0 – Fa0/3
                              │
                    [Cliente - Windows 10]
                        DHCP → Pool
                    20.25.7.10 – 20.25.7.254
```

|Dispositivo      |Rol            |IP           |Interfaz|
|-----------------|---------------|-------------|--------|
|Kali Linux (VM)  |Atacante       |20.25.7.30/24|eth0    |
|Ubuntu Server    |Servidor DHCP  |20.25.7.1/24 |eth0    |
|Cisco Switch 2960|Conmutador     |20.25.7.2/24 |Fa0/0-3 |
|Windows 10 (VM)  |Cliente víctima|DHCP dinámico|NIC     |

**Red:** 20.25.7.0/24 | **Pool:** 20.25.7.10 – 20.25.7.254 (245 IPs)

-----

## Requisitos

- Python 3.x
- Scapy: `pip3 install scapy`
- Wireshark (captura de tráfico)
- isc-dhcp-server en el servidor Ubuntu
- Acceso root en Kali Linux
- PNetLab con la topología configurada

-----

## Instalación

```bash
# Clonar el repositorio
git clone https://github.com/usuario/ataque4-dhcp-starvation
cd ataque4-dhcp-starvation

# Instalar dependencia
pip3 install scapy

# Verificar
python3 dhcp_starvation.py --help
```

-----

## Uso

### Modo simulación (sin tráfico real — demostración)

```bash
python3 dhcp_starvation.py --iface eth0 --count 10 --dry-run --verbose
```

### Modo laboratorio PNetLab (envío real en red virtualizada)

```bash
sudo python3 dhcp_starvation.py --iface eth0 --count 245 --delay 0.1 --verbose
# El script pedirá confirmar escribiendo: LABORATORIO
```

### Parámetros disponibles

|Parámetro  |Descripción                        |Default|
|-----------|-----------------------------------|-------|
|`--iface`  |Interfaz de red                    |eth0   |
|`--count`  |Número de solicitudes DHCP a enviar|245    |
|`--delay`  |Segundos entre paquetes            |0.1    |
|`--verbose`|Mostrar detalle de cada paquete    |False  |
|`--dry-run`|Simular sin enviar tráfico         |False  |

-----

## Cómo funciona el ataque

1. El script genera una MAC aleatoria por iteración.
1. Construye un DHCP Discover (Ether / IP / UDP / BOOTP / DHCP).
1. Lo envía en broadcast a la red.
1. Captura el DHCP Offer del servidor.
1. Responde con DHCP Request → la IP queda **reservada**.
1. Repite hasta agotar las 245 IPs del pool.
1. Los clientes reales ya no reciben IP → obtienen APIPA `169.254.x.x`.

-----

## Evidencias requeridas

- [ ] Captura Wireshark: DHCP Discover masivos con MACs distintas
- [ ] Log del servidor: `journalctl -u isc-dhcp-server | grep "no free leases"`
- [ ] Cliente Windows: `ipconfig /renew` fallido o IP APIPA
- [ ] Salida del script con resumen de IPs agotadas
- [ ] Captura Wireshark post-contramedida: paquetes dropeados por DHCP Snooping

-----

## Contramedidas — DHCP Snooping

Configurar en el switch Cisco:

```cisco
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 1
Switch(config)# no ip dhcp snooping information option

! Puerto confiable → hacia el servidor DHCP legítimo
Switch(config)# interface Fa0/1
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# exit

! Puerto no confiable → limitar tasa (bloquea Starvation)
Switch(config)# interface Fa0/2
Switch(config-if)# ip dhcp snooping limit rate 10
Switch(config-if)# exit
```

**Verificación:**

```cisco
Switch# show ip dhcp snooping statistics
Switch# show ip dhcp snooping binding
```

Tras aplicar la contramedida, el cliente Windows debe obtener IP exitosamente con `ipconfig /renew`.

-----

## Autor

**Matrícula:** 20250730  
**Institución:** ITLA — Instituto Tecnológico de las Américas  
**Carrera:** Seguridad de la Información  
**Fecha:** Junio 2025
