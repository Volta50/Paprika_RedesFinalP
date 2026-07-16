# Implementación de Servidor DNS Centralizado (BIND9) en GNS3

**Topología:** la misma del proyecto — 4 sedes (Bogotá, Santa Marta, Cúcuta, Barranquilla) · EIGRP 100 · HSRP · NAT/BGP hacia ISP.

**Servidor DNS:** contenedor Docker (`gns3/srv-dns`), IP estática `10.4.192.11`, alojado en VLAN 40 (SERVIDORES) de Bogotá, conectado a **SW1-BOG f1/5** (puerto de acceso VLAN 40 libre según el Excel).

**Dominio interno:** `redes2026.local`

---

## 1. Objetivo

Montar un servidor DNS autoritativo para el dominio interno `redes2026.local` usando `bind9` en un contenedor Debian dentro de GNS3, con:

- **Zona directa** (`redes2026.local`) con registros A para los servicios de la red (dhcp, dns, web, ftp, ldap, impresión).
- **Zona inversa** (`10.in-addr.arpa`) para resolución PTR de las IPs internas.
- **Forwarders** hacia 8.8.8.8 / 1.1.1.1 para resolver nombres externos (saliendo por NAT/BGP).
- Integración con el DHCP existente: los clientes reciben `10.4.192.11` como DNS vía `option domain-name-servers`.

A diferencia del DHCP, el DNS **no necesita relay ni `ip helper-address`**: los clientes le hablan directamente por unicast al puerto 53 (UDP/TCP), y EIGRP ya sabe llegar a `10.4.192.0/18` desde todas las sedes. Lo único que hay que garantizar es enrutamiento (ya probado con el DHCP).

---

## 2. Arquitectura

```
Cliente (VPCS/PC, IP por DHCP con DNS=10.4.192.11)
   → consulta DNS unicast al puerto 53
   → gateway HSRP (VIP) de su VLAN
   → EIGRP 100 → WAN → R1-BOG
   → SW1-BOG f1/5 → contenedor Srv-DNS (10.4.192.11)
```

Puntos clave:

- Misma VLAN 40 de Bogotá donde vive el DHCP (`10.4.192.10`). El DNS toma la `.11`.
- Default gateway del contenedor: `10.4.192.2` (SVI real de SW1-BOG), igual criterio que el DHCP — predecible en pruebas; cambiar a la VIP `10.4.192.1` si se quiere resiliencia ante falla de SW1.
- Para nombres externos (`google.com`), bind reenvía a `8.8.8.8` saliendo por la ruta default → NAT/PAT en R1-BOG → ISP.

---

## 3. Plan de nombres (zona directa)

| FQDN | IP | Rol |
|---|---|---|
| dns.redes2026.local | 10.4.192.11 | Este servidor (BOG VLAN 40) |
| dhcp.redes2026.local | 10.4.192.10 | Srv-DHCP (BOG VLAN 40) |
| ldap.bog.redes2026.local | 10.4.192.12 | LDAP Bogotá |
| web.sma.redes2026.local | 10.1.128.10 | WEB Santa Marta (VLAN 40 SMA) |
| ldap.sma.redes2026.local | 10.1.128.11 | LDAP Santa Marta |
| print.sma.redes2026.local | 10.1.128.12 | Impresión Santa Marta |
| web.cuc.redes2026.local | 10.4.64.10 | WEB/FTP Cúcuta (VLAN 40 CUC) |
| ldap.cuc.redes2026.local | 10.4.64.11 | LDAP Cúcuta |
| ldap.baq.redes2026.local | 10.5.0.10 | LDAP Barranquilla (VLAN 40 BAQ) |
| ssh.baq.redes2026.local | 10.5.0.11 | SSH Barranquilla |

> Ajusta las IPs de los demás servidores a medida que los montes; todos deben caer dentro de la VLAN 40 de su sede. También se definen alias (CNAME) `www` y `ftp` apuntando a los servidores web/ftp.

---

## 4. Archivos de configuración

Ubicación en el host: `~/gns3-servers/dns/`

```bash
mkdir -p ~/gns3-servers/dns && cd ~/gns3-servers/dns
```

### 4.1 `named.conf.options`

```bash
cat > named.conf.options << 'EOF'
options {
    directory "/var/cache/bind";

    // Solo la red interna 10.0.0.0/8 puede consultar y usar recursión
    allow-query { 10.0.0.0/8; 127.0.0.1; };
    recursion yes;
    allow-recursion { 10.0.0.0/8; 127.0.0.1; };

    // Resolución externa vía NAT/BGP
    forwarders { 8.8.8.8; 1.1.1.1; };
    forward only;

    // GNS3/laboratorio: sin DNSSEC para evitar fallos de validación
    dnssec-validation no;

    listen-on { 10.4.192.11; 127.0.0.1; };
    listen-on-v6 { none; };
};
EOF
```

### 4.2 `named.conf.local`

