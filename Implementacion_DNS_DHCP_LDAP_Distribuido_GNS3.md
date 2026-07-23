# DNS distribuido, DHCP y LDAP distribuido

Se mantuvo la metodología usada con los servicios anteriores: un Dockerfile por servicio, con la IP, la ruta y el arranque del demonio dentro del `CMD`, y `exec /bin/bash` al final para tener consola utilizable en GNS3. Los nodos de WEB, FTP e impresión (imagen `base-server:v1`, configurados a mano) no se tocaron; solo se referencian desde la zona DNS.

Distribución de servidores según el diagrama del proyecto:

| Ciudad | DNS | LDAP | DHCP | Servidores previos |
|---|---|---|---|---|
| Bogotá | master `10.4.192.11` | `10.4.192.12` | `10.4.192.10` (único) | |
| Santa Marta | usa el master de BOG | `10.1.128.11` | | WEB `10.1.128.10`, IMP `10.1.128.12` |
| Cúcuta | slave `10.4.64.11` | `10.4.64.12` | | WEB/FTP `10.4.64.10` |
| Barranquilla | slave `10.5.0.12` | `10.5.0.10` | | SSH `10.5.0.11` |

Cambios de direccionamiento respecto al plan inicial, que se reflejaron en el Excel de VLSM:

- LDAP Cúcuta pasa de `10.4.64.11` a `10.4.64.12`, porque la `.11` la toma el DNS slave de esa sede.
- DNS Barranquilla es nuevo, en `10.5.0.12`.
- Los servidores ya montados conservan sus IPs y coinciden con la zona DNS existente.

Los gateways de todos los contenedores apuntan a la VIP de HSRP `.1` de la VLAN 40 de su sede. Ese criterio se adoptó después de aplicar `standby use-bia` en los switches, así que hay que confirmar que el fix esté puesto en CUC, BAQ y SMA antes de desplegar allí.

Estructura de directorios en el host:

```
~/gns3-servers/
├── dhcpd.conf, Dockerfile.dhcp        (existentes, se edita dhcpd.conf)
├── dns/                                (existente, master BOG, se actualizan zonas)
├── dns-cuc/                            (nuevo, slave)
├── dns-baq/                            (nuevo, slave)
└── ldap/                               (BOG existente + 3 Dockerfiles nuevos)
```

# Parte 1. DNS distribuido

Esquema master y slaves con transferencia de zona (AXFR) y notificaciones. Se eligió por tres razones: las zonas se editan solo en Bogotá y se propagan solas al subir el serial, si cae Bogotá los slaves siguen resolviendo con su copia local, y cada sede consulta su DNS local con menor latencia. Santa Marta no tiene DNS en el diagrama, sus clientes usan el master.

## 1.1 Master de Bogotá (`10.4.192.11`)

```bash
cd ~/gns3-servers/dns
```

`named.conf.options` no cambia respecto a la versión anterior, se deja como referencia:

```bash
cat > named.conf.options << 'EOF'
options {
    directory "/var/cache/bind";
    allow-query { 10.0.0.0/8; 127.0.0.1; };
    recursion yes;
    allow-recursion { 10.0.0.0/8; 127.0.0.1; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    forward only;
    dnssec-validation no;
    listen-on { 10.4.192.11; 127.0.0.1; };
    listen-on-v6 { none; };
};
EOF
```

`named.conf.local` declara los slaves y activa las notificaciones:

```bash
cat > named.conf.local << 'EOF'
zone "redes2026.local" {
    type master;
    file "/etc/bind/db.redes2026.local";
    allow-transfer { 10.4.64.11; 10.5.0.12; };
    also-notify { 10.4.64.11; 10.5.0.12; };
    notify yes;
};

zone "10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.10";
    allow-transfer { 10.4.64.11; 10.5.0.12; };
    also-notify { 10.4.64.11; 10.5.0.12; };
    notify yes;
};
EOF
```

Zona directa, con los registros nuevos de los DNS slaves, los LDAP por sede y la impresora de Santa Marta:

