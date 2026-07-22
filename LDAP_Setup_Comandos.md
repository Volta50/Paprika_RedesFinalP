# LDAP MirrorMode — Comandos ready to paste

Todo lo de las secciones 0–5 se ejecuta **en el host de GNS3** (el que tiene internet real, donde haces los `docker build`).
Las secciones 6–8 se ejecutan **dentro de los contenedores** o desde el cliente `ldap-utils`.

---

## 0) Crear el directorio de trabajo

```bash
mkdir -p ~/gns3-servers/ldap-repl && cd ~/gns3-servers/ldap-repl
```

---

## 1) `bootstrap.ldif` (solo lo usa Bogotá)

```bash
cat > bootstrap.ldif <<'EOF'
dn: ou=usuarios,dc=redes2026,dc=local
objectClass: organizationalUnit
ou: usuarios

dn: ou=grupos,dc=redes2026,dc=local
objectClass: organizationalUnit
ou: grupos

dn: cn=usuarios-red,ou=grupos,dc=redes2026,dc=local
objectClass: posixGroup
cn: usuarios-red
gidNumber: 10000

dn: uid=jperez,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: jperez
cn: Juan Perez
sn: Perez
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/jperez
loginShell: /bin/bash
userPassword: ClaveDeJuan123

dn: uid=mrodriguez,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: mrodriguez
cn: Maria Rodriguez
sn: Rodriguez
uidNumber: 10002
gidNumber: 10000
homeDirectory: /home/mrodriguez
loginShell: /bin/bash
userPassword: ClaveDeMaria456

dn: uid=csierra,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: csierra
cn: Carlos Sierra
sn: Sierra
uidNumber: 10003
gidNumber: 10000
homeDirectory: /home/csierra
loginShell: /bin/bash
userPassword: ClaveDeCarlos789

dn: uid=atorres,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: atorres
cn: Ana Torres
sn: Torres
uidNumber: 10004
gidNumber: 10000
homeDirectory: /home/atorres
loginShell: /bin/bash
userPassword: ClaveDeAna012

dn: uid=lmejia,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: lmejia
cn: Luis Mejia
sn: Mejia
uidNumber: 10005
gidNumber: 10000
homeDirectory: /home/lmejia
loginShell: /bin/bash
userPassword: ClaveDeLuis345
EOF
```

> Son 8 entradas + la base `dc=redes2026,dc=local` que crea osixia sola = los **9 `dn:`** esperados.
> Si ya tienes tu `bootstrap.ldif` funcionando con los 5 usuarios, cópialo a esta carpeta y salta este paso.

---

## 2) `00-syncprov.ldif` (idéntico en las 4 sedes)

```bash
cat > 00-syncprov.ldif <<'EOF'
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: syncprov

dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpCheckpoint: 100 10
olcSpSessionlog: 100
EOF
```

---

## 3) Generar los 4 LDIF de replicación

```bash
cat > gen-repl.sh <<'EOF'
#!/bin/bash
declare -A IP=( [bog]=10.4.192.12 [sma]=10.1.128.11 [cuc]=10.4.64.12 [baq]=10.5.0.10 )
declare -A ID=( [bog]=1 [sma]=2 [cuc]=3 [baq]=4 )
for me in bog sma cuc baq; do
  f="repl-${me}.ldif"
  cat > "$f" <<INNER
dn: cn=config
changetype: modify
replace: olcServerID
olcServerID: ${ID[$me]}

dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSyncRepl
INNER
  for peer in bog sma cuc baq; do
    [ "$peer" = "$me" ] && continue
    printf 'olcSyncRepl: rid=00%s provider=ldap://%s:389 binddn="cn=admin,dc=redes2026,dc=local" bindmethod=simple credentials=Redes2026 searchbase="dc=redes2026,dc=local" type=refreshAndPersist retry="5 5 60 +" timeout=3 starttls=no\n' "${ID[$peer]}" "${IP[$peer]}" >> "$f"
  done
  printf -- '-\nadd: olcMirrorMode\nolcMirrorMode: TRUE\n' >> "$f"
  echo "OK -> $f"
done
EOF
chmod +x gen-repl.sh && ./gen-repl.sh && cat repl-cuc.ldif
```

---

## 4) `repl-start.sh` (idéntico en las 4)

```bash
cat > repl-start.sh <<'EOF'
#!/bin/bash
for i in $(seq 1 30); do
  ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -s base dn >/dev/null 2>&1 && break
  sleep 2
done
if [ ! -f /var/lib/ldap/.repl-ok ]; then
  ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/00-syncprov.ldif -c 2>&1 | tee    /var/log/repl.log
  ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/repl-local.ldif  -c 2>&1 | tee -a /var/log/repl.log
  touch /var/lib/ldap/.repl-ok
  echo "[repl] configuracion aplicada" 
else
  echo "[repl] ya estaba configurado"
fi
EOF
chmod +x repl-start.sh
```

