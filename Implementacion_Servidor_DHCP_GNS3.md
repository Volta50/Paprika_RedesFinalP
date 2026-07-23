# Servidor DHCP centralizado en GNS3

Topología del proyecto: 4 sedes (Bogotá, Santa Marta, Cúcuta y Barranquilla), 4 routers c7200, 8 switches multicapa c3725, EIGRP 100, enlaces Frame Relay, HDLC y Metro-Ethernet, HSRP y salida a Internet con NAT y BGP.

El servidor DHCP es un contenedor Docker (`gns3/srv-dhcp`) con IP estática `10.4.192.10`, ubicado en la VLAN 40 (SERVIDORES) de Bogotá.

## 1. Objetivo

Centralizar la asignación de direcciones para los clientes de las 11 subredes de usuario (VLANs 10, 20 y 30 de cada sede) en un único `isc-dhcp-server`, usando relay (`ip helper-address`) en las SVIs de los switches multicapa.

## 2. Arquitectura del relay

El servidor no está conectado a las VLANs de usuario. Las peticiones llegan reenviadas por los switches:

```
Cliente (VPCS/PC) -> puerto de acceso en SW1/SW2 (VLAN de usuario)
                  -> SVI de esa VLAN (con ip helper-address 10.4.192.10)
                  -> EIGRP 100 hacia el R1 de la sede
                  -> WAN (Frame Relay / HDLC / Metro-E) hacia R1-BOG
                  -> SW2-BOG f1/4 -> contenedor Srv-DHCP (10.4.192.10)
```

Consideraciones del diseño:

- El gateway real de cada VLAN de usuario es la IP virtual de HSRP en el switch multicapa, no el router. Por eso el `ip helper-address` va en la SVI del switch.
- El helper se configuró en los dos switches de cada sede, para que el relay siga operando sin importar cuál sea el activo de HSRP.
- La VLAN 40 no lleva `ip helper-address`, porque ahí vive el propio servidor con IP estática.
- El servidor recibe las peticiones ya en unicast, con el campo `giaddr` igual a la IP de la SVI que hizo el relay. `dhcpd` usa ese `giaddr` para decidir qué bloque `subnet {}` aplica.

## 3. Direccionamiento de las VLANs de usuario

| Ciudad | VLAN | Nombre | Subred | Máscara | Gateway HSRP | Helper |
|---|---|---|---|---|---|---|
| BOG | 10 | INTERNET-DEMANDA | 10.3.64.0 | /18 | 10.3.64.1 | 10.4.192.10 |
| BOG | 20 | INGENIERIA | 10.3.192.0 | /18 | 10.3.192.1 | 10.4.192.10 |
| BOG | 30 | AREA-TECNICA | 10.5.64.0 | /19 | 10.5.64.1 | 10.4.192.10 |
| BOG | 40 | SERVIDORES | 10.4.192.0 | /18 | 10.4.192.1 | no aplica |
| SMA | 10 | BANDA-ANCHA | 10.0.0.0 | /16 | 10.0.0.1 | 10.4.192.10 |
| SMA | 20 | MARKETING | 10.2.128.0 | /17 | 10.2.128.1 | 10.4.192.10 |
| SMA | 30 | INGENIERIA | 10.4.128.0 | /18 | 10.4.128.1 | 10.4.192.10 |
| SMA | 40 | SERVIDORES | 10.1.128.0 | /17 | 10.1.128.1 | no aplica |
| CUC | 10 | MARKETING | 10.1.0.0 | /17 | 10.1.0.1 | 10.4.192.10 |
| CUC | 20 | AREA-TECNICA | 10.3.0.0 | /18 | 10.3.0.1 | 10.4.192.10 |
| CUC | 30 | INTERNET-DEMANDA | 10.3.128.0 | /18 | 10.3.128.1 | 10.4.192.10 |
| CUC | 40 | SERVIDORES | 10.4.64.0 | /18 | 10.4.64.1 | no aplica |
| BAQ | 10 | FINANZAS | 10.2.0.0 | /17 | 10.2.0.1 | 10.4.192.10 |
| BAQ | 20 | MANTENIMIENTO | 10.4.0.0 | /18 | 10.4.0.1 | 10.4.192.10 |
| BAQ | 40 | SERVIDORES | 10.5.0.0 | /18 | 10.5.0.1 | no aplica |

Barranquilla no tiene VLAN 30 de usuario.