```bash
cat > db.redes2026.local << 'EOF'
$TTL 604800
@   IN  SOA dns.redes2026.local. admin.redes2026.local. (
        2026071610 ; Serial, se sube en cada cambio
        604800 86400 2419200 604800 )

@       IN  NS   dns.redes2026.local.
@       IN  NS   dns.cuc.redes2026.local.
@       IN  NS   dns.baq.redes2026.local.
@       IN  A    10.4.192.11

; ---- Servidores DNS ----
dns         IN  A   10.4.192.11
dns.bog     IN  A   10.4.192.11
dns.cuc     IN  A   10.4.64.11
dns.baq     IN  A   10.5.0.12

; ---- Bogotá (VLAN 40: 10.4.192.0/18) ----
dhcp        IN  A   10.4.192.10
ldap.bog    IN  A   10.4.192.12

; ---- Santa Marta (VLAN 40: 10.1.128.0/17) ----
web.sma     IN  A   10.1.128.10
ldap.sma    IN  A   10.1.128.11
print.sma   IN  A   10.1.128.12

; ---- Cúcuta (VLAN 40: 10.4.64.0/18) ----
web.cuc     IN  A   10.4.64.10
ldap.cuc    IN  A   10.4.64.12

; ---- Barranquilla (VLAN 40: 10.5.0.0/18) ----
ldap.baq    IN  A   10.5.0.10
ssh.baq     IN  A   10.5.0.11

; ---- Alias ----
www         IN  CNAME  web.sma.redes2026.local.
ftp         IN  CNAME  web.cuc.redes2026.local.
EOF
```

Zona inversa:

```bash
cat > db.10 << 'EOF'
$TTL 604800
@   IN  SOA dns.redes2026.local. admin.redes2026.local. (
        2026071610 ; Serial
        604800 86400 2419200 604800 )

@   IN  NS   dns.redes2026.local.
@   IN  NS   dns.cuc.redes2026.local.
@   IN  NS   dns.baq.redes2026.local.

11.192.4    IN  PTR  dns.redes2026.local.
10.192.4    IN  PTR  dhcp.redes2026.local.
12.192.4    IN  PTR  ldap.bog.redes2026.local.
10.128.1    IN  PTR  web.sma.redes2026.local.
11.128.1    IN  PTR  ldap.sma.redes2026.local.
12.128.1    IN  PTR  print.sma.redes2026.local.
10.64.4     IN  PTR  web.cuc.redes2026.local.
11.64.4     IN  PTR  dns.cuc.redes2026.local.
12.64.4     IN  PTR  ldap.cuc.redes2026.local.
10.0.5      IN  PTR  ldap.baq.redes2026.local.
11.0.5      IN  PTR  ssh.baq.redes2026.local.
12.0.5      IN  PTR  dns.baq.redes2026.local.
EOF
```

`Dockerfile.dns` del master, ya con el gateway apuntando a la VIP:

```bash
cat > Dockerfile.dns << 'EOF'
FROM debian:12-slim
RUN apt-get update && apt-get install -y bind9 bind9utils dnsutils iputils-ping net-tools iproute2 && apt-get clean
COPY named.conf.options /etc/bind/named.conf.options
COPY named.conf.local   /etc/bind/named.conf.local
COPY db.redes2026.local /etc/bind/db.redes2026.local
COPY db.10              /etc/bind/db.10
CMD ifconfig eth0 10.4.192.11 netmask 255.255.192.0 up; \
    route add default gw 10.4.192.1; \
    /usr/sbin/named -u bind; \
    exec /bin/bash
EOF
```

Build y redespliegue:

```bash
docker build -t gns3/srv-dns -f Dockerfile.dns .
```

En GNS3 se detiene y arranca el nodo Srv-DNS, o se recrea. Verificación:

```bash
named-checkconf
dig @127.0.0.1 dns.cuc.redes2026.local +short   # 10.4.64.11
dig @127.0.0.1 -x 10.4.64.12 +short             # ldap.cuc.redes2026.local.
```

## 1.2 Slave de Cúcuta (`10.4.64.11`)

Los slaves no llevan archivos de zona en la imagen. Los reciben del master por AXFR y los guardan en `/var/cache/bind/`, que es escribible.

```bash
mkdir -p ~/gns3-servers/dns-cuc && cd ~/gns3-servers/dns-cuc

cat > named.conf.options << 'EOF'
options {
    directory "/var/cache/bind";
    allow-query { 10.0.0.0/8; 127.0.0.1; };
    recursion yes;
    allow-recursion { 10.0.0.0/8; 127.0.0.1; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    forward only;
    dnssec-validation no;
    listen-on { 10.4.64.11; 127.0.0.1; };
    listen-on-v6 { none; };
};
EOF

cat > named.conf.local << 'EOF'
zone "redes2026.local" {
    type slave;
    file "db.redes2026.local";
    masters { 10.4.192.11; };
};
zone "10.in-addr.arpa" {
    type slave;
    file "db.10";
    masters { 10.4.192.11; };
};
EOF

cat > Dockerfile.dns << 'EOF'
FROM debian:12-slim
RUN apt-get update && apt-get install -y bind9 bind9utils dnsutils iputils-ping net-tools iproute2 && apt-get clean
COPY named.conf.options /etc/bind/named.conf.options
COPY named.conf.local   /etc/bind/named.conf.local
CMD ifconfig eth0 10.4.64.11 netmask 255.255.192.0 up; \
    route add default gw 10.4.64.1; \
    /usr/sbin/named -u bind; \
    exec /bin/bash
EOF

docker build -t gns3/srv-dns-cuc -f Dockerfile.dns .
```