---

## 5) Los 4 Dockerfile

### 5.1 Bogotá — `Dockerfile.bog` (**el único con datos**)

```bash
cat > Dockerfile.bog <<'EOF'
FROM osixia/openldap:1.5.0
RUN echo "deb http://archive.debian.org/debian buster main" > /etc/apt/sources.list && \
    echo "deb http://archive.debian.org/debian-security buster/updates main" >> /etc/apt/sources.list && \
    apt-get -o Acquire::Check-Valid-Until=false update && \
    apt-get install -y --no-install-recommends iproute2 net-tools iputils-ping procps ldap-utils netcat && \
    apt-get clean
COPY bootstrap.ldif    /container/service/slapd/assets/config/bootstrap/ldif/custom/50-bootstrap.ldif
COPY 00-syncprov.ldif  /root/00-syncprov.ldif
COPY repl-bog.ldif     /root/repl-local.ldif
COPY repl-start.sh     /root/repl-start.sh
RUN chmod +x /root/repl-start.sh
ENV LDAP_ORGANISATION="Redes2026"
ENV LDAP_DOMAIN="redes2026.local"
ENV LDAP_ADMIN_PASSWORD="Redes2026"
ENV LDAP_TLS=false
ENTRYPOINT []
CMD ip addr add 10.4.192.12/18 dev eth0; ip link set eth0 up; ip route add default via 10.4.192.1; \
    /container/tool/run --copy-service & \
    sleep 5; /root/repl-start.sh; exec /bin/bash
EOF
```

### 5.2 Santa Marta — `Dockerfile.sma` (sin bootstrap)

```bash
cat > Dockerfile.sma <<'EOF'
FROM osixia/openldap:1.5.0
RUN echo "deb http://archive.debian.org/debian buster main" > /etc/apt/sources.list && \
    echo "deb http://archive.debian.org/debian-security buster/updates main" >> /etc/apt/sources.list && \
    apt-get -o Acquire::Check-Valid-Until=false update && \
    apt-get install -y --no-install-recommends iproute2 net-tools iputils-ping procps ldap-utils netcat && \
    apt-get clean
COPY 00-syncprov.ldif  /root/00-syncprov.ldif
COPY repl-sma.ldif     /root/repl-local.ldif
COPY repl-start.sh     /root/repl-start.sh
RUN chmod +x /root/repl-start.sh
ENV LDAP_ORGANISATION="Redes2026"
ENV LDAP_DOMAIN="redes2026.local"
ENV LDAP_ADMIN_PASSWORD="Redes2026"
ENV LDAP_TLS=false
ENTRYPOINT []
CMD ip addr add 10.1.128.11/17 dev eth0; ip link set eth0 up; ip route add default via 10.1.128.1; \
    /container/tool/run --copy-service & \
    sleep 5; /root/repl-start.sh; exec /bin/bash
EOF
```

### 5.3 Cúcuta — `Dockerfile.cuc`

```bash
sed -e 's|repl-sma.ldif|repl-cuc.ldif|' \
    -e 's|10.1.128.11/17|10.4.64.12/18|' \
    -e 's|via 10.1.128.1|via 10.4.64.1|' \
    Dockerfile.sma > Dockerfile.cuc
grep -E 'repl-|ip addr|route' Dockerfile.cuc
```

### 5.4 Barranquilla — `Dockerfile.baq`

```bash
sed -e 's|repl-sma.ldif|repl-baq.ldif|' \
    -e 's|10.1.128.11/17|10.5.0.10/16|' \
    -e 's|via 10.1.128.1|via 10.5.0.1|' \
    Dockerfile.sma > Dockerfile.baq
grep -E 'repl-|ip addr|route' Dockerfile.baq
```

> **Verifica las máscaras contra tu Excel de VLSM antes de construir.** Las de arriba salen de los `netmask` que ya usas en los Dockerfile de WEB/FTP/Impresión.

---

## 6) Build

```bash
cd ~/gns3-servers/ldap-repl
docker build -t gns3/srv-ldap     -f Dockerfile.bog .
docker build -t gns3/srv-ldap-sma -f Dockerfile.sma .
docker build -t gns3/srv-ldap-cuc -f Dockerfile.cuc .
docker build -t gns3/srv-ldap-baq -f Dockerfile.baq .
docker images | grep srv-ldap
```

---

## 7) En GNS3

