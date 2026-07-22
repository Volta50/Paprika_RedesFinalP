# Guía rápida — Modificar DNS y DHCP en caliente

Referencia para la sustentación. Todo se ejecuta **dentro del contenedor** correspondiente.
Dominio: `redes2026.local` · DNS y DHCP centrales en Bogotá.

> Aviso: los nombres de archivo y rutas de abajo son los estándar de BIND9 e isc-dhcp-server.
> Antes de la sustentación, confirma las rutas reales de TUS contenedores con:
> `find / -name "*.conf" 2>/dev/null | grep -E 'bind|named|dhcp'`
> y ajusta la guía si difieren.

---

## PARTE 1 — DNS (BIND9)

### 1.1 Dónde vive cada cosa

| Archivo | Para qué |
|---|---|
| `/etc/bind/named.conf.local` | Declara las zonas (qué dominios maneja este servidor) |
| `/etc/bind/db.redes2026.local` | Zona directa: nombre → IP |
| `/etc/bind/db.rev...` | Zona inversa: IP → nombre |
| `/etc/bind/named.conf.options` | Opciones globales (forwarders, recursión) |

### 1.2 Añadir un registro (lo más pedido)

Editar la zona directa:

```bash
nano /etc/bind/db.redes2026.local
```

Agregar al final una línea. Ejemplos de los tipos más comunes:

```dns
; A -- nombre a IPv4
pc-nuevo    IN  A     10.3.192.50

; CNAME -- alias que apunta a otro nombre
intranet    IN  CNAME web.redes2026.local.

; MX -- servidor de correo (el numero es la prioridad)
@           IN  MX 10 mail.redes2026.local.
```

**El paso que más se olvida:** subir el número de serie de la zona, o BIND ignora el cambio y los esclavos no se actualizan. Está en el bloque SOA, arriba del archivo:

```dns
@   IN  SOA ns.redes2026.local. admin.redes2026.local. (
        2026072201   ; Serial  <-- SUBIR ESTE (formato AAAAMMDDnn)
        3600         ; Refresh
        1800         ; Retry
        604800       ; Expire
        86400 )      ; Minimum
```

Convención: `AAAAMMDDnn`. Si hoy ya ibas en `...01`, el siguiente es `...02`.

### 1.3 Aplicar y verificar

```bash
# 1. Validar la sintaxis ANTES de recargar
named-checkconf
named-checkzone redes2026.local /etc/bind/db.redes2026.local

# 2. Recargar sin reiniciar
rndc reload
# si rndc no está configurado:  service named reload   (o: kill -HUP $(pidof named))

# 3. Probar la resolución
dig @localhost pc-nuevo.redes2026.local +short
nslookup pc-nuevo.redes2026.local localhost
```

`named-checkzone` es tu red de seguridad: si el serial no subió o hay un typo, te avisa aquí en vez de tumbar el servicio.

### 1.4 Zona inversa (IP → nombre)

Si te piden que `dig -x` también funcione, hay que añadir el PTR en la zona inversa. La IP va **al revés** y solo el último octeto:

```bash
nano /etc/bind/db.rev.10.3.192   # el nombre real depende de tu named.conf.local
```

```dns
50   IN  PTR  pc-nuevo.redes2026.local.   ; para la IP 10.3.192.50
```

Subir también el serial de ESA zona y recargar. Verificar:

```bash
dig @localhost -x 10.3.192.50 +short   # → pc-nuevo.redes2026.local.
```

### 1.5 Cambiar un forwarder (a dónde manda lo que no conoce)

En `/etc/bind/named.conf.options`:

```
options {
    forwarders { 8.8.8.8; };   ; DNS externo al que reenvía
    ...
};
```

Recargar con `rndc reload`.

### 1.6 DNS maestro / esclavo

Si tienes DNS esclavo en otra sede (Cúcuta/Barranquilla), NO se editan sus archivos de zona: se copian solos del maestro cuando subes el serial. Forzar transferencia y verla:

```bash
# en el esclavo
rndc retransfer redes2026.local
tail -f /var/log/syslog | grep named   # buscar "transfer of ... completed"
```

---

## PARTE 2 — DHCP (isc-dhcp-server)

### 2.1 Dónde vive

