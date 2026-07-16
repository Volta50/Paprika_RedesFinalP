# Configuración de BGP (eBGP) y NAT/PAT — Bordes de Internet

**Routers involucrados:** R1-BOG, R1-SMA, R1-CUC, R1-BAQ (c7200) · interfaz `f3/1` en cada uno, hacia su respectivo `ISP-<ciudad>` (AS 65000).

Cada sede tiene **su propio AS** y **su propio ISP simulado** — no hay un único punto de salida, cada router sale directo a Internet por su `f3/1`.

---

## 1. Datos de referencia (del Excel)

| Sede | Router | AS local | IP R1 (f3/1) | IP ISP (f0/0) | AS ISP |
|---|---|---|---|---|---|
| BOG | R1-BOG | 65001 | 203.0.113.1/30 | 203.0.113.2 | 65000 |
| SMA | R1-SMA | 65002 | 203.0.113.9/30 | 203.0.113.10 | 65000 |
| CUC | R1-CUC | 65003 | 203.0.113.5/30 | 203.0.113.6 | 65000 |
| BAQ | R1-BAQ | 65004 | 203.0.113.13/30 | 203.0.113.14 | 65000 |

Red interna anunciada por EIGRP 100: `10.0.0.0/8` (ver resumen del proyecto). Cada sede solo necesita anunciar **su propio bloque** hacia el ISP si quieres que la vuelta sea óptima, o puedes resumir `10.0.0.0/8` completo desde las 4 — más abajo se explican ambas opciones.

---

## 2. NAT/PAT — misma lógica en los 4 routers

Concepto: `f3/1` es `nat outside` (hacia el ISP), **todas** las interfaces internas del router (`f0/0`, `f3/0`, seriales/Metro-E hacia otras sedes) son `nat inside`. Esto ya viene parcialmente aplicado por el patch (`ip nat inside` en cada interfaz interna, `ip nat outside` en `f3/1`) — aquí falta la **access-list** y el comando `overload`.

### 2.1 Access-list de tráfico a traducir

Usamos la red interna completa `10.0.0.0/8` para que cualquier host de cualquier sede pueda salir por el borde local:

```
access-list 1 permit 10.0.0.0 0.255.255.255
```

(Igual en los 4 routers — así el tráfico que llega por EIGRP desde otra sede y sale por este borde también se traduce.)

### 2.2 PAT overload

```
ip nat inside source list 1 interface FastEthernet3/1 overload
```

### 2.3 Verificar que las interfaces ya tengan `nat inside`/`nat outside`

```
show ip interface f3/1 | include NAT
show run | include ip nat
```

Si falta alguna (el patch debería haberlas dejado), agrégalas:

```
interface FastEthernet3/1
 ip nat outside

interface FastEthernet0/0
 ip nat inside
interface FastEthernet3/0
 ip nat inside
! + cada interfaz WAN interna (seriales, Metro-E) hacia otras sedes
```

---

## 3. BGP — configuración por router

> **Sobre los `network`:** BGP solo anuncia lo que exista **exacto** en la tabla de rutas (no VLSM automático). Como tus VLANs no caen limpio en un solo resumen simple por sede (mezcla /17, /18, /19), lo más simple y robusto para este laboratorio es anunciar **`10.0.0.0/8` desde las 4 sedes** — el ISP ve 4 caminos hacia la misma red interna y elige por AS-path/metric, y evitas tener que declarar `ip route 10.x.x.x mask ... Null0` por cada resumen que no exista literal en la tabla:

```
router bgp <AS-local>
 network 10.0.0.0 mask 255.0.0.0
 network 10.0.0.0 mask 255.0.0.0
```

Con ruta estática de resumen apuntando a Null0 (necesaria para que BGP tenga la ruta exacta que anunciar):

```
ip route 10.0.0.0 255.0.0.0 Null0 254
```

Esa ruta con AD 254 nunca compite con las rutas EIGRP internas reales (AD 90) — solo existe para satisfacer el requisito de BGP de tener una ruta exacta que anunciar. Aplica este bloque (`network` + `ip route ... Null0 254`) igual en los 4 routers.

### 3.5 Configuración recomendada por router (versión final, con NAT y BGP juntos)

**R1-BOG:**
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

**R1-SMA:**
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

**R1-CUC:**
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

**R1-BAQ:**
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

---

## 4. Los routers ISP (extremo remoto)

Los ISP son nodos nuevos: hay que configurarles **la interfaz con IP y `no shutdown`** además del BGP. Sin IP en ninguna interfaz, BGP no puede elegir router-id (`%BGP-4-NORTRID`) y el vecino queda en `Idle` para siempre; además el ping desde la red interna muere en el borde porque nadie responde ARP por la IP del ISP.

**ISP-BOG:**
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

**ISP-SMA:**
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

**ISP-CUC:**
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

**ISP-BAQ:**
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

**Loopbacks 8.8.8.8 / 1.1.1.1 (para que los forwarders del DNS respondan):** los 4 ISP comparten AS 65000 pero **no están conectados entre sí** — son islas. Para que `8.8.8.8` responda desde las 4 sedes, configura las mismas loopbacks en los **4** ISP (repetir la IP entre islas no genera conflicto):

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

---

## 5. Verificación

En cada R1:

```
show ip bgp summary                  ! State/PfxRcd — debe decir el número de prefijos, no "Idle" ni "Active"
show ip route bgp                    ! ver rutas aprendidas del ISP (o default si aplica)
show ip nat translations             ! después de generar tráfico
show ip nat statistics
```

Prueba desde un VPCS interno:

```
VPCS> ping 203.0.113.2      ! (o .10/.6/.14 según la sede) — llega hasta el ISP local
```

En el router, mientras el VPCS hace ping:

```
R1-BOG# show ip nat translations
```

Debes ver una entrada tipo:
```
Pro  Inside global      Inside local       Outside local      Outside global
icmp 203.0.113.1:5      10.2.255.253:5     203.0.113.2:5      203.0.113.2:5
```

Eso confirma NAT/PAT funcionando: la IP interna `10.2.255.253` sale traducida a `203.0.113.1` (la IP pública de R1-BOG).

Con las loopbacks de la sección 4 configuradas, `ping 8.8.8.8` desde el Srv-DNS o un VPCS debe funcionar (vía NAT). Nota: no hay un DNS real detrás de la loopback, así que `dig google.com` seguirá sin resolver — la loopback solo prueba que el camino y el NAT funcionan; la resolución interna de `redes2026.local` es el entregable real.

---

## 6. Checklist BGP/NAT

- [ ] `show ip bgp summary` en los 4 R1: vecino en estado `Established` (o número, no `Idle`/`Active`/`Connect`).
- [ ] `show ip bgp summary` en los 4 ISP: mismo resultado desde el otro lado.
- [ ] `ip route ... Null0 254` presente y **no** compitiendo con EIGRP (AD 254 vs AD 90 — EIGRP siempre gana para el tráfico interno real).
- [ ] `access-list 1` cubre `10.0.0.0/8` en los 4 routers.
- [ ] `ip nat inside`/`ip nat outside` correctos en todas las interfaces (usa `show ip nat statistics` → sección "Interfaces").
- [ ] `show ip nat translations` muestra entradas activas al hacer ping/DNS desde un cliente interno hacia una IP pública.
- [ ] (Opcional) Loopback 8.8.8.8/1.1.1.1 en un ISP si quieres que los forwarders del DNS "respondan" dentro del laboratorio.
