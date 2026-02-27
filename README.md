# 🕵️ DNS Spoofing / DNS Poisoning Attack

> **Laboratorio de Seguridad en Redes** — ITLA  
> Matrícula: 2024-2026 | Maite Cruz

---

## 📌 Objetivo del Script

El script implementa un **servidor DNS falso (rogue DNS server)** que realiza un ataque de **DNS Spoofing / DNS Poisoning**. Su función es interceptar consultas DNS legítimas y responder con una dirección IP controlada por el atacante, redirigiendo a los usuarios hacia un servicio web falso sin que se percaten del engaño.

Específicamente, el script:
- Escucha consultas DNS en el puerto 53 (UDP)
- Intercepta las consultas al dominio `itla.edu.do`
- Responde con la IP del servicio falso (`20.24.11.10`) en lugar de la IP real
- Responde con `NXDOMAIN` a cualquier otro dominio no objetivo

---

## 🖥️ Topología de Red

```
                    ┌──────────┐
                    │  vIOS    │  Gi0/0 → 20.24.11.1/24
                    │ (Router) │
                    └────┬─────┘
                         │ Gi0/0
                    ┌────┴─────────┐
                    │Switch Central│  VLAN 1 → 20.24.11.2/24
                    └──┬─────┬──┬──┘
               Gi0/1   │  Gi0/3 │  Gi0/2
          ┌────────────┘        │          └────────────┐
   ┌──────┴──────┐         (directo)          ┌─────────┴──────┐
   │ Switch IZQ  │                            │  Switch DER    │
   │20.24.11.3/24│                            │ 20.24.11.4/24  │
   └──────┬──────┘                            └────────┬───────┘
        Gi0/1                                        Gi0/1
          │ e0                                          │ e0
   ┌──────┴──────┐            ┌──────┐        ┌────────┴───────┐
   │ Win (IZQ)   │            │ Win  │        │  Linux         │
   │ 20.24.1.10  │            │(Win) │        │ 20.24.2.10/24  │
   │  (atacante) │            └──────┘        │   (víctima)    │
   └─────────────┘                            └────────────────┘
```

### 📋 Tabla de Direccionamiento IP

| Dispositivo         | Interfaz | Dirección IP    |
|---------------------|----------|-----------------|
| vIOS (Router)       | Gi0/0    | 20.24.11.1/24   |
| Switch Central (L3) | VLAN 1   | 20.24.11.2/24   |
| Switch IZQ          | VLAN 1   | 20.24.11.3/24   |
| Switch DER          | VLAN 1   | 20.24.11.4/24   |
| Win – Atacante      | e0       | 20.24.1.10/24   |
| Linux – Víctima     | e0       | 20.24.2.10/24   |

---

## ⚙️ Parámetros del Script

| Parámetro      | Valor           | Descripción                                       |
|----------------|-----------------|---------------------------------------------------|
| `DOMAIN`       | `itla.edu.do`   | Dominio objetivo a suplantar                      |
| `REDIRECT_IP`  | `20.24.11.10`   | IP del servidor falso al que se redirige el tráfico |
| `LISTEN_IP`    | `0.0.0.0`       | Escucha en todas las interfaces disponibles       |
| `LISTEN_PORT`  | `53`            | Puerto DNS estándar (UDP)                         |

---

## 🛠️ Requisitos para Utilizar la Herramienta

### Software
- **Python 3.x** (sin dependencias externas, solo biblioteca estándar)
- Sistema operativo: **Linux** (recomendado Kali Linux / Ubuntu)
- Privilegios de **root/sudo** (el puerto 53 es privilegiado)

### Red
- El host atacante debe estar en la misma red o segmento que la víctima
- El DNS del host víctima debe estar configurado para apuntar a la IP del atacante
- Acceso al puerto UDP 53

### Ejecución
```bash
# Clonar o descargar el script
git clone https://github.com/maitecruz23/DNS-Spoofing-DNS-Poisoning

# Ejecutar con privilegios de root
sudo python3 dns_spoof.py
```

