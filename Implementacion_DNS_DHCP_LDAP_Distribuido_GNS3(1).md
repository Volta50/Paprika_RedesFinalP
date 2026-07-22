# DNS Distribuido + DHCP + LDAP Distribuido (metodología Dockerfile)

**Flujo de trabajo:** el mismo ya probado con DHCP, DNS y LDAP de Bogotá — un Dockerfile por servicio, IP/ruta/demonio en el `CMD` (arranque automático), `exec /bin/bash` al final para consola usable. Los nodos WEB/FTP/Impresión del compañero (imagen `base-server:v1`, config manual) **conviven sin cambios** — solo los referencia la zona DNS.

**Topología de servidores (según diagrama del proyecto):**

| Ciudad | DNS | LDAP | DHCP | Ya montado (compañero) |
|---|---|---|---|---|
| Bogotá | ✔ master `10.4.192.11` | ✔ `10.4.192.12` | ✔ `10.4.192.10` (único) | — |
| Santa Marta | — | ✔ `10.1.128.11` | — | WEB `10.1.128.10`, IMP `10.1.128.12` |
| Cúcuta | ✔ slave `10.4.64.11` | ✔ `10.4.64.12` | — | WEB/FTP `10.4.64.10` |
| Barranquilla | ✔ slave `10.5.0.12` | ✔ `10.5.0.10` | — | SSH `10.5.0.11` |

> **Cambios de IP vs. plan viejo (actualizar el Excel):**
> - LDAP CUC: `10.4.64.11` → **`10.4.64.12`** (la `.11` la toma el DNS slave de CUC)
> - DNS BAQ: nuevo en **`10.5.0.12`**
> - Los servidores del compañero coinciden con la zona DNS existente — sin cambios.

**Gateways:** VIP HSRP `.1` de la VLAN 40 de cada sede (criterio actualizado tras el fix `standby use-bia` — verificar que esté aplicado en los switches de CUC, BAQ y SMA antes de desplegar ahí).

**Estructura de directorios en el host:**

```
~/gns3-servers/
├── dhcpd.conf, Dockerfile.dhcp        (existentes — se edita dhcpd.conf)
├── dns/                                (existente — master BOG, se actualizan zonas)
├── dns-cuc/                            (nuevo — slave)
├── dns-baq/                            (nuevo — slave)
└── ldap/                               (existente BOG + 3 Dockerfiles nuevos)
```

---

# PARTE 1 — DNS distribuido (master BOG + slaves CUC/BAQ)

**Diseño:** master–slave con transferencia de zona (AXFR) y notificaciones. Justificación para la entrega: una sola fuente de verdad (las zonas se editan SOLO en Bogotá y se propagan solas al subir el Serial), alta disponibilidad (si cae BOG, los slaves siguen resolviendo con su copia local), y menor latencia (cada sede consulta su DNS local). SMA no tiene DNS en el diagrama — sus clientes usan el master de BOG.

## 1.1 Actualizar el MASTER (Bogotá, `10.4.192.11`)

```bash
cd ~/gns3-servers/dns
```

### `named.conf.options` — solo cambia la ruta default en el Dockerfile, options queda igual salvo verificación:

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

### `named.conf.local` — declara slaves y activa notify:

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

### Zona directa — registros nuevos (DNS slaves, LDAP por sede, impresión SMA):

