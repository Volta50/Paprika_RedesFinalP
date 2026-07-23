# Configuración de eBGP y NAT/PAT en los bordes de Internet

Routers de borde: R1-BOG, R1-SMA, R1-CUC y R1-BAQ (c7200). Cada uno sale por su interfaz `f3/1` hacia el `ISP-<ciudad>` correspondiente, todos ellos en el AS 65000.

El diseño no usa un único punto de salida. Cada sede tiene su propio AS y su propio ISP simulado, y sale directo a Internet por su enlace local.

## 1. Direccionamiento de los enlaces de borde

| Sede | Router | AS local | IP R1 (f3/1) | IP ISP (f0/0) | AS ISP |
|---|---|---|---|---|---|
| BOG | R1-BOG | 65001 | 203.0.113.1/30 | 203.0.113.2 | 65000 |
| SMA | R1-SMA | 65002 | 203.0.113.9/30 | 203.0.113.10 | 65000 |
| CUC | R1-CUC | 65003 | 203.0.113.5/30 | 203.0.113.6 | 65000 |
| BAQ | R1-BAQ | 65004 | 203.0.113.13/30 | 203.0.113.14 | 65000 |

La red interna que maneja EIGRP 100 es `10.0.0.0/8`.

## 2. NAT/PAT

La lógica es la misma en los cuatro routers: `f3/1` queda como `nat outside` y todas las interfaces internas del router (`f0/0`, `f3/0`, seriales y Metro-Ethernet hacia otras sedes) quedan como `nat inside`.

### 2.1 Access-list del tráfico a traducir

Se tradujo la red interna completa, de modo que cualquier host de cualquier sede pueda salir por el borde local:

```
access-list 1 permit 10.0.0.0 0.255.255.255
```

La misma ACL se aplicó en los cuatro routers, así el tráfico que llega por EIGRP desde otra sede y sale por este borde también se traduce.

### 2.2 PAT con overload

```
ip nat inside source list 1 interface FastEthernet3/1 overload
```

### 2.3 Marcado de interfaces

```
interface FastEthernet3/1
 ip nat outside

interface FastEthernet0/0
 ip nat inside
interface FastEthernet3/0
 ip nat inside
```

Lo mismo en cada interfaz WAN interna (seriales y Metro-Ethernet) hacia las otras sedes. Para revisar qué quedó aplicado:

```
show ip interface f3/1 | include NAT
show run | include ip nat
```

## 3. Anuncio BGP de la red interna

BGP solo anuncia prefijos que existan exactos en la tabla de rutas. Como las VLANs del proyecto mezclan /17, /18 y /19 y no caen en un resumen limpio por sede, se optó por anunciar `10.0.0.0/8` desde las cuatro sedes. El ISP ve cuatro caminos hacia la misma red interna y elige por AS-path y métrica, y se evita declarar una estática Null0 por cada resumen intermedio.

Para que BGP tenga la ruta exacta que anunciar se agregó una estática de resumen hacia Null0 con distancia administrativa 254:

```
ip route 10.0.0.0 255.0.0.0 Null0 254
```

Con AD 254 nunca compite con las rutas EIGRP reales (AD 90), que siguen siendo las que transportan el tráfico interno.

## 4. Configuración aplicada en cada router de borde

**R1-BOG**
```
configure terminal
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface FastEthernet3/1 overload
ip route 10.0.0.0 255.0.0.0 Null0 254
router bgp 65001
 bgp router-id 203.0.113.1
 neighbor 203.0.113.2 remote-as 65000
 network 10.0.0.0 mask 255.0.0.0
end
write memory
```

**R1-SMA**
```
configure terminal
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface FastEthernet3/1 overload
ip route 10.0.0.0 255.0.0.0 Null0 254
router bgp 65002
 bgp router-id 203.0.113.9
 neighbor 203.0.113.10 remote-as 65000
 network 10.0.0.0 mask 255.0.0.0
end
write memory
```

**R1-CUC**
```
configure terminal
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface FastEthernet3/1 overload
ip route 10.0.0.0 255.0.0.0 Null0 254
router bgp 65003
 bgp router-id 203.0.113.5
 neighbor 203.0.113.6 remote-as 65000
 network 10.0.0.0 mask 255.0.0.0
end
write memory
```