## 1.3 Slave de Barranquilla (`10.5.0.12`)

```bash
mkdir -p ~/gns3-servers/dns-baq && cd ~/gns3-servers/dns-baq

sed 's|10.4.64.11|10.5.0.12|' ../dns-cuc/named.conf.options > named.conf.options
cp ../dns-cuc/named.conf.local named.conf.local

cat > Dockerfile.dns << 'EOF'
FROM debian:12-slim
RUN apt-get update && apt-get install -y bind9 bind9utils dnsutils iputils-ping net-tools iproute2 && apt-get clean
COPY named.conf.options /etc/bind/named.conf.options
COPY named.conf.local   /etc/bind/named.conf.local
CMD ifconfig eth0 10.5.0.12 netmask 255.255.192.0 up; \
    route add default gw 10.5.0.1; \
    /usr/sbin/named -u bind; \
    exec /bin/bash
EOF

docker build -t gns3/srv-dns-baq -f Dockerfile.dns .
```

## 1.4 Despliegue y verificación de los slaves

1. Plantillas GNS3 `gns3/srv-dns-cuc` y `gns3/srv-dns-baq`, con 1 NIC y Start command vacío.
2. Se conectan a un puerto de acceso de la VLAN 40 libre en SW1-CUC y SW1-BAQ, confirmado con `show vlan brief`.
3. Al iniciar, segundos después, en cada slave:

```bash
ls -l /var/cache/bind/                        # deben aparecer db.redes2026.local y db.10
dig @127.0.0.1 dhcp.redes2026.local +short    # 10.4.192.10, solo pudo llegar por AXFR
dig @127.0.0.1 redes2026.local SOA +short     # serial idéntico al del master
```

Si los archivos no aparecen, se revisa la conectividad al master con `ping 10.4.192.11` y en el master que `allow-transfer` tenga la IP exacta del slave.

## 1.5 Pruebas de la parte 1

| Prueba | Procedimiento | Resultado esperado |
|---|---|---|
| Propagación | Agregar `test IN A 10.4.192.99` en el master, subir el serial y hacer rebuild y restart (o `rndc reload`) | `dig test.redes2026.local` responde en ambos slaves en menos de un minuto |
| Failover | Apagar el nodo master y consultar los slaves | Siguen resolviendo toda la zona interna |
| Regresión web | `dig @10.4.64.11 web.cuc.redes2026.local +short` | 10.4.64.10 |

# Parte 2. DHCP

La arquitectura de relay no cambia, los helpers ya estaban configurados en las SVIs. Lo único que se actualizó fue el DNS que recibe cada sede: su servidor local como primario y el master de Bogotá como respaldo.