### Configurar la víctima
En el host víctima (Windows/Linux), cambiar el servidor DNS primario a la IP del atacante (`20.24.1.10`) para que las consultas DNS sean interceptadas.

---

## 📸 Demostración del Ataque

### Servidor DNS Falso Activo
El script escucha en el puerto 53 y responde a consultas del dominio `itla.edu.do`:
```
[*] Servidor DNS escuchando en 0.0.0.0:53
[*] Respondiendo itla.edu.do -> 20.24.11.10
[*] Otros dominios → NXDOMAIN
[*] Consulta para 'itla.edu.do' desde 20.24.2.10
[+] Coincide! Respondiendo 20.24.11.10
```

### Resultado en la Víctima
Desde el host víctima, al hacer `ping itla.edu.do`, el dominio resuelve a la IP del atacante (`20.24.11.10`) en lugar de la IP legítima:

```
C:\Windows\system32> ping itla.edu.do

Pinging itla.edu.do [20.24.11.10] with 32 bytes of data:
Reply from 20.24.11.10: bytes=32 time=170ms TTL=64
Reply from 20.24.11.10: bytes=32 time=26ms  TTL=64
Reply from 20.24.11.10: bytes=32 time=38ms  TTL=64
Reply from 20.24.11.10: bytes=32 time=30ms  TTL=64

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
Minimum = 26ms, Maximum = 170ms, Average = 66ms
```

✅ **El ataque fue exitoso**: el dominio `itla.edu.do` fue redirigido a `20.24.11.10` (servicio controlado por el atacante).

---

## 🔒 Medidas de Mitigación

### 1. DNSSEC (DNS Security Extensions)
Implementar DNSSEC para firmar criptográficamente las respuestas DNS, garantizando su autenticidad e integridad. Cualquier respuesta manipulada será detectada y descartada.

### 2. DNS sobre HTTPS / TLS (DoH / DoT)
Utilizar DNS cifrado para que las consultas no puedan ser interceptadas ni modificadas en tránsito.

### 3. Configuración estática de DNS
En entornos corporativos, fijar el servidor DNS mediante políticas de red (DHCP Snooping, políticas de grupo) para evitar que un atacante redirija las consultas.

### 4. DHCP Snooping y DAI
- **DHCP Snooping**: Impide que hosts no autorizados actúen como servidores DHCP y asignen DNS maliciosos.
- **Dynamic ARP Inspection (DAI)**: Valida las respuestas ARP para evitar ataques MitM que podrían combinarse con DNS Spoofing.

### 5. Monitoreo y detección
- Usar herramientas IDS/IPS (como Snort o Suricata) para detectar respuestas DNS anómalas.
- Monitorear cambios inesperados en registros DNS mediante logs de servidores y SIEM.
- Alertar cuando un host responde en el puerto 53 sin ser el servidor DNS autorizado.

### 6. Segmentación de red
Aislar los servidores DNS en segmentos de red protegidos y aplicar ACLs para que solo los servidores legítimos puedan responder en el puerto 53.

---

## 📁 Estructura del Repositorio

```
DNS-Spoofing-DNS-Poisoning/
├── Script ATAQUE DNS          # Script Python del servidor DNS falso
├── Configuracion del router   # Configuración vIOS Router
├── Configuracion Switch Central-L3
├── Configuracion SW-IZQ
├── Configuracion SW-DER
├── Configuracion-RADIUS-NPS   # Configuración autenticación AAA/RADIUS
└── README.md                  # Este documento
```

---

## ⚠️ Disclaimer

> Este laboratorio fue realizado con fines **exclusivamente educativos** en un entorno de red controlado y aislado como parte del curso de Seguridad en Redes del ITLA. El uso de estas técnicas en redes reales sin autorización explícita es **ilegal** y está sujeto a sanciones penales.