1. **Borrar** (no solo apagar) los 4 nodos LDAP existentes → clic derecho › Delete. Esto destruye el `/var/lib/ldap` con los `entryUUID` viejos.
2. Editar las 4 plantillas para que apunten a las imágenes recién construidas: 1 NIC, **Start command vacío**, consola `telnet`.
3. Arrastrar los 4 nodos, cablearlos a los puertos de la VLAN 40 de cada sede.
4. **Encender primero BOG.** Esperar ~20 s. Luego SMA, CUC y BAQ.

Comprobar en la consola de cada nodo que salió el mensaje del script:

```
[repl] configuracion aplicada
```

---

## 8) Verificación

### 8.1 ServerID correcto en cada nodo (dentro del contenedor)

```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -s base olcServerID -LLL
```

Debe dar 1 en BOG, 2 en SMA, 3 en CUC, 4 en BAQ. **Si alguno da 0 o repetido, la replicación hará bucle.**

### 8.2 Que el overlay syncprov cargó

```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -LLL "(olcOverlay=syncprov)" dn
```

### 8.3 Conteo de entradas en las 4 sedes (desde el cliente `ldap-utils`)

```bash
for ip in 10.4.192.12 10.1.128.11 10.4.64.12 10.5.0.10; do
  printf "%-14s -> " "$ip"
  ldapsearch -x -H ldap://$ip -b "dc=redes2026,dc=local" \
    -D "cn=admin,dc=redes2026,dc=local" -w Redes2026 2>/dev/null | grep -c '^dn:'
done
```

Los cuatro deben decir **9**.

### 8.4 `contextCSN` — captura para el informe

```bash
for ip in 10.4.192.12 10.1.128.11 10.4.64.12 10.5.0.10; do
  echo "=== $ip ==="
  ldapsearch -x -H ldap://$ip -b "dc=redes2026,dc=local" -s base contextCSN -LLL
done
```

Cada servidor debe listar **4 valores** (`#001#`, `#002#`, `#003#`, `#004#`) y el conjunto debe ser idéntico entre los 4.

### 8.5 Propagación en vivo — crear en CUC, leer en las otras 3

```bash
# --- en la consola del nodo LDAP-CUC ---
cat > /tmp/nuevo.ldif <<'EOF'
dn: uid=pgomez,ou=usuarios,dc=redes2026,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: pgomez
cn: Pedro Gomez
sn: Gomez
uidNumber: 10006
gidNumber: 10000
homeDirectory: /home/pgomez
loginShell: /bin/bash
userPassword: ClaveDePedro678
EOF
ldapadd -x -H ldap://localhost -D "cn=admin,dc=redes2026,dc=local" -w Redes2026 -f /tmp/nuevo.ldif
```

```bash
# --- desde el cliente, contra las otras tres ---
for ip in 10.4.192.12 10.1.128.11 10.5.0.10; do
  printf "%-14s -> " "$ip"
  ldapsearch -x -H ldap://$ip -b "uid=pgomez,ou=usuarios,dc=redes2026,dc=local" -LLL cn 2>&1 | grep -E '^cn:|No such'
done
```

### 8.6 Autenticación cruzada — la matriz de la entrega

```bash
declare -A P=( [jperez]=ClaveDeJuan123 [mrodriguez]=ClaveDeMaria456 \
               [csierra]=ClaveDeCarlos789 [atorres]=ClaveDeAna012 [lmejia]=ClaveDeLuis345 )
for u in "${!P[@]}"; do
  for ip in 10.4.192.12 10.1.128.11 10.4.64.12 10.5.0.10; do
    printf "%-12s @ %-14s : " "$u" "$ip"
    ldapwhoami -x -H ldap://$ip -D "uid=$u,ou=usuarios,dc=redes2026,dc=local" -w "${P[$u]}" 2>&1
  done
done
```

Las 20 líneas deben devolver `dn:uid=<user>,ou=usuarios,dc=redes2026,dc=local`.

---

## 9) Si algo falla — subir el log de sync

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<'EOF'
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: sync stats
EOF

# ver qué está pasando
cat /var/log/repl.log
tail -f /var/log/syslog 2>/dev/null || docker logs <nombre-contenedor>
```

Volver a silencio cuando termines:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<'EOF'
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: none
EOF
```

---

## 10) Al cerrar — exportar imágenes

```bash
docker save -o imagenes-docker.tar \
  gns3/srv-dhcp gns3/srv-dns gns3/srv-dns-cuc gns3/srv-dns-baq \
  gns3/srv-ldap gns3/srv-ldap-sma gns3/srv-ldap-cuc gns3/srv-ldap-baq \
  gns3/srv-web-sma gns3/srv-web-ftp-cuc gns3/srv-print-sma gns3/ldap-utils-client
ls -lh imagenes-docker.tar
```