```bash
cd ~/gns3-servers
cat > dhcpd.conf << 'EOF'
default-lease-time 600;
max-lease-time 7200;
authoritative;
option domain-name "redes2026.local";

subnet 10.4.192.0 netmask 255.255.192.0 {
}

# ================= BOGOTA (DNS master local) =================
subnet 10.3.64.0 netmask 255.255.192.0 {
  range 10.3.64.4 10.3.127.254;
  option routers 10.3.64.1;
  option domain-name-servers 10.4.192.11;
}
subnet 10.3.192.0 netmask 255.255.192.0 {
  range 10.3.192.4 10.3.255.254;
  option routers 10.3.192.1;
  option domain-name-servers 10.4.192.11;
}
subnet 10.5.64.0 netmask 255.255.224.0 {
  range 10.5.64.4 10.5.95.254;
  option routers 10.5.64.1;
  option domain-name-servers 10.4.192.11;
}

# ================= SANTA MARTA (sin DNS propio, usa el master) =================
subnet 10.0.0.0 netmask 255.255.0.0 {
  range 10.0.0.4 10.0.255.254;
  option routers 10.0.0.1;
  option domain-name-servers 10.4.192.11;
}
subnet 10.2.128.0 netmask 255.255.128.0 {
  range 10.2.128.4 10.2.255.254;
  option routers 10.2.128.1;
  option domain-name-servers 10.4.192.11;
}
subnet 10.4.128.0 netmask 255.255.192.0 {
  range 10.4.128.4 10.4.191.254;
  option routers 10.4.128.1;
  option domain-name-servers 10.4.192.11;
}

# ================= CUCUTA (slave local + respaldo BOG) =================
subnet 10.1.0.0 netmask 255.255.128.0 {
  range 10.1.0.4 10.1.127.254;
  option routers 10.1.0.1;
  option domain-name-servers 10.4.64.11, 10.4.192.11;
}
subnet 10.3.0.0 netmask 255.255.192.0 {
  range 10.3.0.4 10.3.63.254;
  option routers 10.3.0.1;
  option domain-name-servers 10.4.64.11, 10.4.192.11;
}
subnet 10.3.128.0 netmask 255.255.192.0 {
  range 10.3.128.4 10.3.191.254;
  option routers 10.3.128.1;
  option domain-name-servers 10.4.64.11, 10.4.192.11;
}

# ================= BARRANQUILLA (slave local + respaldo BOG) =================
subnet 10.2.0.0 netmask 255.255.128.0 {
  range 10.2.0.4 10.2.127.254;
  option routers 10.2.0.1;
  option domain-name-servers 10.5.0.12, 10.4.192.11;
}
subnet 10.4.0.0 netmask 255.255.192.0 {
  range 10.4.0.4 10.4.63.254;
  option routers 10.4.0.1;
  option domain-name-servers 10.5.0.12, 10.4.192.11;
}
EOF
```

`Dockerfile.dhcp` con el gateway ya apuntando a la VIP:

```bash
cat > Dockerfile.dhcp << 'EOF'
FROM debian:12-slim
RUN apt-get update && apt-get install -y isc-dhcp-server iputils-ping net-tools iproute2 && apt-get clean
RUN touch /var/lib/dhcp/dhcpd.leases
COPY dhcpd.conf /etc/dhcp/dhcpd.conf
RUN echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
CMD ifconfig eth0 10.4.192.10 netmask 255.255.192.0 up; \
    route add default gw 10.4.192.1; \
    dhcpd -cf /etc/dhcp/dhcpd.conf eth0; \
    exec /bin/bash
EOF

docker build -t gns3/srv-dhcp -f Dockerfile.dhcp .
```

Después de detener y arrancar el nodo Srv-DHCP, en un VPCS de Cúcuta o Barranquilla se ejecuta `ip dhcp` y `show ip` debe mostrar el DNS local de esa sede en el campo DNS.

# Parte 3. LDAP distribuido

Cuatro servidores autónomos con el mismo árbol (`dc=redes2026,dc=local`), cargado desde el bootstrap. La actividad obligatoria pide autenticación, no replicación, y con servidores independientes cada sede sigue autenticando aunque se caiga la WAN. La contrapartida es que un usuario creado después en una sede no se propaga a las demás, lo que no afecta a las pruebas porque los usuarios son fijos. Sobre este esquema se puede montar syncrepl si se aborda el punto opcional de replicación.

## 3.1 Usuarios

| uid | Nombre | Sede de prueba | Clave |
|---|---|---|---|
| jperez | Juan Perez | BOG | ClaveDeJuan123 |
| mrodriguez | Maria Rodriguez | BOG | ClaveDeMaria456 |
| csierra | Carlos Sierra | SMA | ClaveDeCarlos789 |
| atorres | Ana Torres | CUC | ClaveDeAna012 |
| lmejia | Luis Mejia | BAQ | ClaveDeLuis345 |

Los cinco existen en las cuatro sedes. Los hashes se generaron con el LDAP de Bogotá que ya estaba corriendo:

```bash
for c in "ClaveDeJuan123" "ClaveDeMaria456" "ClaveDeCarlos789" "ClaveDeAna012" "ClaveDeLuis345"; do
  docker exec -i $(docker ps -qf "ancestor=gns3/srv-ldap") slappasswd -s "$c"
done
```

## 3.2 `bootstrap.ldif`

El mismo archivo para las cuatro imágenes:

```bash
cd ~/gns3-servers/ldap
cat > bootstrap.ldif << 'EOF'
dn: ou=usuarios,dc=redes2026,dc=local
objectClass: organizationalUnit
ou: usuarios

dn: ou=grupos,dc=redes2026,dc=local
objectClass: organizationalUnit
ou: grupos

dn: uid=jperez,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: jperez
cn: Juan Perez
sn: Perez
givenName: Juan
mail: jperez@redes2026.local
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/jperez
loginShell: /bin/bash
userPassword: HASH_JPEREZ

dn: uid=mrodriguez,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: mrodriguez
cn: Maria Rodriguez
sn: Rodriguez
givenName: Maria
mail: mrodriguez@redes2026.local
uidNumber: 10002
gidNumber: 10000
homeDirectory: /home/mrodriguez
loginShell: /bin/bash
userPassword: HASH_MRODRIGUEZ

dn: uid=csierra,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: csierra
cn: Carlos Sierra
sn: Sierra
givenName: Carlos
mail: csierra@redes2026.local
uidNumber: 10003
gidNumber: 10000
homeDirectory: /home/csierra
loginShell: /bin/bash
userPassword: HASH_CSIERRA

dn: uid=atorres,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: atorres
cn: Ana Torres
sn: Torres
givenName: Ana
mail: atorres@redes2026.local
uidNumber: 10004
gidNumber: 10000
homeDirectory: /home/atorres
loginShell: /bin/bash
userPassword: HASH_ATORRES

dn: uid=lmejia,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: lmejia
cn: Luis Mejia
sn: Mejia
givenName: Luis
mail: lmejia@redes2026.local
uidNumber: 10005
gidNumber: 10000
homeDirectory: /home/lmejia
loginShell: /bin/bash
userPassword: HASH_LMEJIA

dn: cn=usuarios-red,ou=grupos,dc=redes2026,dc=local
objectClass: posixGroup
cn: usuarios-red
gidNumber: 10000
memberUid: jperez
memberUid: mrodriguez
memberUid: csierra
memberUid: atorres
memberUid: lmejia
EOF
```

Los cinco marcadores se reemplazan por los hashes reales, editando el archivo o con `sed -i 's|HASH_JPEREZ|{SSHA}...|'`. Comprobación:

```bash
grep HASH_ bootstrap.ldif   # no debe devolver nada
```

## 3.3 Dockerfiles

El de Bogotá ya existía con el arreglo de `archive.debian.org` y se reconstruyó para tomar el LDIF con los cinco usuarios. Los otros tres son el mismo archivo con IP y gateway distintos.

```bash
cd ~/gns3-servers/ldap

# ---- Bogotá ----
cat > Dockerfile.ldap << 'EOF'
FROM osixia/openldap:1.5.0
RUN echo "deb http://archive.debian.org/debian buster main" > /etc/apt/sources.list && \
    echo "deb http://archive.debian.org/debian-security buster/updates main" >> /etc/apt/sources.list && \
    apt-get -o Acquire::Check-Valid-Until=false update && \
    apt-get install -y --no-install-recommends iproute2 net-tools iputils-ping procps && \
    apt-get clean
COPY bootstrap.ldif /container/service/slapd/assets/config/bootstrap/ldif/custom/50-bootstrap.ldif
ENV LDAP_ORGANISATION="Redes2026"
ENV LDAP_DOMAIN="redes2026.local"
ENV LDAP_ADMIN_PASSWORD="Redes2026"
ENV LDAP_TLS=false
ENTRYPOINT []
CMD ip addr add 10.4.192.12/18 dev eth0; \
    ip link set eth0 up; \
    ip route add default via 10.4.192.1; \
    /container/tool/run --copy-service & \
    exec /bin/bash
EOF

# ---- Derivados ----
sed -e 's|10.4.192.12/18|10.1.128.11/17|' -e 's|via 10.4.192.1|via 10.1.128.1|' Dockerfile.ldap > Dockerfile.ldap-sma
sed -e 's|10.4.192.12/18|10.4.64.12/18|'  -e 's|via 10.4.192.1|via 10.4.64.1|'  Dockerfile.ldap > Dockerfile.ldap-cuc
sed -e 's|10.4.192.12/18|10.5.0.10/18|'   -e 's|via 10.4.192.1|via 10.5.0.1|'   Dockerfile.ldap > Dockerfile.ldap-baq

# ---- Build ----
docker build -t gns3/srv-ldap     -f Dockerfile.ldap     .
docker build -t gns3/srv-ldap-sma -f Dockerfile.ldap-sma .
docker build -t gns3/srv-ldap-cuc -f Dockerfile.ldap-cuc .
docker build -t gns3/srv-ldap-baq -f Dockerfile.ldap-baq .
```

