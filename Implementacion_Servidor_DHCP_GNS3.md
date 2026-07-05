# Implementación de Servidor DHCP Centralizado en GNS3

**Topología:** 4 sedes (Bogotá, Santa Marta, Cúcuta, Barranquilla) · 4 routers c7200 · 8 switches multicapa c3725 · EIGRP 100 · Frame Relay / HDLC / Metro-Ethernet · HSRP · NAT/BGP hacia ISP.

**Servidor DHCP:** contenedor Docker (`gns3/srv-dhcp`), IP estática `10.4.192.10`, alojado en VLAN 40 (SERVIDORES) de Bogotá.

---

## 1. Objetivo

Centralizar la asignación de direcciones IP para los clientes de las 11 subredes de usuario (VLANs 10/20/30 de cada sede), usando un único servidor `isc-dhcp-server` corriendo en un contenedor Docker dentro de GNS3, con relay (`ip helper-address`) configurado en las SVIs de cada switch multicapa.

---

## 2. Arquitectura del relay

El servidor DHCP **no está conectado directamente a las VLANs de usuario**. Las peticiones de los clientes llegan por *DHCP relay*:

```
Cliente (VPCS/PC) → puerto de acceso en SW1/SW2 (VLAN de usuario)
                  → SVI de esa VLAN (con ip helper-address 10.4.192.10)
                  → EIGRP 100 hacia R1 de la sede
                  → WAN (Frame Relay / HDLC / Metro-E) hacia R1-BOG
                  → SW2-BOG f1/4 → contenedor Srv-DHCP (10.4.192.10)
```

**Puntos clave de la arquitectura:**

- El *default gateway* real de cada VLAN de usuario es la **IP virtual de HSRP (VIP)** en el switch multicapa, no el router. Por eso el `ip helper-address` va en la SVI del switch, no en el router.
- El helper debe configurarse en **ambos switches** (SW1 y SW2) de cada sede, para que el relay siga funcionando sin importar cuál de los dos sea el activo HSRP en ese momento.
- La VLAN 40 (SERVIDORES) **nunca** lleva `ip helper-address`, porque ahí vive el propio servidor DHCP con IP estática.
- El servidor solo ve las peticiones ya "unicast" reenviadas por el switch (`giaddr` = IP de la SVI relayadora); `dhcpd` usa ese `giaddr` para decidir qué bloque `subnet {}` de `dhcpd.conf` le corresponde.

---

## 3. Direccionamiento de referencia (VLANs de usuario)

| Ciudad | VLAN | Nombre | Subred | Máscara | Gateway HSRP (VIP) | Helper DHCP |
|---|---|---|---|---|---|---|
| BOG | 10 | INTERNET-DEMANDA | 10.3.64.0 | /18 | 10.3.64.1 | 10.4.192.10 |
| BOG | 20 | INGENIERIA | 10.3.192.0 | /18 | 10.3.192.1 | 10.4.192.10 |
| BOG | 30 | AREA-TECNICA | 10.5.64.0 | /19 | 10.5.64.1 | 10.4.192.10 |
| BOG | 40 | SERVIDORES | 10.4.192.0 | /18 | 10.4.192.1 | — (aquí vive el servidor) |
| SMA | 10 | BANDA-ANCHA | 10.0.0.0 | /16 | 10.0.0.1 | 10.4.192.10 |
| SMA | 20 | MARKETING | 10.2.128.0 | /17 | 10.2.128.1 | 10.4.192.10 |
| SMA | 30 | INGENIERIA | 10.4.128.0 | /18 | 10.4.128.1 | 10.4.192.10 |
| SMA | 40 | SERVIDORES | 10.1.128.0 | /17 | 10.1.128.1 | — |
| CUC | 10 | MARKETING | 10.1.0.0 | /17 | 10.1.0.1 | 10.4.192.10 |
| CUC | 20 | AREA-TECNICA | 10.3.0.0 | /18 | 10.3.0.1 | 10.4.192.10 |
| CUC | 30 | INTERNET-DEMANDA | 10.3.128.0 | /18 | 10.3.128.1 | 10.4.192.10 |
| CUC | 40 | SERVIDORES | 10.4.64.0 | /18 | 10.4.64.1 | — |
| BAQ | 10 | FINANZAS | 10.2.0.0 | /17 | 10.2.0.1 | 10.4.192.10 |
| BAQ | 20 | MANTENIMIENTO | 10.4.0.0 | /18 | 10.4.0.1 | 10.4.192.10 |
| BAQ | 40 | SERVIDORES | 10.5.0.0 | /18 | 10.5.0.1 | — |

