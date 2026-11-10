# Evaluación del Protocolo IRC mediante Contenedores

> Despliegue de una arquitectura cliente-servidor para el protocolo Internet Relay Chat (IRC) en entornos aislados con Docker, análisis de paquetes en texto plano sobre TCP y evaluación de hipótesis de alteración de tráfico en tránsito (*on-the-fly*).

[![Docker](https://img.shields.io/badge/Docker-24.x%20%7C%2028.x-blue?logo=docker&logoColor=white)](https://www.docker.com/)
[![Docker Compose](https://img.shields.io/badge/Docker_Compose-v2-blue?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![InspIRCd](https://img.shields.io/badge/Server-InspIRCd_latest-green)](https://www.inspircd.org/)
[![WeeChat](https://img.shields.io/badge/Client-WeeChat_4.9.2-orange)](https://weechat.org/)
[![Wireshark](https://img.shields.io/badge/Capture-Wireshark_pcapng-1679A7?logo=wireshark&logoColor=white)](https://www.wireshark.org/)

---

## Tabla de Contenidos

* [Descripción del Proyecto](#descripción-del-proyecto)
* [Entregables](#entregables)
* [Topología y Arquitectura](#topología-y-arquitectura)
* [Estructura del Repositorio](#estructura-del-repositorio)
* [Requisitos Previos](#requisitos-previos)
* [Instalación y Despliegue](#instalación-y-despliegue)
* [Guía de Uso](#guía-de-uso)
* [Análisis de Tráfico y Evidencias](#análisis-de-tráfico-y-evidencias)
* [Autores](#autores)
* [Referencias](#referencias)

---

## Descripción del Proyecto

Este proyecto corresponde a la **Tarea 2 del Taller de Redes y Servicios (Universidad Diego Portales)**. El objetivo principal es desplegar un servicio de red funcional basado en el protocolo **IRC (RFC 1459 y RFC 2812)** utilizando contenedores Docker independientes, orquestados en una red virtual privada en modo puente (*bridge*). 

Se evalúa la interacción cliente-servidor operando deliberadamente en el puerto estándar **6667/TCP sin cifrado TLS (`-notls`)** con el fin de auditar la carga útil en texto plano, caracterizar el flujo de señalización y mantenimiento de conexión (`PING`/`PONG`), y documentar el impacto operativo de modificaciones de tráfico en tránsito.

---

## Entregables

* **Informe Técnico (Capítulos 1 y 2):** Archivo compilado [`informe/main.pdf`](informe/main.pdf) y código fuente LaTeX [`informe/main.tex`](informe/main.tex).
* **Captura de Red:** Archivo de paquetes original [`capturas/trafico_irc.pcapng`](capturas/trafico_irc.pcapng).
* **Video Demostrativo (Máximo 8 minutos):** [Ver demostración en YouTube](https://youtu.be/TU_ENLACE_AQUI) *(Reemplazar con enlace definitivo)*.

---

## Topología y Arquitectura

La solución se implementa en un sistema anfitrión **Kali Linux**, comunicando dos contenedores aislados mediante la red virtual puente `red-irc`:

```text
+-------------------------------------------------------------------------------+
| Máquina Anfitriona: Kali Linux (VMware / VirtualBox en modo NAT)              |
|                                                                               |
|   +-----------------------------------------------------------------------+   |
|   | Red Docker Privada: red-irc (driver: bridge)                          |   |
|   |                                                                       |   |
|   |   +-------------------------+             +-----------------------+   |   |
|   |   | Contenedor Cliente      |             | Contenedor Servidor   |   |   |
|   |   | weechat (Alpine Linux)  | <=========> | inspircd (InspIRCd)   |   |   |
|   |   | IP: 172.18.0.3          |   TCP/6667  | IP: 172.18.0.2        |   |   |
|   |   | Puerto efímero: 49224   |  (texto     | Puerto: 6667/TCP      |   |   |
|   |   +-------------------------+   plano)    +-----------------------+   |   |
|   +-----------------------------------------------------------------------+   |
|                                    ^                                          |
|                                    | Interfaz puente: br-73de0fc6d41b         |
|                          +-------------------+                                |
|                          | Wireshark Host    | (Filtro: tcp.port == 6667)     |
|                          +-------------------+                                |
+-------------------------------------------------------------------------------+
```

---

## Estructura del Repositorio

```text
.
├── capturas/
│   └── trafico_irc.pcapng         # Archivo pcapng con la traza completa de paquetes
├── cliente/
│   └── Dockerfile                 # Imagen ligera Alpine con cliente WeeChat
├── servidor/
│   └── Dockerfile                 # Imagen basada en inspircd/inspircd-docker
├── informe/
│   ├── main.pdf                   # Versión final compilada del informe
├── docker-compose.yml             # Orquestación de servicios, volúmenes y red
└── README.md                      # Documentación principal del proyecto
```

---

## Requisitos Previos

Antes de ejecutar el entorno, asegúrate de contar con las siguientes herramientas instaladas en el sistema anfitrión:

* **Sistema Operativo:** Kali Linux, Ubuntu 22.04+ o cualquier distribución Linux compatible.
* **Docker Engine:** Versión 24.0 o superior (`docker --version`).
* **Docker Compose:** Plugin v2 o superior (`docker compose version`).
* **Wireshark:** Para la inspección y visualización de trazas de red.
* **Git:** Para control de versiones y sincronización de cambios.

---

## Instalación y Despliegue

### 1. Clonar el repositorio

```bash
git clone https://github.com/Panxxto/tarea2-redes-irc.git
cd tarea2-redes-irc
```

### 2. Construir las imágenes y levantar los servicios

Ejecuta el despliegue orquestado en segundo plano:

```bash
docker compose up -d --build
```

### 3. Verificar el estado de los contenedores

Comprueba que ambos servicios se encuentren en estado `Up`:

```bash
docker ps
```

---

## Guía de Uso

### 1. Iniciar la captura de paquetes con Wireshark

Identifica la interfaz virtual puente asociada a la red `red-irc`:

```bash
ip -brief link
```

(Típicamente identificada con el prefijo `br-`, por ejemplo `br-73de0fc6d41b`).

Abre Wireshark escuchando sobre dicha interfaz y aplica el filtro de captura o visualización:

```text
tcp.port == 6667
```

### 2. Acceder al cliente WeeChat

Ingresa a la consola interactiva del contenedor cliente:

```bash
docker exec -it cliente-irc weechat --dir /tmp/weechat
```

### 3. Secuencia de comandos dentro de WeeChat

Una vez dentro de la interfaz interactiva de WeeChat, ejecuta los siguientes comandos en orden:

```text
/server add local servidor-irc/6667 -notls
/connect local
/join #redes
```

A continuación, escribe el mensaje de prueba **sin barra inicial** (se envía como `PRIVMSG` al canal `#redes`):

```text
Holaaa probando comunicacion IRC con WeeChat e InspIRCd
```

Para finalizar la sesión y salir de WeeChat:

```text
/quit
```

> **Nota:** `/quit` cierra WeeChat por completo. Si solo deseas desconectarte del servidor sin cerrar el cliente, utiliza `/disconnect local`.

### 4. Detener el entorno

Para pausar y remover los contenedores sin eliminar volúmenes ni capturas:

```bash
docker compose down
```

---

## Análisis de Tráfico y Evidencias

El archivo de captura `capturas/trafico_irc.pcapng` contiene el registro de los 10 tipos de tráfico caracterizados a lo largo del ciclo de vida de la sesión:

| N° | Tipo de Tráfico / Fase | Origen → Destino | Detalle Técnico y Banderas |
|----|------------------------|------------------|----------------------------|
| 1 | Establecimiento TCP | 172.18.0.3:49224 → 172.18.0.2:6667 | Three-Way Handshake TCP (SYN, SYN-ACK, ACK). |
| 2 | Registro de Identidad | 172.18.0.3 → 172.18.0.2 | Comandos `NICK` y `USER` transmitidos en texto claro. |
| 3 | Bienvenida y Capacidades | 172.18.0.2 → 172.18.0.3 | Códigos numéricos 001-004, MOTD y marcas `@time`. |
| 4 | Ingreso a Canal | 172.18.0.3 → 172.18.0.2 | `JOIN #redes` (Pkt 40) y respuestas 353 / 366 (Pkt 41). |
| 5 | Sincronización de Modos | 172.18.0.3 → 172.18.0.2 | Comando `MODE #redes` para auditar permisos del operador (+nt). |
| 6 | Consulta de Miembros | 172.18.0.3 → 172.18.0.2 | Consulta `WHO #redes` y respuestas de presencia 352 / 315. |
| 7 | Mensajería de Aplicación | 172.18.0.3 → 172.18.0.2 | `PRIVMSG` (Pkt 52) con delimitador `0x0D 0x0A` y réplica echo (Pkt 53). |
| 8 | Mantenimiento Keep-Alive | Alternado (PING ↔ PONG) | Sondas periódicas de presencia para evitar el Ping timeout. |
| 9 | Terminación de Aplicación | 172.18.0.3 → 172.18.0.2 | Comando `QUIT` (Pkt 69) y procesamiento en servidor (Pkt 70). |
| 10 | Cierre de Transporte TCP | 172.18.0.2 ↔ 172.18.0.3 | Intercambio de bandera FIN, ACK (Pkt 72) y liberación vía RST, ACK (Pkt 73). |

---

## Autores

* **Diego Maldonado** — Configuración de entorno Docker, captura de red y documentación técnica.
* **Nicolas Quintanilla** — Despliegue de servicios, análisis de tráfico en Wireshark y redacción de informe.

---

## Referencias

* [RFC 1459: Internet Relay Chat Protocol](https://datatracker.ietf.org/doc/html/rfc1459)
* [RFC 2812: Internet Relay Chat: Client Protocol](https://datatracker.ietf.org/doc/html/rfc2812)
* [InspIRCd Documentation](https://docs.inspircd.org/)
* [WeeChat Quick Start Guide](https://weechat.org/doc/quickstart/)
* [Docker Compose Specification](https://docs.docker.com/compose/compose-file/)
* [README Cheatsheet by @ritaly](https://github.com/ritaly/README-cheatsheet)
