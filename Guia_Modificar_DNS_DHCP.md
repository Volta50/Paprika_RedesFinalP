# Operación y mantenimiento de DNS y DHCP

Procedimientos usados para modificar los servicios ya montados sin rehacer las imágenes. Todo se ejecuta dentro del contenedor correspondiente. Dominio interno: `redes2026.local`, con DNS y DHCP centrales en Bogotá.

Las rutas que aparecen abajo son las estándar de BIND9 e isc-dhcp-server y coinciden con las de los contenedores del proyecto. Para confirmarlas en una imagen distinta:

```bash
find / -name "*.conf" 2>/dev/null | grep -E 'bind|named|dhcp'
```

## 1. DNS (BIND9)

### 1.1 Archivos

| Archivo | Contenido |
|---|---|
| `/etc/bind/named.conf.local` | Declaración de zonas |
| `/etc/bind/db.redes2026.local` | Zona directa (nombre a IP) |
| `/etc/bind/db.10` | Zona inversa (IP a nombre) |
| `/etc/bind/named.conf.options` | Opciones globales: forwarders, recursión, listen-on |

### 1.2 Agregar un registro

Se edita la zona directa:

```bash
nano /etc/bind/db.redes2026.local
```

Formato de los tipos usados en el proyecto:

```dns
; A: nombre a IPv4
pc-nuevo    IN  A     10.3.192.50

; CNAME: alias hacia otro nombre
intranet    IN  CNAME web.redes2026.local.

; MX: servidor de correo, el número es la prioridad
@           IN  MX 10 mail.redes2026.local.
```

Después de cualquier edición hay que subir el número de serie del SOA. Si no se sube, BIND ignora el cambio y los esclavos no se actualizan:

```dns
@   IN  SOA ns.redes2026.local. admin.redes2026.local. (
        2026072201   ; Serial, formato AAAAMMDDnn
        3600         ; Refresh
        1800         ; Retry
        604800       ; Expire
        86400 )      ; Minimum
```

La convención usada es `AAAAMMDDnn`: si el serial del día iba en `...01`, el siguiente es `...02`.

### 1.3 Aplicar y verificar

```bash
# Validar sintaxis antes de recargar
named-checkconf
named-checkzone redes2026.local /etc/bind/db.redes2026.local

# Recargar sin reiniciar el servicio
rndc reload
# si rndc no está configurado:
service named reload
# o bien:
kill -HUP $(pidof named)

# Probar la resolución
dig @localhost pc-nuevo.redes2026.local +short
nslookup pc-nuevo.redes2026.local localhost
```

`named-checkzone` detecta el serial sin subir y los errores de sintaxis antes de tocar el servicio en marcha.

### 1.4 Zona inversa

Para que `dig -x` responda hay que agregar el PTR correspondiente. En `10.in-addr.arpa` los octetos van invertidos:

```bash
nano /etc/bind/db.10
```

```dns
50.192.3   IN  PTR  pc-nuevo.redes2026.local.   ; para 10.3.192.50
```

También hay que subir el serial de esa zona y recargar:

```bash
dig @localhost -x 10.3.192.50 +short
```

### 1.5 Cambiar un forwarder

En `/etc/bind/named.conf.options`:

```
options {
    forwarders { 8.8.8.8; };
    ...
};
```

Se aplica con `rndc reload`.

### 1.6 Maestro y esclavos

Los esclavos de Cúcuta y Barranquilla no tienen archivos de zona editables: los reciben del maestro por AXFR cuando sube el serial. Para forzar la transferencia y verificarla:

```bash
# en el esclavo
rndc retransfer redes2026.local
tail -f /var/log/syslog | grep named
```

En el log debe aparecer la línea de `transfer of ... completed`.

## 2. DHCP (isc-dhcp-server)

### 2.1 Archivos

| Archivo | Contenido |
|---|---|
| `/etc/dhcp/dhcpd.conf` | Subredes, rangos y opciones |
| `/var/lib/dhcp/dhcpd.leases` | Asignaciones activas, no se edita a mano |

### 2.2 Estructura de una subred

```
subnet 10.3.192.0 netmask 255.255.192.0 {
    range 10.3.192.100 10.3.192.200;
    option routers 10.3.192.1;              # VIP de HSRP
    option domain-name-servers 10.4.192.11;
    option domain-name "redes2026.local";
    default-lease-time 3600;
    max-lease-time 7200;
}
```

### 2.3 Modificaciones habituales

Ampliar o mover el rango:

```
range 10.3.192.50 10.3.192.250;
```

Cambiar el DNS o el gateway entregado: se editan `option domain-name-servers` u `option routers` dentro del bloque de esa subred.

Reserva por MAC, para que un equipo reciba siempre la misma IP:

```
host pc-bog {
    hardware ethernet 0c:aa:bb:cc:dd:ee;
    fixed-address 10.3.192.10;
}
```

La MAC se obtiene con `ip link show eth0` en el equipo destino. La IP reservada debe quedar fuera del `range` para evitar colisiones.

### 2.4 Aplicar y verificar

```bash
# Validar sintaxis antes de reiniciar
dhcpd -t -cf /etc/dhcp/dhcpd.conf

# El demonio no tiene reload, hay que reiniciarlo
service isc-dhcp-server restart
# o directamente:
kill $(pidof dhcpd); dhcpd -cf /etc/dhcp/dhcpd.conf eth0

# Ver asignaciones activas
cat /var/lib/dhcp/dhcpd.leases | grep -E 'lease|hardware|binding'

# Desde un cliente Linux
dhclient -v eth0
```

En el cliente debe verse la secuencia DHCPDISCOVER, DHCPOFFER y DHCPACK.

### 2.5 Relay

El DHCP está solo en Bogotá y reparte a las cuatro sedes porque los switches multicapa tienen el relay configurado en la SVI de cada VLAN de usuarios:

```
interface Vlan20
 ip helper-address 10.4.192.10
```

Cuando una sede o VLAN no recibe direcciones, la causa suele estar en el switch y no en el servidor: falta la línea `ip helper-address` en la SVI (en SW1 y SW2, por el HSRP) o falta el bloque `subnet` correspondiente en `dhcpd.conf`.

## 3. Diagnóstico

| Síntoma | Comando de arranque | Causa habitual |
|---|---|---|
| El cambio de DNS no se refleja | `dig @localhost <nombre>` | Serial sin subir o falta recargar |
| `named` no arranca tras editar | `named-checkzone` / `named-checkconf` | Typo o falta el punto final en un FQDN |
| El PC no recibe IP | `dhclient -v eth0` en el cliente | Falta `ip helper-address` en el switch |
| `dhcpd` no arranca | `dhcpd -t` | Typo en `dhcpd.conf` o subred sin declarar |
| Resuelve directo pero no `dig -x` | Revisar la zona inversa | Falta el PTR o su serial |

Antes de recargar cualquiera de los dos servicios se valida la sintaxis: `named-checkzone` para DNS y `dhcpd -t` para DHCP.

## 4. Punto final en los FQDN de BIND

En los archivos de zona, un nombre terminado en punto es absoluto; uno sin punto recibe el dominio de la zona concatenado:

```dns
web    IN  CNAME  servidor.redes2026.local.   ; correcto
web    IN  CNAME  servidor.redes2026.local    ; queda servidor.redes2026.local.redes2026.local
```

Cuando una consulta devuelve un nombre duplicado, la causa es casi siempre ese punto faltante.
