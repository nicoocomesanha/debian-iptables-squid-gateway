# 🛡️ Enrutamiento NAT y Proxy Transparente con Squid en Debian

Este proyecto documenta el laboratorio práctico de despliegue y configuración de una pasarela de red (router/firewall) y servidor proxy transparente sobre **Debian** (`practicasad`) mediante **iptables** y **Squid 6.13**, proporcionando salida a Internet y filtrado de navegación web por listas de control de acceso (ACL) a clientes **Ubuntu** en una subred aislada.

## 📐 Arquitectura del Sistema

- **Servidor (`practicasad` - Debian):**
  - **Interfaz Externa / WAN (`enp0s3` - Modo NAT):** Dirección asignada por DHCP (salida a Internet)
  - **Interfaz Interna / LAN (`enp0s8` - Red Interna):** `192.168.1.8/24`
- **Cliente (`nico-VirtualBox` - Ubuntu VM):** Subred `192.168.1.0/24` (Puerta de enlace: `192.168.1.8`)
- **Servicios / Cortafuegos:** Kernel IP Forwarding, `iptables` (`iptables-persistent`), Squid 6.13

## 🔀 Enrutamiento IP Forwarding y Traducción NAT

1. **Habilitación del Reenvío en el Kernel:** Activación permanente del tránsito de paquetes IPv4 entre las interfaces de red mediante la directiva `net.ipv4.ip_forward=1` en `/etc/sysctl.conf`.

![Figura 1: Habilitación de net.ipv4.ip_forward=1 en /etc/sysctl.conf](Captura%201.png)
*Figura 1: Habilitación de `net.ipv4.ip_forward=1` en `/etc/sysctl.conf`.*

2. **Aplicación de Parámetros en Tiempo de Ejecución:** Carga inmediata de la configuración del núcleo sin reiniciar el sistema mediante `sudo sysctl -p`.

![Figura 2: Carga inmediata del parámetro con sudo sysctl -p](Captura%202.png)
*Figura 2: Confirmación del reenvío de paquetes activo (`net.ipv4.ip_forward = 1`).*

3. **Enmascaramiento de Tráfico Saliente (`MASQUERADE`):** Traducción dinámica de las direcciones IP privadas de la subred `192.168.1.0/24` hacia la interfaz externa `enp0s3` en la cadena `POSTROUTING` de la tabla `nat`:

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

![Figura 3: Regla MASQUERADE aplicada en la cadena POSTROUTING](Captura%203.png)
*Figura 3: Verificación de la regla `MASQUERADE` sobre la interfaz WAN `enp0s3`.*

## ⚡ Verificación de Conectividad desde el Cliente

Comprobación desde el cliente Ubuntu (`nico-VirtualBox`) mediante pruebas ICMP (`ping`) confirmando tanto el enlace directo con la puerta de enlace interna (`192.168.1.8`) como la salida hacia Internet (`8.8.8.8`) sin pérdida de paquetes:

![Figura 4: Pruebas ICMP hacia la puerta de enlace interna y salida externa](Captura%204.png)
*Figura 4: Verificación de conectividad LAN (`192.168.1.8`) y WAN (`8.8.8.8`) desde el cliente.*

## 🔒 Configuración de Squid y Listas de Control de Acceso (ACL)

1. **Lista Negra en Fichero Externo (`/etc/squid/sitios_bloqueados.txt`):** Definición de los dominios restringidos anteponiendo un punto inicial (`.facebook.com`, `.instagram.com`, `.marca.com`) para abarcar tanto el dominio principal como cualquier subdominio asociado.

![Figura 5: Definición de dominios bloqueados en /etc/squid/sitios_bloqueados.txt](Captura%205.png)
*Figura 5: Listado de dominios restringidos en `/etc/squid/sitios_bloqueados.txt`.*

2. **Orden de Evaluación de Reglas en `/etc/squid/squid.conf`:** Declaración de las ACL de origen (`mi_red_prueba`) y destino (`webs_bloqueadas`), situando la regla `http_access deny` por encima de `http_access allow` para garantizar el bloqueo selectivo antes de permitir la navegación general de la LAN:

```squid
acl mi_red_prueba src 192.168.1.0/24
acl webs_bloqueadas dstdomain "/etc/squid/sitios_bloqueados.txt"

http_access deny mi_red_prueba webs_bloqueadas
http_access allow mi_red_prueba
```

![Figura 6: Reglas ACL y directivas http_access en /etc/squid/squid.conf](Captura%206.png)
*Figura 6: Configuración de las ACL y orden de prioridad en `/etc/squid/squid.conf`.*

3. **Estado del Servicio:** Reinicio y comprobación del demonio mediante `sudo systemctl status squid` confirmando su ejecución activa (`active (running)`) y habilitación en el arranque (`enabled`).

![Figura 7: Estado activo del servicio squid.service](Captura%207.png)
*Figura 7: Verificación del servicio `squid.service` activo y habilitado.*

## 🌐 Interceptación Transparente y Persistencia del Firewall

Para forzar el paso obligatorio del tráfico web HTTP (puerto `80`) de la LAN hacia el puerto de intercepción de Squid (`http_port 3129 intercept`) sin requerir configuración manual de proxy en los clientes, se implementó una regla de redirección en `PREROUTING` y se guardaron todas las reglas con `iptables-persistent` (`sudo netfilter-persistent save`):

```bash
sudo iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 80 -j REDIRECT --to-port 3129
```

![Figura 8: Regla REDIRECT en PREROUTING hacia el puerto 3129 de Squid](Captura%208.png)
*Figura 8: Redirección del tráfico TCP/80 entrante por `enp0s8` al puerto `3129`.*

## ✅ Validación Final de Filtrado en el Cliente

Pruebas de cabeceras HTTP con `curl -I` desde `nico-VirtualBox` sin proxy manual configurado, demostrando que `squid/6.13` intercepta las peticiones de forma transparente y devuelve `403 Forbidden` (`ERR_ACCESS_DENIED`) para los dominios restringidos mientras permite con `200 OK` el resto de la navegación web:

```bash
curl -I [http://www.marca.com](http://www.marca.com)
curl -I [http://example.com](http://example.com)
```

![Figura 9: Respuesta HTTP 403 Forbidden en dominio bloqueado y 200 OK en permitido](Captura%209.png)
*Figura 9: Contraste de respuestas HTTP `403 Forbidden` (`marca.com`) y `200 OK` (`example.com`) a través de `practicasad`.*
