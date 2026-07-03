<style>
/* Evitar orfandad de títulos al exportar a PDF */
h1, h2, h3, h4, h5, h6 {
  page-break-after: avoid !important;
  break-after: avoid !important;
}

/* Evitar que los bloques de artículos se corten entre páginas */
.article-block {
  display: block !important;
  page-break-inside: avoid !important;
  break-inside: avoid !important;
}

/* Evitar que imágenes, tablas, código, párrafos, listas y citas se dividan */
img, table, pre, p, li, tr, blockquote, figure, div[style*="text-align: center"] {
  page-break-inside: avoid !important;
  break-inside: avoid !important;
}

/* Asegurar que el body no interfiera con los saltos de página en la impresión */
@media print {
  body {
    max-width: none !important;
    margin: 0 !important;
    padding: 0 !important;
  }
}
</style>

<div style="text-align: center; padding-top: 50px; font-family: 'Outfit', sans-serif;">

<h1>Instituto Tecnológico de Las Américas (ITLA)</h1>
<br><br>
<h2>Configuración y Verificación de VPN Client-to-Site (Remote Access SSL & IPsec) en Firewall FortiGate</h2>
<p style="text-align: center; font-size: 1.2em; color: #555; margin: 1em 0;">Documentación Técnica Profesional — Práctica 5 (Semana 6)</p>
<br><br><br>
<div class="presentacion-card">
<strong>Estudiante:</strong> Alan Daniel Garcia Mendez<br>
<strong>Matrícula:</strong> 2025-1403<br>
<strong>Carrera:</strong> Seguridad Informática<br>
<strong>Asignatura:</strong> Seguridad de Redes (TSI-203)<br>
<strong>Docente:</strong> Jonathan Esteban Rondon Corniel<br>
<strong>Fecha de Entrega:</strong> 3 de julio de 2026<br>
<strong>Repositorio de GitHub:</strong> <a href="https://github.com/imAlanG16/10_fortigate_client_to_site_vpn">https://github.com/imAlanG16/10_fortigate_client_to_site_vpn</a>
</div>
</div>

## Objetivo de la VPN
El objetivo principal de esta práctica consiste en configurar y validar una red privada virtual de acceso remoto (VPN Client-to-Site o Remote Access VPN) utilizando como puerta de enlace central un firewall FortiGate (`FG-Int`). Este diseño permite que un cliente externo posicionado en internet (`C-Externo`) establezca un túnel cifrado y seguro para acceder a los recursos internos de la red corporativa (`14.3.0.0/24`) a través del enrutador de distribución interno (`R-Int`).

Se documentan e implementan dos metodologías de conexión:
* **SSL VPN (Secure Sockets Layer VPN):** Basado en TLS, que permite la conexión mediante portal web o mediante el cliente ligero FortiClient a través del puerto HTTPS TCP personalizado.
* **IPsec VPN Client-to-Site (Dial-Up):** Configuración clásica de IPsec utilizando el cliente VPN nativo para conexiones de alto rendimiento en la capa de red.

## Topología de Red y Direccionamiento IP
La topología física implementada en GNS3 consta de un cliente externo, un enrutador ISP que simula el tránsito en la nube pública, un firewall FortiGate que protege el perímetro interno, un enrutador interno (`R-Int`) que gestiona el enrutamiento local y un switch que distribuye la conexión a los clientes de la LAN interna.

<div style="text-align: center; margin: 10px 0;">
  <img src="images/topologia_client_to_site.png" width="600" alt="Topología de Red VPN Client-to-Site FortiGate">
  <p style="font-size: 0.9em; color: #666; font-style: italic;">Esquema físico de la topología Client-to-Site implementada</p>
</div>

El direccionamiento IP asignado a los distintos segmentos y equipos de la red se define detalladamente a continuación:

| Dispositivo / Rol | Interfaz | Dirección IP / Subred | Gateway | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| **Cliente Externo (C-Externo)** | e0 | `1.1.1.2/30` | `1.1.1.1` | Host remoto simulado en internet |
| **Router ISP (Tránsito)** | e0/1 | `1.1.1.1/30` | N/A | Enlace hacia el Cliente Externo |
| | e0/0 | `2.2.2.1/30` | N/A | Enlace hacia el Firewall FortiGate |
| **Firewall FortiGate (FG-Int)** | port1 (WAN) | `2.2.2.2/30` | `2.2.2.1` | Interfaz pública perimetral |
| | port2 (Internal) | `10.0.0.1/30` | N/A | Interfaz interna conectada a R-Int |
| | port3 (Management) | `192.168.211.202/24` | N/A | Interfaz de administración (Acceso GUI/SSH) |
| **Router Interno (R-Int)** | e0/0 | `10.0.0.2/30` | `10.0.0.1` | Interfaz de tránsito hacia FortiGate |
| | e0/1 | `14.3.0.1/24` | N/A | Puerta de enlace de la LAN interna |
| **Clientes Internos (VPCs)** | eth0 | `14.3.0.10/24` | `14.3.0.1` | Hosts locales de prueba en la LAN |
| **Pool Virtual VPN** | Virtual | `10.14.3.100 - 10.14.3.200/24` | N/A | Rango IP asignado a clientes de la VPN |

## Parámetros de la Conexión VPN
Los siguientes parámetros criptográficos y operativos han sido definidos para establecer el túnel seguro:

| Parámetro Operativo | Configuración SSL VPN | Configuración IPsec VPN (Dial-Up) |
| :--- | :--- | :--- |
| **Puerto de Escucha** | TCP `10443` (Personalizado) | UDP `500`, UDP `4500` (IPsec estándar) |
| **Pool de IPs Virtuales** | `10.14.3.100 - 10.14.3.200` | `10.14.3.100 - 10.14.3.200` |
| **Split Tunneling** | Habilitado (Solo tráfico LAN a través del túnel) | Habilitado (Solo tráfico LAN a través del túnel) |
| **Cifrado (Fase 1)** | TLS 1.2 / TLS 1.3 | AES-256 |
| **Autenticación Hash** | SHA-256 | SHA-256 |
| **Diffie-Hellman Group** | N/A | Group 14 (2048-bit) |
| **Clave Compartida (Preshared)** | N/A | `ITLAvpn2026` |
| **Usuario de Prueba** | `estudiante` | `estudiante` |
| **Contraseña de Usuario** | `FortiClient2026` | `FortiClient2026` |

## Configuración de Dispositivos Cisco de la Topología
Para que el tráfico fluya de manera correcta a través de la infraestructura física del laboratorio, se deben configurar los dispositivos de enrutamiento intermedio.

<div class="article-block">

### 1. Router ISP (Tránsito Público)
Este equipo simplemente actúa como un enrutador de internet. No tiene conocimientos de las redes privadas (`14.3.0.0/24` o `10.0.0.0/30`), enrutando exclusivamente entre las redes WAN conectadas directamente (`1.1.1.0/30` y `2.2.2.0/30`).
El script de configuración completo de este dispositivo se encuentra disponible en: [config_isp.txt](resources/config_isp.txt).

```cisco
ISP# configure terminal
ISP(config)# interface Ethernet0/1
ISP(config-if)# description Enlace hacia Cliente Externo (C-Externo)
ISP(config-if)# ip address 1.1.1.1 255.255.255.252
ISP(config-if)# no shutdown
ISP(config-if)# exit
ISP(config)# interface Ethernet0/0
ISP(config-if)# description Enlace hacia Firewall FortiGate (FG-Int)
ISP(config-if)# ip address 2.2.2.1 255.255.255.252
ISP(config-if)# no shutdown
ISP(config-if)# exit
```

</div>

<div class="article-block">

### 2. Router Interno (R-Int)
Este enrutador interconecta la LAN con el firewall FortiGate. En este dispositivo se configura un servidor DHCP local para la red `14.3.0.0/24`, permitiendo asignar direcciones IP dinámicamente a los clientes de la LAN (excluyendo el rango administrativo del `.1` al `.9`). También requiere una ruta por defecto apuntando a la IP interna del FortiGate para dar salida a internet y permitir que el tráfico de retorno de los clientes de la VPN (pertenecientes al pool `10.14.3.0/24`) sea reenviado correctamente al firewall.
El script de configuración completo de este dispositivo se encuentra disponible en: [config_r_int.txt](resources/config_r_int.txt).