```bash
cat > db.redes2026.local << 'EOF'
$TTL 604800
@   IN  SOA dns.redes2026.local. admin.redes2026.local. (
        2026071610 ; Serial — SUBIR en cada cambio
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

### Zona inversa:

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

### `Dockerfile.dns` — actualizar gateway a la VIP (decisión ya tomada):

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

### Build y redespliegue:

```bash
docker build -t gns3/srv-dns -f Dockerfile.dns .
```

En GNS3: stop/start del nodo Srv-DNS (o recrear). Verificar:

```bash
named-checkconf
dig @127.0.0.1 dns.cuc.redes2026.local +short   # → 10.4.64.11
dig @127.0.0.1 -x 10.4.64.12 +short             # → ldap.cuc.redes2026.local.
```

## 1.2 SLAVE Cúcuta (`10.4.64.11`)

Los slaves **no llevan archivos de zona** — los reciben del master por AXFR y los guardan en `/var/cache/bind/` (writable).

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

## 1.3 SLAVE Barranquilla (`10.5.0.12`)

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

1. Plantillas GNS3: `gns3/srv-dns-cuc` y `gns3/srv-dns-baq` (1 NIC, Start command vacío).
2. Conectar a puerto de acceso **VLAN 40** libre de SW1-CUC y SW1-BAQ (`show vlan brief` para confirmar).
3. Iniciar. En cada slave, segundos después:

```bash
ls -l /var/cache/bind/                        # deben existir db.redes2026.local y db.10
dig @127.0.0.1 dhcp.redes2026.local +short    # → 10.4.192.10 (solo pudo llegar por AXFR)
dig @127.0.0.1 redes2026.local SOA +short     # Serial idéntico al del master
```

Si los archivos no aparecen: `ping 10.4.192.11` (¿llega al master?), y en el master revisar que `allow-transfer` tenga la IP exacta del slave.

## 1.5 Pruebas de la Parte 1 (documentar con capturas)

| Prueba | Cómo | Esperado |
|---|---|---|
| Propagación | Agregar `test IN A 10.4.192.99` en el master, subir Serial, rebuild+restart (o `rndc reload`) | `dig test.redes2026.local` responde en ambos slaves en <1 min |
| Failover | Apagar el nodo master; consultar los slaves | Siguen resolviendo toda la zona interna |
| Regresión web | `dig @10.4.64.11 web.cuc.redes2026.local +short` | 10.4.64.10 (nodo del compañero) |

---

# PARTE 2 — DHCP (solo cambia `dhcpd.conf`: DNS local por sede)

La arquitectura de relay no cambia (helpers en las SVIs ya configurados). Solo se actualiza qué DNS recibe cada sede: **su servidor local como primario, el master de BOG como respaldo**.

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

# ================= SANTA MARTA (sin DNS propio -> master BOG) =================
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

### `Dockerfile.dhcp` — actualizar gateway a la VIP (si no se hizo ya):

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

Stop/start del nodo Srv-DHCP. Prueba: en un VPCS de CUC o BAQ, `ip dhcp` → `show ip` debe mostrar el **DNS local de esa sede** en el campo DNS.

---

# PARTE 3 — LDAP distribuido (un servidor por ciudad)

**Diseño:** 4 servidores autónomos con árbol idéntico (`dc=redes2026,dc=local`), cargado por bootstrap. Justificación: la actividad 8 (obligatoria) pide autenticación, no replicación (eso es el punto 6, opcional); servidores independientes sobreviven a cortes de WAN — cada sede autentica localmente. Trade-off: usuarios creados a posteriori en una sede no se propagan (irrelevante con usuarios de prueba fijos). Si se aborda el punto opcional 6, se puede montar syncrepl entre dos sedes encima de esto.

## 3.1 Usuarios (2 originales + 1 "asignado" por sede — los 5 en todas)

| uid | Nombre | Sede de prueba | Clave |
|---|---|---|---|
| jperez | Juan Perez | BOG | ClaveDeJuan123 |
| mrodriguez | Maria Rodriguez | BOG | ClaveDeMaria456 |
| csierra | Carlos Sierra | SMA | ClaveDeCarlos789 |
| atorres | Ana Torres | CUC | ClaveDeAna012 |
| lmejia | Luis Mejia | BAQ | ClaveDeLuis345 |

Generar los 5 hashes con el LDAP de Bogotá que ya corre (anotarlos):

```bash
for c in "ClaveDeJuan123" "ClaveDeMaria456" "ClaveDeCarlos789" "ClaveDeAna012" "ClaveDeLuis345"; do
  docker exec -i $(docker ps -qf "ancestor=gns3/srv-ldap") slappasswd -s "$c"
done
```

## 3.2 `bootstrap.ldif` unificado (el mismo archivo para las 4 imágenes)

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

**Reemplazar los 5 placeholders** por los hashes reales (editar el archivo, o `sed -i 's|HASH_JPEREZ|{SSHA}...|'` etc.). Verificar:

```bash
grep HASH_ bootstrap.ldif   # NO debe devolver nada
```

## 3.3 Los 4 Dockerfiles

El de BOG ya existe (con el fix de archive.debian.org) — se reconstruye para tomar el LDIF de 5 usuarios. Los otros 3 son el mismo con IP/gateway distintos:

```bash
cd ~/gns3-servers/ldap

# ---- Plantilla común (Bogotá — ya existente, se muestra completa como referencia) ----
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

# ---- Derivar SMA / CUC / BAQ ----
sed -e 's|10.4.192.12/18|10.1.128.11/17|' -e 's|via 10.4.192.1|via 10.1.128.1|' Dockerfile.ldap > Dockerfile.ldap-sma
sed -e 's|10.4.192.12/18|10.4.64.12/18|'  -e 's|via 10.4.192.1|via 10.4.64.1|'  Dockerfile.ldap > Dockerfile.ldap-cuc
sed -e 's|10.4.192.12/18|10.5.0.10/18|'   -e 's|via 10.4.192.1|via 10.5.0.1|'   Dockerfile.ldap > Dockerfile.ldap-baq