**R1-BAQ**
```
configure terminal
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface FastEthernet3/1 overload
ip route 10.0.0.0 255.0.0.0 Null0 254
router bgp 65004
 bgp router-id 203.0.113.13
 neighbor 203.0.113.14 remote-as 65000
 network 10.0.0.0 mask 255.0.0.0
end
write memory
```

## 5. Routers ISP

Los ISP se agregaron como nodos nuevos, así que hubo que configurarles la interfaz con IP y `no shutdown` además del BGP. Sin ninguna IP configurada, BGP no puede elegir router-id (aparece `%BGP-4-NORTRID`) y el vecino queda en `Idle`; además el ping desde la red interna muere en el borde porque nadie responde ARP por la IP del ISP.

**ISP-BOG**
```
configure terminal
hostname ISP-BOG
interface FastEthernet0/0
 ip address 203.0.113.2 255.255.255.252
 no shutdown
 exit
router bgp 65000
 bgp router-id 203.0.113.2
 neighbor 203.0.113.1 remote-as 65001
end
write memory
```

**ISP-SMA**
```
configure terminal
hostname ISP-SMA
interface FastEthernet0/0
 ip address 203.0.113.10 255.255.255.252
 no shutdown
 exit
router bgp 65000
 bgp router-id 203.0.113.10
 neighbor 203.0.113.9 remote-as 65002
end
write memory
```

**ISP-CUC**
```
configure terminal
hostname ISP-CUC
interface FastEthernet0/0
 ip address 203.0.113.6 255.255.255.252
 no shutdown
 exit
router bgp 65000
 bgp router-id 203.0.113.6
 neighbor 203.0.113.5 remote-as 65003
end
write memory
```

**ISP-BAQ**
```
configure terminal
hostname ISP-BAQ
interface FastEthernet0/0
 ip address 203.0.113.14 255.255.255.252
 no shutdown
 exit
router bgp 65000
 bgp router-id 203.0.113.14
 neighbor 203.0.113.13 remote-as 65004
end
write memory
```

### 5.1 Loopbacks 8.8.8.8 y 1.1.1.1

Los cuatro ISP comparten el AS 65000 pero no están conectados entre sí, funcionan como islas independientes. Para que `8.8.8.8` responda desde las cuatro sedes se configuraron las mismas loopbacks en los cuatro ISP; repetir la IP entre islas no genera conflicto.

```
configure terminal
interface Loopback0
 ip address 8.8.8.8 255.255.255.255
interface Loopback1
 ip address 1.1.1.1 255.255.255.255
 exit
router bgp 65000
 network 8.8.8.8 mask 255.255.255.255
 network 1.1.1.1 mask 255.255.255.255
end
write memory
```

## 6. Verificación

En cada R1:

```
show ip bgp summary
show ip route bgp
show ip nat translations
show ip nat statistics
```

En `show ip bgp summary` la columna State/PfxRcd debe mostrar el número de prefijos recibidos, no `Idle` ni `Active`.

Desde un VPCS interno:

```
VPCS> ping 203.0.113.2
```

(o .10 / .6 / .14 según la sede). Mientras el VPCS hace ping, en el router:

```
R1-BOG# show ip nat translations
```

La salida obtenida:

```
Pro  Inside global      Inside local       Outside local      Outside global
icmp 203.0.113.1:5      10.2.255.253:5     203.0.113.2:5      203.0.113.2:5
```

La IP interna `10.2.255.253` sale traducida a `203.0.113.1`, la IP pública de R1-BOG.

Con las loopbacks de la sección 5.1 configuradas, `ping 8.8.8.8` funciona desde el servidor DNS o desde un VPCS. No hay un resolutor real detrás de esa loopback, así que `dig google.com` no resuelve; la loopback solo comprueba que el camino y la traducción funcionan. La resolución interna de `redes2026.local` es la que se documenta como entregable.

## 7. Puntos revisados al cerrar

- `show ip bgp summary` en los cuatro R1 y en los cuatro ISP, con la sesión en `Established`.
- La estática `ip route ... Null0 254` presente y sin competir con EIGRP.
- `access-list 1` cubriendo `10.0.0.0/8` en los cuatro routers.
- `ip nat inside` e `ip nat outside` correctos en todas las interfaces, revisado con la sección "Interfaces" de `show ip nat statistics`.
- Entradas activas en `show ip nat translations` al generar tráfico desde un cliente interno.