```cisco
R-Int# configure terminal
R-Int(config)# ip dhcp excluded-address 14.3.0.1 14.3.0.9
R-Int(config)# ip dhcp pool LAN_POOL
R-Int(dhcp-config)# network 14.3.0.0 255.255.255.0
R-Int(dhcp-config)# default-router 14.3.0.1
R-Int(dhcp-config)# dns-server 8.8.8.8 1.1.1.1
R-Int(dhcp-config)# exit
R-Int(config)# interface Ethernet0/0
R-Int(config-if)# description Enlace hacia Firewall FortiGate (FG-Int)
R-Int(config-if)# ip address 10.0.0.2 255.255.255.252
R-Int(config-if)# no shutdown
R-Int(config-if)# exit
R-Int(config)# interface Ethernet0/1
R-Int(config-if)# description Enlace hacia la LAN (SW-Int)
R-Int(config-if)# ip address 14.3.0.1 255.255.255.0
R-Int(config-if)# no shutdown
R-Int(config-if)# exit
R-Int(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1
```

</div>

## Configuración Detallada del Firewall FortiGate (FG-Int)
La puerta de enlace VPN se configura en el firewall FortiGate a través de su interfaz de comandos de consola (CLI) o mediante el panel gráfico web (GUI). A continuación, se detalla el proceso paso a paso utilizando ambos métodos.

<div class="article-block">

### Opción A: Configuración de SSL VPN (Recomendado)

#### Paso 1: Configurar las Interfaces del Firewall e Interfaces de Red
Establecemos el enrutamiento básico y las IPs físicas del FortiGate.

```fortinet
config system interface
    edit "port1"
        set vdom "root"
        set ip 2.2.2.2 255.255.255.252
        set allowaccess ping
        set type physical
    next
    edit "port2"
        set vdom "root"
        set ip 10.0.0.1 255.255.255.252
        set allowaccess ping
        set type physical
    next
    edit "port3"
        set vdom "root"
        set ip 192.168.211.202 255.255.255.0
        set allowaccess ping https http ssh
        set type physical
    next
end

config router static
    edit 1
        set gateway 2.2.2.1
        set device "port1"
    next
    edit 2
        set dst 14.3.0.0 255.255.255.0
        set gateway 10.0.0.2
        set device "port2"
    next
end
```

#### Paso 2: Crear el Pool de Direcciones IP y el Objeto de Red LAN
Definimos el bloque de IPs virtuales que se entregarán a los clientes VPN y creamos un objeto para representar nuestra red LAN interna.

```fortinet
config firewall address
    edit "SSL_VPN_Pool"
        set type iprange
        set start-ip 10.14.3.100
        set end-ip 10.14.3.200
    next
    edit "LAN_Interna"
        set subnet 14.3.0.0 255.255.255.0
    next
end
```

#### Paso 3: Crear los Usuarios Locales y Grupos de Seguridad
Creamos la cuenta del estudiante y el grupo para la autenticación VPN.

```fortinet
config user local
    edit "estudiante"
        set status enable
        set passwd FortiClient2026
    next
end

config user group
    edit "VPN-Users"
        set member "estudiante"
    next
end
```

#### Paso 4: Configurar el SSL VPN Portal
Establecemos las políticas del portal SSL, habilitando el modo túnel con "Split Tunneling" para que el tráfico no dirigido a la LAN corporativa se enrute de manera local por la conexión del usuario.

```fortinet
config vpn ssl web portal
    edit "ITLA_SSL_Portal"
        set tunnel-mode enable
        set split-tunneling enable
        set split-tunneling-routing-address "LAN_Interna"
        set ip-pools "SSL_VPN_Pool"
    next
end
```

#### Paso 5: Configurar los SSL VPN Settings Generales
Activamos el servidor VPN SSL en la interfaz pública `port1` en el puerto `10443` y asignamos el portal a los miembros de nuestro grupo de seguridad.

```fortinet
config vpn ssl settings
    set serversource-interface "port1"
    set serverport 10443
    set ssl-min-proto-ver tls1-2
    set default-portal "full-access"
    config authentication-rule
        edit 1
            set groups "VPN-Users"
            set portal "ITLA_SSL_Portal"
        next
    next
end
```

#### Paso 6: Configurar la Política de Firewall
Para permitir que los clientes conectados pasen tráfico a la red interna, se debe crear una política de seguridad que conecte la interfaz virtual `ssl.root` con `port2`.