> Nota: Barranquilla no tiene VLAN 30 de usuario.

---

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

**Por qué el bloque `10.4.192.0/18` está vacío:** es la subred propia del servidor (donde vive la interfaz `eth0` con IP estática `10.4.192.10`). `dhcpd` exige que la subred local de la interfaz esté declarada en el archivo, aunque no tenga rango de asignación, o el demonio se niega a levantar el relay en esa interfaz.

---

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

### Explicación de cada línea del `CMD`

| Instrucción | Función |
|---|---|
| `ifconfig eth0 10.4.192.10 ...` | Asigna la IP estática del servidor al arrancar el contenedor (no persiste en la imagen, se aplica en tiempo de ejecución). |
| `route add default gw 10.4.192.2` | Ruta por defecto del servidor hacia SW1-BOG (SVI real de VLAN 40). Ver nota de la sección 7 sobre por qué se dejó así y qué alternativa existe. |
| `dhcpd -cf /etc/dhcp/dhcpd.conf eth0` | Arranca el demonio DHCP **en segundo plano** (sin `-f`), escuchando en `eth0`. |
| `exec /bin/bash` | Reemplaza el proceso actual (PID 1) por una shell interactiva, para que la consola de GNS3 muestre un CLI usable en vez de quedar en blanco. |
| `;` en vez de `&&` | Si alguna instrucción anterior falla (por ejemplo, la ruta ya existe), las siguientes igual se ejecutan — así siempre se llega a la shell para poder diagnosticar. |

> **Importante — por qué NO se usa `-f`:** con `-f`, `dhcpd` corre en primer plano y se convierte en el PID 1 del contenedor. Eso deja la consola de GNS3 "pegada" al log del demonio, sin prompt interactivo. Quitando `-f`, `dhcpd` se demoniza solo y el `bash` final queda como proceso principal, entregando una CLI real.

---

## 6. Build y despliegue en GNS3

```bash
cd ~/gns3-servers
docker build -t gns3/srv-dhcp -f Dockerfile.dhcp .
```

En GNS3:

1. Si ya existe un nodo con la imagen vieja: clic derecho → **Delete**, y arrastrar un nodo nuevo desde la plantilla `gns3/srv-dhcp` (o simplemente detener y reiniciar el nodo existente para que tome el `CMD` nuevo).
2. Verificar que en la plantilla del nodo (Edit template → Advanced) el campo **Start command** esté vacío — si tiene algo escrito ahí, **sobrescribe** el `CMD` del Dockerfile.
3. Conectar la única NIC del contenedor a **SW2-BOG f1/4** (puerto de acceso VLAN 40, confirmado con line-protocol up).
4. Iniciar el nodo.

---

## 7. Verificación de arranque del servidor

Al abrir la consola del nodo debe aparecer un **prompt de shell**, no un log en blanco ni texto corriendo sin parar.

```bash
ifconfig eth0            # confirma IP 10.4.192.10/18 asignada
route -n                 # confirma ruta default
ps aux | grep dhcpd      # confirma que el demonio quedó corriendo en background
ping -c 4 10.4.192.2     # confirma conectividad hacia SW1-BOG
```

> **Nota sobre la ruta default:** el `Dockerfile.dhcp` apunta la ruta default a `10.4.192.2` (IP real de SW1-BOG), evitando deliberadamente la VIP de HSRP (`10.4.192.1`) para no depender del estado de HSRP en el arranque. Si se prefiere que el servidor sobreviva a una falla de SW1 sin intervención manual, cambiar a la VIP:
> ```bash
> route del default
> route add default gw 10.4.192.1
> ```
> Trade-off: apuntar a la IP real es más predecible durante pruebas; apuntar a la VIP es más resiliente en producción/simulación de fallas.

---

## 8. Configuración del relay en los switches multicapa

Se configura `ip helper-address 10.4.192.10` en la SVI de **cada VLAN de usuario**, en **ambos switches** de cada sede (SW1 y SW2), para que el relay no dependa de cuál switch sea el activo HSRP.

