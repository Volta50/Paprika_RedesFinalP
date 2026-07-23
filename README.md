# Paprika_RedesFinalP

Proyecto final de redes en GNS3: 4 sedes (Bogotá, Santa Marta, Cúcuta y Barranquilla) con EIGRP 100, HSRP, STP, NAT/PAT y eBGP hacia ISP, y los servicios de DHCP, DNS y LDAP montados en contenedores Docker.

## Instalación

Descargar `imagenes-docker.tar` y cargar las imágenes:

```
docker load -i imagenes-docker.tar
```

Con las imágenes cargadas ya se puede importar el proyecto en GNS3.

Si la importación falla porque falta una imagen, se puede renombrar alguno de los clientes al nombre de la imagen faltante para poder importar el proyecto.

## Documentación

| Archivo | Contenido |
|---|---|
| `Configuracion_BGP_NAT.md` | eBGP y NAT/PAT en los routers de borde y los ISP |
| `Implementacion_Servidor_DHCP_GNS3.md` | DHCP centralizado y relay en los switches multicapa |
| `Implementacion_Servidor_DNS_GNS3.md` | DNS con BIND9, zonas directa e inversa |
| `Implementacion_DNS_DHCP_LDAP_Distribuido_GNS3.md` | DNS master/slave, DHCP por sede y LDAP distribuido |
| `LDAP_Setup_Comandos.md` | LDAP en MirrorMode entre las cuatro sedes |
| `Guia_Modificar_DNS_DHCP.md` | Operación y mantenimiento de DNS y DHCP |

## Credenciales

Enable, consola y vty de los equipos Cisco: `Redes2026`
Admin de LDAP: `cn=admin,dc=redes2026,dc=local` con clave `Redes2026`