```fortinet
config firewall policy
    edit 10
        set name "Permitir_Acceso_SSL_VPN"
        set srcintf "ssl.root"
        set dstintf "port2"
        set action accept
        set srcaddr "all"
        set dstaddr "LAN_Interna"
        set schedule "always"
        set service "ALL"
        set groups "VPN-Users"
    next
end
```

</div>

<div class="article-block">

### Opción B: Configuración de IPsec VPN (Dial-Up / Acceso Remoto)
Si se prefiere una solución basada en IPsec, la configuración correspondiente en el FortiGate se realiza mediante los siguientes comandos:

```fortinet
config vpn ipsec phase1-interface
    edit "IPsec_DialUp"
        set type dynamic
        set interface "port1"
        set ike-version 1
        set peertype any
        set proposal aes256-sha256
        set dhgrp 14
        set psksecret ITLAvpn2026
        set authusrgrp "VPN-Users"
    next
end

config vpn ipsec phase2-interface
    edit "IPsec_DialUp_P2"
        set phase1name "IPsec_DialUp"
        set proposal aes256-sha256
        set dhgrp 14
        set keepalive enable
    next
end

config system ipsec-aggregate
end

config firewall policy
    edit 11
        set name "Permitir_Acceso_IPsec"
        set srcintf "IPsec_DialUp"
        set dstintf "port2"
        set action accept
        set srcaddr "SSL_VPN_Pool"
        set dstaddr "LAN_Interna"
        set schedule "always"
        set service "ALL"
    next
end
```

</div>

## Configuración y Conexión del Cliente Externo (C-Externo)
Para conectarse a la VPN desde el cliente externo (`C-Externo`), se detallan las instrucciones correspondientes:

<div class="article-block">

### Configuración en FortiClient (Para SSL VPN)
1. Instalar la aplicación de escritorio **FortiClient VPN**.
2. Hacer clic en **Configure VPN** y seleccionar **SSL-VPN**.
3. Configurar los siguientes campos en la ventana de configuración:
   * **Connection Name:** `VPN-ITLA`
   * **Remote Gateway:** `2.2.2.2` (IP pública del FortiGate)
   * **Customize Port:** Habilitar y colocar `10443`
   * **Client Certificate:** None (o configurar si se requiere autenticación de doble factor)
   * **Authentication:** Prompt en login.
4. Hacer clic en **Save**.
5. En la interfaz principal de FortiClient, ingresar las credenciales:
   * **Username:** `estudiante`
   * **Password:** `FortiClient2026`
6. Hacer clic en **Connect** para iniciar el túnel.

</div>

## Guión de Pruebas y Validación de Conectividad
Para garantizar que la VPN Client-to-Site funciona correctamente y que el túnel enruta el tráfico como es debido, se ejecutan las siguientes pruebas de verificación en el laboratorio:

<div class="article-block">

### 1. Pruebas desde el Cliente Externo (Una vez conectado a la VPN)
* **Validación de Asignación IP:** Confirmar que la interfaz virtual del cliente haya recibido una IP del pool (`10.14.3.100 - 10.14.3.200`).
* **Ping a la LAN Interna:** Comprobar conectividad directa hacia la LAN protegida.
  ```bash
  ping 14.3.0.10
  ping 14.3.0.1
  ```
* **Ruta de Tránsito (Trace):** Comprobar que los paquetes hacia la subred `14.3.0.0/24` pasen directamente por el túnel cifrado.
  ```bash
  tracert 14.3.0.10
  ```

</div>

<div class="article-block">

### 2. Comandos de Verificación en la Consola del FortiGate (CLI)
* **Monitorear Clientes VPN Activos:**
  ```fortinet
  get vpn ssl monitor
  ```
  *Este comando muestra los usuarios actualmente logueados, su IP pública de origen, la IP virtual asignada del pool y los bytes transmitidos/recibidos.*
* **Verificar las Asociaciones de Seguridad (SAs) Activas de IPsec:**
  ```fortinet
  get vpn ipsec tunnel details
  ```
* **Consultar la Tabla de Enrutamiento Activa:**
  ```fortinet
  get router info routing-table database
  ```
</div>