| Archivo | Para qué |
|---|---|
| `/etc/dhcp/dhcpd.conf` | Toda la configuración: subredes, rangos, opciones |
| `/var/lib/dhcp/dhcpd.leases` | Asignaciones activas (no editar a mano) |

### 2.2 Anatomía de una subred

Cada VLAN que reparte IP tiene un bloque así:

```
subnet 10.3.192.0 netmask 255.255.192.0 {
    range 10.3.192.100 10.3.192.200;        # rango que reparte
    option routers 10.3.192.1;              # gateway (VIP de HSRP)
    option domain-name-servers 10.4.192.12; # servidor DNS
    option domain-name "redes2026.local";
    default-lease-time 3600;
    max-lease-time 7200;
}
```

### 2.3 Cambios más pedidos

**Ampliar o mover el rango:**

```
range 10.3.192.50 10.3.192.250;   # antes 100-200
```

**Cambiar el DNS o el gateway que entrega:** editar `option domain-name-servers` u `option routers` en el bloque de esa subred.

**IP fija por MAC (reserva):** para que un equipo siempre reciba la misma IP:

```
host pc-bog {
    hardware ethernet 0c:aa:bb:cc:dd:ee;   # MAC del equipo
    fixed-address 10.3.192.10;
}
```

La MAC la sacas con `ip link show eth0` en el PC destino. La IP fija debe estar **fuera** del `range` para no chocar.

### 2.4 Aplicar y verificar

```bash
# 1. Validar sintaxis ANTES de reiniciar (un typo deja el DHCP caído)
dhcpd -t -cf /etc/dhcp/dhcpd.conf

# 2. Reiniciar (DHCP no tiene "reload", hay que reiniciar)
service isc-dhcp-server restart
# o directo:  kill $(pidof dhcpd); dhcpd -cf /etc/dhcp/dhcpd.conf eth0

# 3. Ver asignaciones activas
cat /var/lib/dhcp/dhcpd.leases | grep -E 'lease|hardware|binding'

# 4. Probar desde un PC cliente
dhclient -v eth0     # debe mostrar DHCPDISCOVER -> DHCPOFFER -> DHCPACK
```

### 2.5 El relay (clave para sedes remotas)

El DHCP está solo en Bogotá, pero reparte a las cuatro sedes. Eso funciona porque los switches L3 de cada sede tienen configurado el **relay** (`ip helper-address`) en la SVI de la VLAN de usuarios:

```
interface Vlan20
 ip helper-address <IP-del-DHCP-de-Bogota>
```

Si te piden habilitar DHCP en una sede que no lo recibe, el arreglo casi nunca está en el servidor DHCP: está en **añadir esa línea en el switch** de la sede (en los dos, SW1 y SW2, por el HSRP), y en crear el bloque `subnet` correspondiente en `dhcpd.conf`.

---

## PARTE 3 — Diagnóstico exprés (si algo falla en vivo)

| Síntoma | Primer comando | Causa típica |
|---|---|---|
| El cambio de DNS no se ve | `dig @localhost <nombre>` | No subiste el serial, o no recargaste |
| `named` no arranca tras editar | `named-checkzone` / `named-checkconf` | Typo o falta un punto final en un FQDN |
| El PC no recibe IP | `dhclient -v eth0` en el PC | Falta `ip helper-address` en el switch |
| `dhcpd` no arranca | `dhcpd -t` | Typo en `dhcpd.conf`, o subred sin declarar |
| Resuelve IP pero no `dig -x` | revisar zona inversa | Falta el PTR o su serial |

**Regla de oro para no romper nada en la sustentación:** antes de recargar, SIEMPRE valida sintaxis (`named-checkzone` para DNS, `dhcpd -t` para DHCP). Esos dos comandos te dicen si el cambio es seguro sin tocar el servicio en marcha.

---

## PARTE 4 — El error clásico de los FQDN en BIND

En los archivos de zona, un nombre que **termina en punto** es absoluto; uno **sin punto** se le pega el dominio. Esto causa la mitad de los fallos:

```dns
web    IN  CNAME  servidor.redes2026.local.   ; CORRECTO (punto final)
web    IN  CNAME  servidor.redes2026.local    ; MAL -> se vuelve servidor.redes2026.local.redes2026.local
```

Cuando algo resuelve a un nombre duplicado y raro, casi siempre es un punto final que falta.