## 4. Archivo `dhcpd.conf`

Ubicación en el host: `~/gns3-servers/dhcpd.conf`

```bash
cd ~/gns3-servers
cat > dhcpd.conf << 'EOF'
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 10.4.192.0 netmask 255.255.192.0 {
}

# ================= BOGOTA =================
subnet 10.3.64.0 netmask 255.255.192.0 {
  range 10.3.64.4 10.3.127.254;
  option routers 10.3.64.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
subnet 10.3.192.0 netmask 255.255.192.0 {
  range 10.3.192.4 10.3.255.254;
  option routers 10.3.192.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
subnet 10.5.64.0 netmask 255.255.224.0 {
  range 10.5.64.4 10.5.95.254;
  option routers 10.5.64.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}

# ================= SANTA MARTA =================
subnet 10.0.0.0 netmask 255.255.0.0 {
  range 10.0.0.4 10.0.255.254;
  option routers 10.0.0.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
subnet 10.2.128.0 netmask 255.255.128.0 {
  range 10.2.128.4 10.2.255.254;
  option routers 10.2.128.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
subnet 10.4.128.0 netmask 255.255.192.0 {
  range 10.4.128.4 10.4.191.254;
  option routers 10.4.128.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}

# ================= CUCUTA =================
subnet 10.1.0.0 netmask 255.255.128.0 {
  range 10.1.0.4 10.1.127.254;
  option routers 10.1.0.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
subnet 10.3.0.0 netmask 255.255.192.0 {
  range 10.3.0.4 10.3.63.254;
  option routers 10.3.0.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
subnet 10.3.128.0 netmask 255.255.192.0 {
  range 10.3.128.4 10.3.191.254;
  option routers 10.3.128.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}

# ================= BARRANQUILLA =================
subnet 10.2.0.0 netmask 255.255.128.0 {
  range 10.2.0.4 10.2.127.254;
  option routers 10.2.0.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
subnet 10.4.0.0 netmask 255.255.192.0 {
  range 10.4.0.4 10.4.63.254;
  option routers 10.4.0.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
EOF
```

El bloque `10.4.192.0/18` se declara vacío porque es la subred propia de `eth0`. `dhcpd` exige que la subred local de la interfaz esté declarada aunque no reparta direcciones; de lo contrario se niega a levantar el servicio en esa interfaz.

Los servidores DNS entregados aquí (8.8.8.8 y 1.1.1.1) se reemplazaron por el DNS interno cuando se montó BIND9. Ese cambio está documentado en la implementación del servidor DNS.

## 5. `Dockerfile.dhcp`

Ubicación en el host: `~/gns3-servers/Dockerfile.dhcp`

```dockerfile
FROM debian:12-slim
RUN apt-get update && apt-get install -y isc-dhcp-server iputils-ping net-tools iproute2 && apt-get clean
RUN touch /var/lib/dhcp/dhcpd.leases
COPY dhcpd.conf /etc/dhcp/dhcpd.conf
RUN echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
CMD ifconfig eth0 10.4.192.10 netmask 255.255.192.0 up; \
    route add default gw 10.4.192.2; \
    dhcpd -cf /etc/dhcp/dhcpd.conf eth0; \
    exec /bin/bash
```

La IP y la ruta se aplican en tiempo de ejecución dentro del `CMD`, no quedan grabadas en la imagen. La ruta por defecto apunta a `10.4.192.2`, la SVI real de SW1-BOG.

`dhcpd` se arranca sin `-f`, de modo que se demoniza y deja `bash` como proceso principal del contenedor. Con `-f` el demonio queda como PID 1 y la consola de GNS3 se queda pegada al log, sin prompt utilizable.

Las instrucciones del `CMD` van separadas por `;` y no por `&&`, para que un fallo intermedio (por ejemplo, una ruta que ya existe) no impida llegar a la shell.

## 6. Build y despliegue

```bash
cd ~/gns3-servers
docker build -t gns3/srv-dhcp -f Dockerfile.dhcp .
```

En GNS3:

1. Si ya existía un nodo con la imagen anterior, se elimina y se arrastra uno nuevo desde la plantilla `gns3/srv-dhcp`. También sirve detener y reiniciar el nodo existente para que tome el `CMD` nuevo.
2. En la plantilla del nodo (Edit template, pestaña Advanced) el campo **Start command** debe quedar vacío. Si tiene algo escrito, sobrescribe el `CMD` del Dockerfile.
3. Se conecta la NIC del contenedor a **SW2-BOG f1/4**, puerto de acceso de la VLAN 40.
4. Se inicia el nodo.

