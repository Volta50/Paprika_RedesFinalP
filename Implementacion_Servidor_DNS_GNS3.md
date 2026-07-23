# Servidor DNS centralizado con BIND9 en GNS3

Misma topología del proyecto: 4 sedes, EIGRP 100, HSRP y salida a Internet con NAT y BGP.

El servidor DNS es un contenedor Docker (`gns3/srv-dns`) con IP estática `10.4.192.11`, ubicado en la VLAN 40 (SERVIDORES) de Bogotá y conectado a **SW1-BOG f1/5**, puerto de acceso de esa VLAN.

Dominio interno: `redes2026.local`

## 1. Objetivo

Montar un DNS autoritativo para `redes2026.local` con `bind9` sobre Debian dentro de GNS3, con:

- Zona directa con registros A para los servicios de la red (dhcp, dns, web, ftp, ldap, impresión).
- Zona inversa `10.in-addr.arpa` para resolución PTR de las IPs internas.
- Forwarders hacia 8.8.8.8 y 1.1.1.1 para nombres externos, saliendo por NAT y BGP.
- Integración con el DHCP: los clientes reciben `10.4.192.11` como servidor DNS vía `option domain-name-servers`.

A diferencia del DHCP, el DNS no necesita relay ni `ip helper-address`. Los clientes le hablan directamente por unicast al puerto 53 y EIGRP ya sabe llegar a `10.4.192.0/18` desde todas las sedes, lo cual quedó comprobado con el DHCP.

## 2. Arquitectura

```
Cliente (VPCS/PC, con DNS=10.4.192.11 por DHCP)
   -> consulta unicast al puerto 53
   -> gateway HSRP de su VLAN
   -> EIGRP 100 -> WAN -> R1-BOG
   -> SW1-BOG f1/5 -> contenedor Srv-DNS (10.4.192.11)
```

El DNS comparte la VLAN 40 de Bogotá con el DHCP (`10.4.192.10`) y toma la `.11`. El gateway del contenedor es `10.4.192.2`, la SVI real de SW1-BOG, con el mismo criterio que el DHCP. Para los nombres externos, BIND reenvía a 8.8.8.8 por la ruta por defecto y sale traducido en R1-BOG.

## 3. Plan de nombres

| FQDN | IP | Rol |
|---|---|---|
| dns.redes2026.local | 10.4.192.11 | Este servidor (BOG VLAN 40) |
| dhcp.redes2026.local | 10.4.192.10 | Srv-DHCP (BOG VLAN 40) |
| ldap.bog.redes2026.local | 10.4.192.12 | LDAP Bogotá |
| web.sma.redes2026.local | 10.1.128.10 | WEB Santa Marta |
| ldap.sma.redes2026.local | 10.1.128.11 | LDAP Santa Marta |
| print.sma.redes2026.local | 10.1.128.12 | Impresión Santa Marta |
| web.cuc.redes2026.local | 10.4.64.10 | WEB/FTP Cúcuta |
| ldap.cuc.redes2026.local | 10.4.64.11 | LDAP Cúcuta |
| ldap.baq.redes2026.local | 10.5.0.10 | LDAP Barranquilla |
| ssh.baq.redes2026.local | 10.5.0.11 | SSH Barranquilla |

Todos los servidores caen dentro de la VLAN 40 de su sede. Se definieron además los alias `www` y `ftp` como CNAME hacia los servidores web y ftp.

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

    // Solo la red interna puede consultar y usar recursión
    allow-query { 10.0.0.0/8; 127.0.0.1; };
    recursion yes;
    allow-recursion { 10.0.0.0/8; 127.0.0.1; };

    // Resolución externa vía NAT/BGP
    forwarders { 8.8.8.8; 1.1.1.1; };
    forward only;

    // Sin DNSSEC en el laboratorio, para evitar fallos de validación
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

// Zona inversa para todo 10.0.0.0/8, cubre las cuatro sedes
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
        2026071602 ; Serial (fecha + nn, se incrementa en cada cambio)
        604800     ; Refresh
        86400      ; Retry
        2419200    ; Expire
        604800 )   ; Negative Cache TTL

@       IN  NS   dns.redes2026.local.
@       IN  A    10.4.192.11   ; apex, permite resolver el dominio sin subdominio

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

