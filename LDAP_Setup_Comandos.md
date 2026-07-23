# LDAP en MirrorMode entre las cuatro sedes

Las secciones 0 a 5 se ejecutan en el host de GNS3, que es el que tiene salida a Internet real y donde se hacen los `docker build`. Las secciones 6 a 8 se ejecutan dentro de los contenedores o desde el cliente `ldap-utils`.

## 0. Directorio de trabajo

```bash
mkdir -p ~/gns3-servers/ldap-repl && cd ~/gns3-servers/ldap-repl
```

## 1. `bootstrap.ldif`

Solo lo carga la imagen de Bogotá. Las demás sedes reciben los datos por replicación.

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

Son 8 entradas más la base `dc=redes2026,dc=local` que crea la imagen de osixia, lo que da los 9 `dn:` que se esperan en la verificación.

## 2. `00-syncprov.ldif`

Idéntico en las cuatro sedes.

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

## 3. Generación de los cuatro LDIF de replicación

Cada sede necesita su propio `olcServerID` y una línea `olcSyncRepl` por cada uno de los otros tres servidores. Se generaron con un script para evitar errores al escribirlos a mano:

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

## 4. `repl-start.sh`

Espera a que `slapd` esté escuchando y aplica los dos LDIF una sola vez, marcando con un archivo testigo para no repetir la configuración en cada arranque del contenedor. Es igual en las cuatro sedes.

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

## 5. Dockerfiles

### 5.1 Bogotá

Es la única imagen que lleva el bootstrap con los datos.

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

La base de osixia es Debian buster, que ya salió de soporte, por eso se apuntan los repositorios a `archive.debian.org` antes de instalar las herramientas de diagnóstico.

### 5.2 Santa Marta

Sin bootstrap.

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

### 5.3 Cúcuta y Barranquilla

Se derivan del de Santa Marta cambiando LDIF, IP y gateway:

```bash
sed -e 's|repl-sma.ldif|repl-cuc.ldif|' \
    -e 's|10.1.128.11/17|10.4.64.12/18|' \
    -e 's|via 10.1.128.1|via 10.4.64.1|' \
    Dockerfile.sma > Dockerfile.cuc
grep -E 'repl-|ip addr|route' Dockerfile.cuc

sed -e 's|repl-sma.ldif|repl-baq.ldif|' \
    -e 's|10.1.128.11/17|10.5.0.10/16|' \
    -e 's|via 10.1.128.1|via 10.5.0.1|' \
    Dockerfile.sma > Dockerfile.baq
grep -E 'repl-|ip addr|route' Dockerfile.baq
```

Las máscaras salen de los `netmask` usados en los Dockerfile de WEB, FTP e impresión, y se contrastaron contra la tabla de VLSM antes de construir.

## 6. Build

```bash
cd ~/gns3-servers/ldap-repl
docker build -t gns3/srv-ldap     -f Dockerfile.bog .
docker build -t gns3/srv-ldap-sma -f Dockerfile.sma .
docker build -t gns3/srv-ldap-cuc -f Dockerfile.cuc .
docker build -t gns3/srv-ldap-baq -f Dockerfile.baq .
docker images | grep srv-ldap
```

## 7. Despliegue en GNS3

1. Se eliminan los cuatro nodos LDAP anteriores (borrar, no solo apagar). Eso destruye el `/var/lib/ldap` con los `entryUUID` viejos, que de otro modo entran en conflicto con la replicación.
2. Se editan las cuatro plantillas para que apunten a las imágenes recién construidas: 1 NIC, Start command vacío, consola telnet.
3. Se arrastran los cuatro nodos y se cablean a los puertos de la VLAN 40 de cada sede.
4. Se enciende primero Bogotá. Después de unos 20 segundos, las otras tres.

En la consola de cada nodo debe aparecer el mensaje del script:

```
[repl] configuracion aplicada
```

## 8. Verificación

### 8.1 ServerID

Dentro de cada contenedor:

```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -s base olcServerID -LLL
```

Debe dar 1 en BOG, 2 en SMA, 3 en CUC y 4 en BAQ. Un 0 o un valor repetido hace que la replicación entre en bucle.

### 8.2 Overlay syncprov cargado

```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -LLL "(olcOverlay=syncprov)" dn
```

### 8.3 Conteo de entradas

Desde el cliente `ldap-utils`:

```bash
for ip in 10.4.192.12 10.1.128.11 10.4.64.12 10.5.0.10; do
  printf "%-14s -> " "$ip"
  ldapsearch -x -H ldap://$ip -b "dc=redes2026,dc=local" \
    -D "cn=admin,dc=redes2026,dc=local" -w Redes2026 2>/dev/null | grep -c '^dn:'
done
```

Los cuatro devuelven 9.

### 8.4 contextCSN

```bash
for ip in 10.4.192.12 10.1.128.11 10.4.64.12 10.5.0.10; do
  echo "=== $ip ==="
  ldapsearch -x -H ldap://$ip -b "dc=redes2026,dc=local" -s base contextCSN -LLL
done
```

Cada servidor lista 4 valores (`#001#`, `#002#`, `#003#` y `#004#`) y el conjunto es idéntico en los cuatro.

### 8.5 Propagación en vivo

Se crea una entrada en Cúcuta y se consulta en las otras tres sedes.

En la consola del nodo LDAP-CUC:

```bash
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

Desde el cliente, contra las otras tres:

```bash
for ip in 10.4.192.12 10.1.128.11 10.5.0.10; do
  printf "%-14s -> " "$ip"
  ldapsearch -x -H ldap://$ip -b "uid=pgomez,ou=usuarios,dc=redes2026,dc=local" -LLL cn 2>&1 | grep -E '^cn:|No such'
done
```

### 8.6 Autenticación cruzada

Los cinco usuarios contra los cuatro servidores:

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

Las 20 líneas devuelven `dn:uid=<usuario>,ou=usuarios,dc=redes2026,dc=local`.

## 9. Depuración de la replicación

Para subir el nivel de log:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<'EOF'
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: sync stats
EOF

cat /var/log/repl.log
tail -f /var/log/syslog 2>/dev/null || docker logs <nombre-contenedor>
```

Para volver a dejarlo en silencio:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<'EOF'
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: none
EOF
```

## 10. Exportación de las imágenes

```bash
docker save -o imagenes-docker.tar \
  gns3/srv-dhcp gns3/srv-dns gns3/srv-dns-cuc gns3/srv-dns-baq \
  gns3/srv-ldap gns3/srv-ldap-sma gns3/srv-ldap-cuc gns3/srv-ldap-baq \
  gns3/srv-web-sma gns3/srv-web-ftp-cuc gns3/srv-print-sma gns3/ldap-utils-client
ls -lh imagenes-docker.tar
```