## 7. Verificación del arranque

Al abrir la consola del nodo aparece un prompt de shell.

```bash
ifconfig eth0            # IP 10.4.192.10/18
route -n                 # ruta por defecto
ps aux | grep dhcpd      # demonio corriendo en background
ping -c 4 10.4.192.2     # conectividad hacia SW1-BOG
```

Sobre la ruta por defecto: se dejó apuntando a la IP real de SW1-BOG y no a la VIP de HSRP, para no depender del estado de HSRP durante el arranque y las pruebas. Para que el servidor sobreviva a una caída de SW1 sin intervención, se cambia a la VIP:

```bash
route del default
route add default gw 10.4.192.1
```

## 8. Relay en los switches multicapa

`ip helper-address 10.4.192.10` en la SVI de cada VLAN de usuario, en los dos switches de cada sede. En la VLAN 40 no se configura.

Bogotá (SW1-BOG y SW2-BOG):
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
interface Vlan30
 ip helper-address 10.4.192.10
```

Santa Marta (SW1-SMA y SW2-SMA):
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
interface Vlan30
 ip helper-address 10.4.192.10
```

Cúcuta (SW1-CUC y SW2-CUC):
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
interface Vlan30
 ip helper-address 10.4.192.10
```

Barranquilla (SW1-BAQ y SW2-BAQ, solo VLAN 10 y 20):
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
```

## 9. Ruta de retorno

La respuesta de `dhcpd` va en unicast hacia el `giaddr`, así que la subred del servidor debe estar en la tabla de rutas de cada router remoto:

```
R1-SMA# show ip route eigrp | include 10.4.192.0
R1-CUC# show ip route eigrp | include 10.4.192.0
R1-BAQ# show ip route eigrp | include 10.4.192.0
```

Si la ruta falta, el `DHCPDISCOVER` llega al servidor y aparece en el log, pero el cliente nunca recibe el `DHCPACK`.

## 10. Prueba end-to-end

Desde un VPCS en cualquier VLAN de usuario:

```
VPCS> ip dhcp
VPCS> show ip
```

Resultado obtenido en la VLAN 20 de Bogotá:

```
DDORA IP 10.3.255.254/18 GW 10.3.192.1
```

`DDORA` indica el ciclo Discover, Offer, Request y Ack completo. La IP cae dentro del `range` correspondiente y el gateway coincide con la VIP de HSRP de esa VLAN.

En el servidor:

```bash
cat /var/lib/dhcp/dhcpd.leases
tail -f /var/lib/dhcp/dhcpd.leases
```

Por cada cliente aparece un bloque `lease { ... binding state active; ... client-hostname "VPCS"; }`.

## 11. Verificaciones adicionales

Que el DHCP funcione en las 11 VLANs comprueba que EIGRP y HSRP están sanos, pero no cubre el resto del proyecto. También se revisó:

- Salida a Internet: `ping 8.8.8.8` y ping a la IP del ISP de la sede (203.0.113.2 / .6 / .10 / .14) desde un VPCS; en el router, `show ip bgp summary` y `show ip nat translations`.
- Failover de HSRP: `show standby brief` en SW1 y SW2, apagado del uplink de SW1 y `ip dhcp` de nuevo en un VPCS.
- Spanning-Tree: `show spanning-tree`, con SW1 como root (prioridad 4096) y uno de los trunks entre SW1 y SW2 en `BLK`.
- Alcance inter-VLAN e inter-sede: ping entre VPCS de VLANs y ciudades distintas, no solo al gateway.
- Ruta por defecto del servidor, según lo indicado en la sección 7.

## 12. Datos de referencia

- Enable, consola y vty: `Redes2026`
- Servidor DHCP: `10.4.192.10` (contenedor Docker, VLAN 40 Bogotá)
- HSRP: VIP `.1` en cada subred, SW1 activo (prioridad 110), SW2 standby (prioridad 100), `preempt` en ambos
- STP: SW1 root (prioridad 4096), SW2 (prioridad 8192)
- EIGRP: AS 100, red `10.0.0.0`
- BGP: eBGP hacia AS 65000, un AS por sede (65001 BOG, 65002 SMA, 65003 CUC, 65004 BAQ)