; ---- Alias ----
www         IN  CNAME  web.sma.redes2026.local.
ftp         IN  CNAME  web.cuc.redes2026.local.
EOF
```

### 4.4 Zona inversa `db.10`

Los PTR en `10.in-addr.arpa` llevan los octetos invertidos (10.4.192.11 queda como `11.192.4`).

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

## 5. `Dockerfile.dns`

Se siguió el mismo patrón del DHCP: IP estática en tiempo de ejecución, demonio en background y shell interactiva al final, sin `-f` ni `-g` para no bloquear la consola de GNS3, y con `;` en lugar de `&&` entre instrucciones.

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

`named` corre como usuario `bind` y en segundo plano, y `exec /bin/bash` deja la consola de GNS3 con un prompt utilizable.

Validación de sintaxis antes del build, si el host tiene bind9utils instalado:

```bash
named-checkconf named.conf.local
named-checkzone redes2026.local db.redes2026.local
named-checkzone 10.in-addr.arpa db.10
```

Los mismos comandos se pueden correr dentro del contenedor ya arrancado.

## 6. Build y despliegue

```bash
cd ~/gns3-servers/dns
docker build -t gns3/srv-dns -f Dockerfile.dns .
```

En GNS3:

1. Se crea una plantilla Docker con la imagen `gns3/srv-dns` y 1 NIC, dejando el campo **Start command** vacío para no sobrescribir el `CMD`.
2. Se conecta la NIC a **SW1-BOG f1/5**. El puerto f1/4 de SW1 también está en la VLAN 40 si f1/5 está ocupado.
3. Se inicia el nodo.

## 7. Verificación del arranque

En la consola del contenedor:

```bash
ifconfig eth0                 # IP 10.4.192.11/18
route -n                      # default vía 10.4.192.2
pidof named                   # debian:12-slim no trae ps
named-checkconf               # sin salida significa configuración correcta
ping -c 4 10.4.192.2          # conectividad a SW1-BOG
ping -c 4 10.4.192.10         # alcance al Srv-DHCP en la misma VLAN

dig @127.0.0.1 dns.redes2026.local +short        # 10.4.192.11
dig @127.0.0.1 -x 10.4.192.10 +short             # dhcp.redes2026.local.
dig @127.0.0.1 google.com +short                 # depende de NAT/BGP
```

Cuando `named` no arranca, la causa suele estar en la sintaxis de las zonas. Se revisa con `named-checkconf` y `named-checkzone`, o arrancando el demonio temporalmente con `-g` para ver el log en primer plano.

## 8. Integración con el DHCP

En `~/gns3-servers/dhcpd.conf` se reemplazó, en todos los bloques `subnet`:

```
option domain-name-servers 8.8.8.8, 1.1.1.1;
```

por:

```
option domain-name-servers 10.4.192.11;
option domain-name "redes2026.local";
```

Luego se reconstruyó la imagen del DHCP y se reinició el nodo:

```bash
cd ~/gns3-servers
docker build -t gns3/srv-dhcp -f Dockerfile.dhcp .
```

Los clientes reciben el DNS interno al renovar el lease.

## 9. Prueba end-to-end

Desde un VPCS. No tiene resolutor completo, pero soporta `ip dns` y ping por nombre:

```
VPCS> ip dhcp
VPCS> show ip
VPCS> ping redes2026.local
VPCS> ping dhcp.redes2026.local
VPCS> ping dns.redes2026.local
```

Con un cliente Docker o una VM Linux la prueba es más completa:

```bash
nslookup web.cuc.redes2026.local 10.4.192.11
dig @10.4.192.11 -x 10.4.192.11 +short
dig @10.4.192.11 www.redes2026.local
```

Para observar el tráfico en el servidor:

```bash
rndc querylog on
tail -f /var/log/syslog
```

## 10. Verificaciones realizadas

- `named` corriendo en background y `named-checkconf` sin errores.
- Resolución directa e inversa desde el propio servidor con `dig @127.0.0.1`.
- Resolución desde clientes de las cuatro sedes, al menos una VLAN por ciudad, lo que valida el enrutamiento de retorno de EIGRP.
- Clientes DHCP recibiendo `10.4.192.11` como DNS tras renovar el lease.
- Resolución externa con `dig google.com`, dependiente de NAT y BGP. Cuando falla se revisa `show ip nat translations` y `show ip bgp summary` en R1-BOG.
- Failover de HSRP, con la ruta por defecto del contenedor apuntando a la VIP y SW1-BOG apagado.
- Incremento del Serial del SOA en cada edición de zona, seguido de `rndc reload` o reinicio del nodo.

### Problema encontrado con la VIP de HSRP en c3725 + NM-16ESW

En algunos casos el cliente resolvía el ARP de la VIP a la MAC virtual `0000.0c07.acXX` pero los pings a la VIP hacían timeout, mientras las SVIs reales `.2` y `.3` sí respondían. Es un problema de forwarding de MAC virtual del NM-16ESW en GNS3.

La solución aplicada fue `standby <grupo> use-bia` en las SVIs de los dos switches de la sede, en todas las VLANs, y `clear arp` en los clientes. Con eso el switch activo responde ARP por la VIP con su MAC física y el failover sigue funcionando mediante gratuitous ARP.

## 11. Datos de referencia

- Servidor DNS: `10.4.192.11` (contenedor `gns3/srv-dns`, VLAN 40 Bogotá, SW1-BOG f1/5)
- Servidor DHCP: `10.4.192.10`
- Dominio: `redes2026.local`, zona inversa `10.in-addr.arpa`
- Forwarders externos: 8.8.8.8 y 1.1.1.1, vía NAT y BGP
- Credenciales de red: `Redes2026`