```bash
cat > named.conf.local << 'EOF'
zone "redes2026.local" {
    type master;
    file "/etc/bind/db.redes2026.local";
};

// Zona inversa para todo 10.0.0.0/8 (cubre todas las sedes)
zone "10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.10";
};
EOF
```

### 4.3 Zona directa `db.redes2026.local`

```bash
cat > db.redes2026.local << 'EOF'
$TTL 604800
@   IN  SOA dns.redes2026.local. admin.redes2026.local. (
        2026071602 ; Serial (fecha + nn — incrementar en cada cambio)
        604800     ; Refresh
        86400      ; Retry
        2419200    ; Expire
        604800 )   ; Negative Cache TTL

@       IN  NS   dns.redes2026.local.
@       IN  A    10.4.192.11   ; apex: permite resolver el dominio "pelado" (ping redes2026.local)

; ---- Bogotá (VLAN 40: 10.4.192.0/18) ----
dns         IN  A   10.4.192.11
dhcp        IN  A   10.4.192.10
ldap.bog    IN  A   10.4.192.12

; ---- Santa Marta (VLAN 40: 10.1.128.0/17) ----
web.sma     IN  A   10.1.128.10
ldap.sma    IN  A   10.1.128.11
print.sma   IN  A   10.1.128.12

; ---- Cúcuta (VLAN 40: 10.4.64.0/18) ----
web.cuc     IN  A   10.4.64.10
ldap.cuc    IN  A   10.4.64.11

; ---- Barranquilla (VLAN 40: 10.5.0.0/18) ----
ldap.baq    IN  A   10.5.0.10
ssh.baq     IN  A   10.5.0.11

; ---- Alias de conveniencia ----
www         IN  CNAME  web.sma.redes2026.local.
ftp         IN  CNAME  web.cuc.redes2026.local.
EOF
```

### 4.4 Zona inversa `db.10`

Los PTR en `10.in-addr.arpa` van con los **octetos invertidos** (10.4.192.11 → `11.192.4`).

```bash
cat > db.10 << 'EOF'
$TTL 604800
@   IN  SOA dns.redes2026.local. admin.redes2026.local. (
        2026071601 ; Serial
        604800 86400 2419200 604800 )

@   IN  NS   dns.redes2026.local.

11.192.4    IN  PTR  dns.redes2026.local.
10.192.4    IN  PTR  dhcp.redes2026.local.
12.192.4    IN  PTR  ldap.bog.redes2026.local.
10.128.1    IN  PTR  web.sma.redes2026.local.
11.128.1    IN  PTR  ldap.sma.redes2026.local.
12.128.1    IN  PTR  print.sma.redes2026.local.
10.64.4     IN  PTR  web.cuc.redes2026.local.
11.64.4     IN  PTR  ldap.cuc.redes2026.local.
10.0.5      IN  PTR  ldap.baq.redes2026.local.
11.0.5      IN  PTR  ssh.baq.redes2026.local.
EOF
```

---

## 5. `Dockerfile.dns`

Mismo patrón que el DHCP: IP estática en runtime, demonio en background, shell interactiva al final (sin `-f`/`-g` para no bloquear la consola de GNS3, y `;` en vez de `&&` para llegar siempre al bash).

```bash
cd ~/gns3-servers/dns
cat > Dockerfile.dns << 'EOF'
FROM debian:12-slim
RUN apt-get update && apt-get install -y bind9 bind9utils dnsutils iputils-ping net-tools iproute2 && apt-get clean
COPY named.conf.options /etc/bind/named.conf.options
COPY named.conf.local   /etc/bind/named.conf.local
COPY db.redes2026.local /etc/bind/db.redes2026.local
COPY db.10              /etc/bind/db.10
CMD ifconfig eth0 10.4.192.11 netmask 255.255.192.0 up; \
    route add default gw 10.4.192.2; \
    /usr/sbin/named -u bind; \
    exec /bin/bash
EOF
```

| Instrucción | Función |
|---|---|
| `ifconfig eth0 10.4.192.11 ...` | IP estática del servidor DNS en la VLAN 40 (/18) al arrancar. |
| `route add default gw 10.4.192.2` | Default hacia SW1-BOG (misma decisión que el DHCP; cambiar a `10.4.192.1` para depender de la VIP HSRP). |
| `/usr/sbin/named -u bind` | Arranca BIND9 **en background** (sin `-g`/`-f`), corriendo como usuario `bind`. |
| `exec /bin/bash` | Deja una CLI usable en la consola de GNS3. |

### Validar sintaxis antes del build (opcional pero recomendado)

Si tienes bind9utils en el host:

```bash
named-checkconf named.conf.local
named-checkzone redes2026.local db.redes2026.local
named-checkzone 10.in-addr.arpa db.10
```

(Dentro del contenedor también se pueden ejecutar tras el arranque.)

---

## 6. Build y despliegue en GNS3

```bash
cd ~/gns3-servers/dns
docker build -t gns3/srv-dns -f Dockerfile.dns .
```

En GNS3:

1. Crear plantilla Docker nueva con la imagen `gns3/srv-dns` (1 NIC). Verificar que **Start command** esté vacío para no sobrescribir el `CMD`.
2. Conectar la NIC a **SW1-BOG f1/5** (puerto de acceso VLAN 40 libre; f1/4 de SW1 también es VLAN 40 si f1/5 está ocupado).
3. Iniciar el nodo.

---

## 7. Verificación de arranque

En la consola del contenedor (debe aparecer un prompt de shell):

```bash
ifconfig eth0                 # IP 10.4.192.11/18
route -n                      # default vía 10.4.192.2
pidof named                   # named corriendo en background (debian:12-slim no trae ps)
named-checkconf               # sin salida = config OK
ping -c 4 10.4.192.2          # conectividad a SW1-BOG
ping -c 4 10.4.192.10         # ve al Srv-DHCP en la misma VLAN

# Pruebas locales de resolución:
dig @127.0.0.1 dns.redes2026.local +short        # → 10.4.192.11
dig @127.0.0.1 -x 10.4.192.10 +short             # → dhcp.redes2026.local.
dig @127.0.0.1 google.com +short                 # requiere NAT/BGP funcionando
```

Si `named` no arranca, revisar `named-checkconf` y `named-checkzone` (errores de sintaxis en zonas son la causa típica), o arrancarlo temporalmente con `-g` para ver el log en primer plano.

---

## 8. Integrar con el DHCP existente

Editar `~/gns3-servers/dhcpd.conf` y reemplazar en **todos** los bloques `subnet`:

```
option domain-name-servers 8.8.8.8, 1.1.1.1;
```

por:

```
option domain-name-servers 10.4.192.11;
option domain-name "redes2026.local";
```

Luego reconstruir y reiniciar el nodo DHCP:

```bash
cd ~/gns3-servers
docker build -t gns3/srv-dhcp -f Dockerfile.dhcp .
# En GNS3: stop/start del nodo Srv-DHCP (o recrearlo)
```

Los clientes que renueven su lease (`ip dhcp` de nuevo en VPCS) recibirán el DNS interno.

---

## 9. Prueba funcional end-to-end

Desde un VPCS en cualquier VLAN de usuario (VPCS no tiene resolver completo, pero soporta `ip dns` y ping por nombre):

```
VPCS> ip dhcp                      # renueva; ahora DNS=10.4.192.11
VPCS> show ip                      # confirmar campo DNS
VPCS> ping redes2026.local         # resuelve el ápex (A en @) → 10.4.192.11
VPCS> ping dhcp.redes2026.local    # VPCS resuelve vía el DNS asignado
VPCS> ping dns.redes2026.local
```

Con un cliente Docker/VM Linux la prueba es más completa:

```bash
nslookup web.cuc.redes2026.local 10.4.192.11
dig @10.4.192.11 -x 10.4.192.11 +short
dig @10.4.192.11 www.redes2026.local        # sigue el CNAME
```

En el servidor se puede observar el tráfico en vivo:

```bash
rndc querylog on          # activa log de consultas
tail -f /var/log/syslog   # o journalctl si aplica
```

---

## 10. Checklist DNS (post-implementación)

- [ ] `named` corre en background y `named-checkconf` limpio.
- [ ] Resolución directa e inversa desde el propio servidor (`dig @127.0.0.1`).
- [ ] Resolución desde clientes de las 4 sedes (prueba al menos una VLAN por ciudad — valida enrutamiento de retorno EIGRP, igual que con DHCP).
- [ ] Clientes DHCP reciben `10.4.192.11` como DNS tras renovar lease.
- [ ] Resolución externa (`dig google.com`) — depende de NAT/BGP; si falla, verificar `show ip nat translations` y `show ip bgp summary` en R1-BOG.
- [ ] Failover HSRP: si la default del contenedor apunta a la VIP, apagar SW1-BOG y repetir consultas remotas.
- [ ] Incrementar el **Serial** del SOA cada vez que se edite una zona (y `rndc reload` o reiniciar el nodo).

> **Quirk conocido GNS3 (c3725 + NM-16ESW) — VIP HSRP no responde:** si un cliente resuelve ARP de la VIP a la vMAC `0000.0c07.acXX` pero los pings a la VIP hacen timeout (mientras las SVIs reales `.2`/`.3` sí responden), es el bug de forwarding de MAC virtual del NM-16ESW. Solución: `standby <grupo> use-bia` en la SVI de **ambos** switches de la sede (todas las VLANs), y `clear arp` en los clientes. El Active pasa a responder ARP por la VIP con su MAC física; el failover sigue funcionando vía gratuitous ARP.

---

## 11. Datos de referencia

- **Servidor DNS:** `10.4.192.11` (contenedor `gns3/srv-dns`, VLAN 40 Bogotá, SW1-BOG f1/5)
- **Servidor DHCP:** `10.4.192.10` (ya implementado)
- **Dominio:** `redes2026.local` · zona inversa `10.in-addr.arpa`
- **Forwarders externos:** 8.8.8.8, 1.1.1.1 (vía NAT/BGP)
- **Credenciales de red:** `Redes2026`