## 3.4 Despliegue

En Bogotá el bootstrap solo se aplica en el primer arranque del contenedor. Para que cargue los usuarios nuevos hay que borrar el nodo existente y crear uno nuevo desde la plantilla. La alternativa sin recrear es agregar los usuarios faltantes en caliente con `ldapadd`.

En SMA, CUC y BAQ se crean plantillas nuevas (1 NIC, Start command vacío), se conectan a un puerto de acceso libre de la VLAN 40 del SW1 de cada sede y se inician.

Verificación en cada nodo:

```bash
ip addr show eth0     # IP de la sede
ip route              # default vía la VIP de la sede
pidof slapd
ldapsearch -x -b "dc=redes2026,dc=local" -D "cn=admin,dc=redes2026,dc=local" -w Redes2026 | grep ^dn:
# esperado: base + 2 OUs + 5 uids + 1 grupo
```

## 3.5 Pruebas de la parte 3

El cliente es el nodo `gns3/ldap-utils-client`, que se mueve de VLAN según la sede a probar o se clona con otras IPs.

| # | Prueba | Comando | Resultado esperado |
|---|---|---|---|
| 1 | Bind de admin | `ldapsearch -x -H ldap://<LDAP sede> -b "dc=redes2026,dc=local" -D "cn=admin,dc=redes2026,dc=local" -w Redes2026` | árbol con los 5 usuarios |
| 2 | Bind de usuario | `ldapwhoami -x -H ldap://<LDAP sede> -D "uid=<user>,ou=usuarios,dc=redes2026,dc=local" -w "<clave>"` | DN devuelto |
| 2b | Bind negativo | igual, con clave errónea | `Invalid credentials (49)` |
| 3 | Vía nombre DNS | `ldapwhoami -x -H ldap://ldap.<sede>.redes2026.local ...` | igual que 2, integra DNS y LDAP |
| 4 | Autonomía ante corte de WAN | apagar el enlace WAN de la sede y repetir 2 con el cliente local | sigue autenticando |
| 5 | Login de sistema | cliente sssd apuntando al LDAP de su sede, `getent passwd <user>` y `su - <user>` | shell como el usuario |

La prueba 4 es la que sustenta el diseño distribuido: con el LDAP centralizado del plan anterior, un corte de WAN dejaba tres sedes sin autenticación. Se documentó al menos en una sede.

# Verificaciones al cierre

- Master DNS de BOG reconstruido, con zonas nuevas (serial 2026071610 en adelante), `allow-transfer`, `notify` y gateway a la VIP.
- Slaves de CUC y BAQ desplegados y con las zonas transferidas (`ls /var/cache/bind/`), con serial idéntico al del master.
- Prueba de propagación con un registro de test.
- Prueba de failover con el master apagado.
- `dhcpd.conf` nuevo con DNS local por sede, nodo DHCP reconstruido y `show ip` en VPCS de CUC y BAQ mostrando su DNS local.
- `bootstrap.ldif` con los cinco usuarios y los hashes reales, sin marcadores.
- Cuatro imágenes LDAP construidas y nodo de BOG recreado.
- Bind de admin, bind de usuario y bind negativo por sede.
- Prueba de corte de WAN con el LDAP local respondiendo.
- `su - <usuario>` vía sssd en al menos una sede.
- Regresión: `dig web.cuc...` y `curl http://web.sma.redes2026.local` siguen respondiendo.
- Excel actualizado con DNS CUC `10.4.64.11`, LDAP CUC `10.4.64.12` y DNS BAQ `10.5.0.12`.

# Datos de referencia

- DNS: master `10.4.192.11` (BOG), slaves `10.4.64.11` (CUC) y `10.5.0.12` (BAQ). SMA usa el master.
- DHCP: `10.4.192.10` (BOG, único). Los helpers en las SVIs no cambian.
- LDAP: BOG `10.4.192.12`, SMA `10.1.128.11`, CUC `10.4.64.12`, BAQ `10.5.0.10`.
- Servidores previos: WEB SMA `10.1.128.10`, IMP SMA `10.1.128.12`, WEB/FTP CUC `10.4.64.10`, SSH BAQ `10.5.0.11`.
- Base DN y admin: `dc=redes2026,dc=local`, `cn=admin,...` con clave `Redes2026`.
- Gateways: VIP HSRP `.1` de la VLAN 40 de cada sede, con `standby use-bia` aplicado en los switches.
- Las zonas se editan únicamente en el master y se sube el serial en cada cambio.