# ---- Build de las 4 ----
docker build -t gns3/srv-ldap     -f Dockerfile.ldap     .
docker build -t gns3/srv-ldap-sma -f Dockerfile.ldap-sma .
docker build -t gns3/srv-ldap-cuc -f Dockerfile.ldap-cuc .
docker build -t gns3/srv-ldap-baq -f Dockerfile.ldap-baq .
```

> Nota: el gateway de BOG en esta versión ya apunta a la VIP `10.4.192.1` (decisión tomada tras el fix `use-bia`).

## 3.4 Despliegue

1. **Bogotá:** el bootstrap solo se aplica en el **primer arranque** — para que cargue los 3 usuarios nuevos, **borrar el nodo existente y crear uno nuevo** desde la plantilla (mismo procedimiento que con el fix de hashes). Alternativa sin recrear: agregar solo los 3 usuarios nuevos con `ldapadd` en caliente.
2. **SMA/CUC/BAQ:** plantillas GNS3 nuevas (1 NIC, Start command vacío), conectar a puerto de acceso VLAN 40 libre del SW1 de cada sede, iniciar.

Verificación en cada nodo:

```bash
ip addr show eth0     # IP de la sede
ip route              # default vía la VIP de la sede
pidof slapd
ldapsearch -x -b "dc=redes2026,dc=local" -D "cn=admin,dc=redes2026,dc=local" -w Redes2026 | grep ^dn:
# esperado: base + 2 OUs + 5 uids + 1 grupo
```

## 3.5 Pruebas de la Parte 3 (por sede, con capturas)

Cliente: el nodo `gns3/ldap-utils-client` ya construido (moverlo de VLAN según la sede a probar, o clonar la imagen con otras IPs).

| # | Prueba | Comando | Esperado |
|---|---|---|---|
| 1 | Bind admin local | `ldapsearch -x -H ldap://<LDAP sede> -b "dc=redes2026,dc=local" -D "cn=admin,dc=redes2026,dc=local" -w Redes2026` | árbol con 5 usuarios |
| 2 | Bind usuario de la sede | `ldapwhoami -x -H ldap://<LDAP sede> -D "uid=<user>,ou=usuarios,dc=redes2026,dc=local" -w "<clave>"` | DN devuelto |
| 2b | Bind negativo | mismo, clave errónea | `Invalid credentials (49)` |
| 3 | Vía nombre DNS | `ldapwhoami -x -H ldap://ldap.<sede>.redes2026.local ...` | igual que 2 — integra DNS+LDAP |
| 4 | Autonomía ante corte WAN | apagar el enlace WAN de la sede; repetir 2 con cliente local | **sigue autenticando** |
| 5 | Login de sistema (actividad 8) | cliente sssd apuntando al LDAP de su sede; `getent passwd <user>`; `su - <user>` | shell como el usuario |

La prueba 4 es el argumento estrella del diseño distribuido: con el LDAP centralizado del plan viejo, un corte de WAN dejaba 3 sedes sin autenticación. Documentarla en al menos una sede.

---

# Checklist consolidado

- [ ] Master DNS BOG reconstruido: zonas nuevas (Serial 2026071610+), `allow-transfer`/`notify`, gateway VIP.
- [ ] Slaves CUC/BAQ desplegados; zonas transferidas (`ls /var/cache/bind/`), Serial idéntico en los 3.
- [ ] Prueba de propagación (registro test) capturada.
- [ ] Prueba de failover DNS (master apagado) capturada.
- [ ] `dhcpd.conf` nuevo: DNS local por sede; nodo DHCP reconstruido; `show ip` en VPCS de CUC/BAQ muestra su DNS local.
- [ ] `bootstrap.ldif` con 5 usuarios y hashes reales (sin placeholders).
- [ ] 4 imágenes LDAP construidas; nodo BOG **recreado** (o 3 usuarios agregados en caliente).
- [ ] Bind admin + bind usuario + bind negativo por sede, capturados.
- [ ] Prueba de corte WAN (LDAP local sobrevive) capturada.
- [ ] `su - <usuario>` vía sssd en al menos una sede (actividad 8) capturado.
- [ ] Regresión: `dig web.cuc...` y `curl http://web.sma.redes2026.local` siguen funcionando (nodos del compañero intactos).
- [ ] Excel actualizado: DNS CUC `10.4.64.11` · LDAP CUC `10.4.64.12` · DNS BAQ `10.5.0.12`.

# Datos de referencia

- **DNS:** master `10.4.192.11` (BOG) · slaves `10.4.64.11` (CUC), `10.5.0.12` (BAQ) · SMA usa BOG
- **DHCP:** `10.4.192.10` (BOG, único; helpers en SVIs sin cambios)
- **LDAP:** BOG `10.4.192.12` · SMA `10.1.128.11` · CUC `10.4.64.12` · BAQ `10.5.0.10`
- **Compañero (sin cambios):** WEB SMA `10.1.128.10` · IMP SMA `10.1.128.12` · WEB/FTP CUC `10.4.64.10` · SSH BAQ `10.5.0.11`
- **Base DN / admin:** `dc=redes2026,dc=local` · `cn=admin,...` / `Redes2026`
- **Gateways:** VIP HSRP `.1` de VLAN 40 por sede (requiere `standby use-bia` en los switches)
- **Regla de oro DNS:** editar zonas SOLO en el master + subir el Serial
