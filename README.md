# Infraestructura 2 – VPN IPsec Site-to-Site

### Merolyn Mejia
### Mat.2025-0827

## Video demostrativo

[Ver video demostrativo](https://youtu.be/UnBAwzCTpss)

---

## Índice

- [Propósito del laboratorio](#propósito-del-laboratorio)
- [Topología](#topología)
  - [Direccionamiento utilizado](#direccionamiento-utilizado)
- [Configuración de la infraestructura](#configuración-de-la-infraestructura)
  - [Red de Usuarios y DHCP](#red-de-usuarios-y-dhcp)
  - [NAT en Cisco](#nat-en-cisco)
  - [VPN IPsec Site-to-Site](#vpn-ipsec-site-to-site)
  - [Web Server HTTPS](#web-server-https)
  - [FortiGate](#fortigate)
- [Pruebas de funcionamiento](#pruebas-de-funcionamiento)
  - [Conectividad hacia Internet](#conectividad-hacia-internet)
  - [Comunicación Usuario → Web Server](#comunicación-usuario--web-server)
  - [Traceroute hacia el servidor](#traceroute-hacia-el-servidor)
  - [Prueba con el tráfico VPN bloqueado](#prueba-con-el-tráfico-vpn-bloqueado)
  - [Restauración de la comunicación](#restauración-de-la-comunicación)
- [Running Configuration](#running-configuration)
- [Conclusión](#conclusión)

---

## Propósito del laboratorio

El propósito de este laboratorio es implementar en GNS3 una infraestructura que permita comunicar una red de usuarios con una red de servidores mediante una VPN IPsec Site-to-Site entre un router Cisco y un firewall FortiGate.

También se implementaron NAT, DHCP y un servidor web HTTPS. Finalmente, se realizaron pruebas para comprobar que la comunicación entre el usuario y el servidor funciona cuando el tráfico puede pasar por la VPN y se interrumpe cuando dicho paso es deshabilitado.

### Objetivos específicos

La práctica está orientada a comprobar el funcionamiento de una comunicación entre una red de usuarios y una red de servidores utilizando una VPN IPsec Site-to-Site.

Los objetivos principales son:

1. Comunicar el Usuario con el Servidor mediante el enlace VPN.
2. Comprobar que la comunicación entre ambas redes depende del funcionamiento del enlace VPN.
3. Verificar que el tráfico destinado a la red remota sea tratado correctamente por los dispositivos de red.
4. Comprobar el funcionamiento de NAT para el tráfico que sale hacia el segmento externo.
5. Validar el funcionamiento del servicio Web mediante HTTPS.
6. Comprobar el recorrido del tráfico mediante una prueba de traceroute.

La práctica no se limita a comprobar que el túnel VPN aparece activo. También se realiza una prueba de bloqueo en la que se deshabilita temporalmente la política relacionada con el tráfico VPN y se comprueba que la comunicación deja de funcionar. Posteriormente, la política se habilita nuevamente para verificar que la comunicación puede ser restaurada.

---

## Topología

La infraestructura está compuesta por un router Cisco R1, un firewall FortiGate, un segmento WAN/ISP, un usuario y un servidor web HTTPS.

![Topología de la infraestructura](Imagenes/image01.png)

### Descripción de la topología

La infraestructura está dividida en dos segmentos internos: una red de usuarios y una red de servidores. Ambas redes se encuentran separadas y utilizan diferentes FortiGate como dispositivos de seguridad.

El lado de usuarios está compuesto por el equipo Windows, conectado a la red `10.8.27.0/25`, cuyo gateway es `10.8.27.1`. Esta red se encuentra detrás del router Cisco R1.

El lado de servidores utiliza la red `172.8.27.0/28`, cuyo gateway es `172.8.27.1` y donde se encuentra el Web Server con dirección `172.8.27.2`.

Cisco R1 y el FortiGate se comunican mediante el segmento WAN/ISP `192.168.42.0/24`. Este segmento representa dentro del laboratorio el medio de comunicación externo entre el equipo de red y el firewall.

Sobre esta infraestructura se establece la VPN IPsec Site-to-Site, cuyo propósito es permitir que el tráfico entre la red de usuarios y la red de servidores pueda atravesar de forma controlada el enlace VPN.


### Direccionamiento utilizado

| Dispositivo | Interfaz | Dirección IP | Función |
|---|---|---|---|
| R1 | FastEthernet0/0 | `10.8.27.1/25` | Gateway Usuarios |
| R1 | FastEthernet1/0 | `192.168.42.82/24` | WAN / ISP |
| Usuario | DHCP | `10.8.27.2/25` | VLAN 10 |
| FortiGate | port1 | `192.168.42.6/24` | WAN / ISP |
| FortiGate | port3 | `172.8.27.1/28` | Gateway Servidores |
| Web Server | ens3 | `172.8.27.2/28` | Servidor HTTPS |

> El segmento `192.168.42.0/24` representa la red WAN/ISP dentro del laboratorio GNS3.
> Para las redes internas se utilizó el identificador `8.27`, basado en los últimos cuatro dígitos de la matrícula `2025-0827`. Por esta razón se utilizaron las redes `10.8.27.0/25` para Usuarios y `172.8.27.0/28` para Servidores.

El direccionamiento utilizado permite separar claramente las diferentes zonas de la infraestructura.

La red `10.8.27.0/25` corresponde a los usuarios y utiliza `10.8.27.1` como gateway. El equipo de usuario recibe su dirección mediante DHCP.

La red `172.8.27.0/28` corresponde a los servidores. El FortiGate utiliza `172.8.27.1` como gateway y el Web Server utiliza `172.8.27.2`.

El segmento `192.168.42.0/24` representa la red WAN/ISP utilizada dentro del laboratorio GNS3. A través de este segmento se establece la comunicación entre los dispositivos que forman los extremos de la infraestructura.

La separación de estas redes permite comprobar posteriormente el funcionamiento del enrutamiento, NAT y de la VPN IPsec.

---

# Configuración de la infraestructura

### Organización de la configuración

La configuración de la infraestructura se realizó separando las funciones de cada dispositivo.

Cisco R1 se utiliza principalmente para administrar la red de usuarios, proporcionar DHCP, realizar NAT y participar en la VPN IPsec.

El FortiGate se utiliza para administrar la red de servidores, proporcionar conectividad hacia el Web Server, controlar el tráfico mediante políticas y participar como extremo de la VPN.

De esta forma, cada dispositivo cumple una función dentro de la comunicación y las pruebas posteriores permiten comprobar cómo interactúan entre sí.


## Red de Usuarios y DHCP

La red de usuarios corresponde a `10.8.27.0/25` y utiliza `10.8.27.1` como gateway. El direccionamiento de los clientes se entrega mediante DHCP desde R1.

### Funcionamiento de la red de usuarios

La red de usuarios utiliza el segmento `10.8.27.0/25` y tiene como gateway la dirección `10.8.27.1`.

El direccionamiento de los clientes es proporcionado mediante DHCP desde Cisco R1. Esto permite que el equipo de usuario obtenga automáticamente una dirección IP y los parámetros necesarios para comunicarse con el resto de la infraestructura.

El uso de DHCP facilita la administración del segmento de usuarios y permite validar que el cliente se encuentra correctamente conectado antes de realizar las pruebas de comunicación con el servidor.


### Dirección obtenida por el Usuario

![DHCP Usuario](Imagenes/image02.png)

La dirección obtenida por el usuario confirma que el servicio DHCP configurado en R1 está funcionando correctamente.

Esta validación es importante porque establece el punto inicial de las pruebas. Antes de comprobar la VPN, se debe garantizar que el equipo de usuario posee una dirección válida y puede comunicarse con su gateway.

El equipo utilizado durante la práctica obtuvo la dirección `10.8.27.2/25`.


### Pool DHCP en R1

![DHCP R1](Imagenes/image17.png)

El pool DHCP está asociado a la red `10.8.27.0/25` y utiliza `10.8.27.1` como gateway.

De esta manera, cualquier cliente que solicite una dirección mediante DHCP dentro de este segmento puede recibir una configuración perteneciente a la red de usuarios.

La evidencia permite comprobar que R1 tiene configurado el servicio necesario para proporcionar direccionamiento dinámico a los clientes.


### Interfaces de R1

![Interfaces R1](Imagenes/image18.png)

La interfaz `FastEthernet0/0` se utiliza para conectar R1 con la red de usuarios y funciona como gateway `10.8.27.1`.

La interfaz `FastEthernet1/0` representa la conexión WAN/ISP y utiliza el direccionamiento obtenido para comunicarse con el segmento externo.

La separación entre ambas interfaces permite que R1 funcione como punto de conexión entre la red interna de usuarios y el segmento WAN utilizado para llegar al FortiGate.

---

## NAT en Cisco

R1 utiliza NAT para proporcionar conectividad externa a la red de usuarios.

### Función del NAT

NAT se utiliza en R1 para permitir que los equipos de la red de usuarios puedan utilizar la interfaz WAN para acceder a redes externas.

La red de usuarios `10.8.27.0/25` utiliza direcciones privadas, por lo que el tráfico destinado a Internet necesita ser traducido mediante NAT.

Sin embargo, el tráfico destinado a la red de servidores no debe ser traducido antes de entrar al túnel IPsec. Por esta razón se implementó una excepción para el tráfico que debe ser protegido por la VPN.

Esto permite mantener las direcciones originales de los equipos cuando el tráfico corresponde a la comunicación entre las redes privadas.

![NAT Cisco](Imagenes/image14.png)

El tráfico entre `10.8.27.0/25` y `172.8.27.0/28` se excluye del proceso NAT para permitir su transporte mediante IPsec.

![Excepción NAT VPN](Imagenes/image15.png)

La excepción de NAT evita que el tráfico entre `10.8.27.0/25` y `172.8.27.0/28` sea traducido.

Esto es importante porque la VPN utiliza las redes internas como referencia para identificar el tráfico que debe ser protegido mediante IPsec.

Si el tráfico fuera traducido antes de ser evaluado por la VPN, las direcciones de origen y destino podrían dejar de coincidir con los parámetros definidos para el túnel.

Por esta razón, el tráfico VPN se mantiene sin traducción mientras que el tráfico destinado al segmento externo puede utilizar NAT.

---

## VPN IPsec Site-to-Site

La VPN se estableció entre:

| Parámetro | Cisco | FortiGate |
|---|---|---|
| Peer WAN | `192.168.42.82` | `192.168.42.6` |
| Red protegida | `10.8.27.0/25` | `172.8.27.0/28` |

### Crypto Map de R1

#### Función del Crypto Map

El Crypto Map permite asociar los parámetros de la VPN con la interfaz WAN de R1.

Dentro del Crypto Map se define el peer remoto, el Transform Set utilizado para IPsec, la configuración PFS y la ACL que identifica el tráfico que debe ser protegido.

Una vez aplicado a `FastEthernet1/0`, R1 puede procesar mediante IPsec el tráfico que coincide con los criterios definidos en la VPN.

![Crypto Map](Imagenes/image16.png)

### VPN activa en FortiGate

![VPN activa](Imagenes/image05.png)

### Validación del túnel

El estado activo mostrado en el FortiGate confirma que la negociación de la VPN fue completada correctamente.

Esta evidencia representa la primera validación del túnel. Sin embargo, para comprobar que realmente existe tráfico protegido, también se revisó el estado de ISAKMP y las asociaciones IPsec desde R1.

### Estado ISAKMP

![ISAKMP](Imagenes/image06.png)

El estado `QM_IDLE / ACTIVE` confirma que la asociación ISAKMP se encuentra establecida entre los dos extremos de la VPN.

Esta etapa corresponde a la negociación de los parámetros necesarios para establecer la comunicación segura. Una vez establecida esta asociación, los dispositivos pueden mantener las asociaciones necesarias para procesar el tráfico IPsec.

Por esta razón, el estado ISAKMP constituye una de las validaciones utilizadas para comprobar que el túnel se encuentra correctamente establecido.

### Asociación IPsec

#### Validación del tráfico IPsec

La asociación IPsec permite comprobar que no solamente existe una sesión VPN establecida, sino que también se está procesando tráfico mediante el túnel.

Los contadores de paquetes encapsulados y cifrados representan tráfico enviado a través de IPsec, mientras que los paquetes desencapsulados y descifrados representan tráfico recibido desde el extremo remoto.

Por lo tanto, el incremento de estos contadores permite relacionar la prueba de conectividad realizada por el usuario con el funcionamiento real de la asociación IPsec.

![IPsec SA](Imagenes/image07.png)

Los contadores de paquetes cifrados y descifrados permiten comprobar que el tráfico está siendo procesado por IPsec.

---

## Web Server HTTPS

El servidor web utiliza la dirección `172.8.27.2/28` y tiene como gateway `172.8.27.1`.

### Configuración de red

![Configuración Web Server](Imagenes/image13.png)

### Servicio HTTPS TCP/443

![HTTPS 443](Imagenes/image11.png)

### Respuesta HTTPS

![HTTPS 200 OK](Imagenes/image12.png)

La respuesta `HTTP/1.1 200 OK` confirma el funcionamiento del servicio web mediante HTTPS.

---

## FortiGate

La configuración y demostración del FortiGate se realizó mediante su interfaz gráfica.

### Configuración de interfaces

![Interfaces FortiGate](Imagenes/image19.png)

Las interfaces principales son:

- `port1` → `192.168.42.6/24`
- `port3` → `172.8.27.1/28`

### NAT

![NAT FortiGate](Imagenes/image20.png)

La política `SERVIDOR-INTERNET` permite la salida desde `SERVIDORES (port3)` hacia `port1` utilizando NAT.

---

# Pruebas de funcionamiento

## Conectividad hacia Internet

![Ping Internet](Imagenes/image03.png)

Se verificó conectividad hacia Internet desde la red de usuarios.

---

## Comunicación Usuario → Web Server

### Objetivo de la prueba

Esta prueba tiene como finalidad comprobar que existe comunicación entre el equipo de usuario y el Web Server ubicado en la red remota.

El origen corresponde al usuario `10.8.27.2`, mientras que el destino es el servidor `172.8.27.2`.

Al realizar esta prueba con la VPN operativa, se espera que el tráfico sea transportado mediante el enlace IPsec establecido entre R1 y FortiGate.

![Ping Web Server](Imagenes/image04.png)

### Resultado

El resultado exitoso demuestra que existe comunicación de extremo a extremo entre las dos redes privadas.

La prueba confirma que el usuario puede alcanzar el Web Server cuando la infraestructura VPN se encuentra operativa.

---

## Traceroute hacia el servidor

![Traceroute](Imagenes/image08.png)

### Análisis del recorrido

El traceroute permite observar el recorrido que siguen los paquetes desde el equipo de usuarios hasta el servidor remoto.

Esta prueba ayuda a comprobar que el tráfico está siendo encaminado correctamente y permite identificar los dispositivos intermedios que participan en la comunicación.

El resultado complementa la prueba de ping, ya que no solamente demuestra que existe comunicación, sino que permite observar el camino utilizado para llegar al servidor.

---

## Prueba con el tráfico VPN bloqueado

### Objetivo de la prueba

Esta prueba fue realizada para comprobar el segundo objetivo principal del laboratorio: verificar que la comunicación solo funciona mientras el tráfico pueda atravesar el enlace VPN.

Para realizarla, se deshabilitó temporalmente desde la GUI del FortiGate la política que permite el tráfico correspondiente a la VPN.

Después de realizar el cambio, se volvió a ejecutar la prueba de conectividad hacia `172.8.27.2`.

![VPN bloqueada](Imagenes/image09.png)

### Resultado del bloqueo

Al encontrarse deshabilitada la política, el tráfico hacia el Web Server dejó de ser permitido y la prueba mostró `timeout`.

Este resultado demuestra que la comunicación no se mantiene simplemente porque exista una ruta hacia el servidor. El tráfico también depende de las políticas que permiten su paso por la infraestructura VPN.

La prueba permite comprobar el comportamiento de seguridad esperado: cuando se elimina la autorización del tráfico VPN, la comunicación entre el usuario y el servidor deja de funcionar.

---

## Restauración de la comunicación

![VPN restaurada](Imagenes/image10.png)

Después de habilitar nuevamente la política, el servidor vuelve a responder.

Esta comparación demuestra que la comunicación entre ambas redes depende del paso del tráfico mediante la VPN configurada.

Después de comprobar el bloqueo, la política de firewall fue habilitada nuevamente desde la interfaz gráfica del FortiGate.

Posteriormente se repitió la prueba de conectividad hacia `172.8.27.2`.

El servidor volvió a responder correctamente, demostrando que la comunicación puede restablecerse cuando la política que permite el tráfico VPN vuelve a estar activa.

Esta prueba completa el ciclo de validación:

1. VPN y política habilitadas → comunicación exitosa.
2. Política VPN deshabilitada → comunicación bloqueada.
3. Política VPN habilitada nuevamente → comunicación restaurada.

Con este procedimiento se demuestra de forma práctica que el flujo entre el usuario y el servidor está controlado por el enlace VPN y las políticas de seguridad asociadas.

---

# Running Configuration

La configuración completa del router Cisco R1 se encuentra en:

`Configuraciones/R1-running-config.txt`

Incluye las configuraciones de red, DHCP, NAT y VPN IPsec utilizadas durante la práctica.

---

# Conclusión

En esta práctica se implementó una VPN IPsec Site-to-Site entre un router Cisco R1 y un FortiGate para comunicar la red de usuarios 
10.8.27.0/25 con la red de servidores 172.8.27.0/28. Se configuraron los elementos necesarios para la comunicación, 
incluyendo direccionamiento IP, DHCP, NAT, rutas, políticas de firewall, VPN IPsec y un Web Server mediante HTTPS. También se verificó 
el estado del túnel mediante ISAKMP y las asociaciones IPsec, comprobando que el tráfico estaba siendo procesado correctamente.

Las pruebas demostraron que, con la VPN y las políticas habilitadas, el usuario podía comunicarse con el 
Web Server y realizar un traceroute hacia la red remota. Al deshabilitar la política que permitía el tráfico VPN, 
la comunicación fue bloqueada y se obtuvo timeout; posteriormente, al habilitarla nuevamente, la comunicación fue restaurada. 
Con esto se comprobó de forma práctica que el tráfico entre ambas redes depende del enlace VPN y de las políticas de seguridad configuradas.

---