**NUNCA** se configura en la VLAN 40 (SERVIDORES).

### Bogotá — SW1-BOG y SW2-BOG (idéntico en ambos)
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
interface Vlan30
 ip helper-address 10.4.192.10
```

### Santa Marta — SW1-SMA y SW2-SMA
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
interface Vlan30
 ip helper-address 10.4.192.10
```

### Cúcuta — SW1-CUC y SW2-CUC
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
interface Vlan30
 ip helper-address 10.4.192.10
```

### Barranquilla — SW1-BAQ y SW2-BAQ (solo VLAN 10 y 20)
```
interface Vlan10
 ip helper-address 10.4.192.10
interface Vlan20
 ip helper-address 10.4.192.10
```

---

## 9. Verificación de ruta de retorno (EIGRP)

La respuesta del `dhcpd` (unicast hacia el `giaddr`) debe poder volver a cada SVI remota. Confirmar en cada router remoto que la subred del servidor está en la tabla EIGRP:

```
R1-SMA# show ip route eigrp | include 10.4.192.0
R1-CUC# show ip route eigrp | include 10.4.192.0
R1-BAQ# show ip route eigrp | include 10.4.192.0
```

Si no aparece, el relay nunca recibirá respuesta aunque el `DHCPDISCOVER` sí llegue al servidor (se vería el `DHCPDISCOVER` en el log pero nunca un `DHCPACK` confirmado en el cliente).

---

## 10. Prueba funcional end-to-end

Desde un VPCS/PC en cualquier VLAN de usuario:

```
VPCS> ip dhcp
VPCS> show ip
```

Resultado esperado (ejemplo real, VLAN 20 Bogotá):
```
DDORA IP 10.3.255.254/18 GW 10.3.192.1
```

`DDORA` = Discover → Offer → Request → Ack completo. La IP debe caer dentro del `range` del `dhcpd.conf` correspondiente, y el gateway debe coincidir con la VIP HSRP de esa VLAN.

En el servidor, confirmar la asignación:

```bash
cat /var/lib/dhcp/dhcpd.leases
# o en vivo:
tail -f /var/lib/dhcp/dhcpd.leases
```

Se debe ver un bloque `lease { ... binding state active; ... client-hostname "VPCS"; }` por cada cliente.

---

## 11. Checklist de verificación completa (post-implementación)

Confirmar que DHCP funcione en las 11 VLANs de usuario prueba que EIGRP y HSRP están sanos, pero **no** prueba todo lo demás. Antes de dar por cerrada la implementación, verificar también:

- [ ] **Salida a Internet / NAT / BGP:** desde un VPCS, `ping 8.8.8.8` y `ping` a la IP del ISP correspondiente (203.0.113.2 / .6 / .10 / .14). En el router: `show ip bgp summary` (vecino `Established`) y `show ip nat translations`.
- [ ] **Failover de HSRP:** `show standby brief` en SW1 y SW2 de cada sede; apagar el uplink de SW1 y repetir `ip dhcp` en un VPCS — debe seguir obteniendo IP por la VIP.
- [ ] **Spanning-Tree correcto:** `show spanning-tree` — SW1 debe ser root (prioridad 4096) y uno de los dos trunks entre SW1/SW2 debe estar en `BLK`, para evitar loops.
- [ ] **Alcance inter-VLAN e inter-sede:** ping entre VPCS de VLANs/sedes distintas (no solo al gateway), para confirmar que EIGRP lleva el tráfico de retorno correctamente.
- [ ] **Ruta default del servidor:** confirmar si se dejó apuntando a la IP real de SW1 o a la VIP (ver sección 7), según el comportamiento deseado ante fallas.

---

## 12. Resumen de credenciales y datos de referencia

- **Enable / console / vty:** `Redes2026`
- **Servidor DHCP:** `10.4.192.10` (contenedor Docker, VLAN 40 Bogotá)
- **HSRP:** VIP `.1` en cada subred · SW1 activo (prioridad 110) · SW2 standby (prioridad 100) · `preempt` en ambos
- **STP:** SW1 root (prioridad 4096) · SW2 (prioridad 8192)
- **EIGRP:** AS 100, red interna `10.0.0.0`
- **BGP:** eBGP hacia AS 65000 (ISP), un AS distinto por sede (65001 BOG, 65002 SMA, 65003 CUC, 65004 BAQ)
