# Notas DNS - BIND9 en contenedor Ubuntu (GNS3)

Dominio: networking.mx
IP del server DNS: 60.60.60.3

Aquí anoto cómo quedó esto y en qué me trabé, para no volver a perder el tiempo la próxima vez.

## Instalar

```
apt update
apt install bind9 bind9utils dnsutils -y
```

## Los archivos que hay que tocar

Son 4, todos en /etc/bind/

### 1. Opciones generales

```
nano /etc/bind/named.conf.options
```

Contenido:
```
options {
        directory "/var/cache/bind";

        listen-on port 53 { 127.0.0.1; 60.60.60.3; };
        allow-query { any; };
        recursion yes;

        dnssec-validation auto;

        listen-on-v6 { any; };
};
```

Para la próxima práctica: nomás cambiar el 60.60.60.3 por la IP que le toque al server ahí.

### 2. Declarar las zonas

```
nano /etc/bind/named.conf.local
```

Contenido:
```
zone "networking.mx" {
        type master;
        file "/etc/bind/db.networking.mx";
};

zone "60.60.60.in-addr.arpa" {
        type master;
        file "/etc/bind/rev.60.60.60";
};
```

Cambiar el dominio y el nombre de la zona inversa según la red que toque (si fuera 172.16.5.0/24 sería 5.16.172.in-addr.arpa).

### 3. Zona directa (nombre -> ip)

```
nano /etc/bind/db.networking.mx
```

Contenido:
```
$TTL 86400
@	IN	SOA	dns.networking.mx. root.networking.mx. (
			2026100601; numero de serie
			604800;tiempo de referesco
			86400;tiempo entre reintentos
			604800;tiempo expira
			86400);ttl
@	IN	NS	dns.networking.mx.
@	IN	A	60.60.60.3
dns.networking.mx.	IN	A	60.60.60.3
dhcp			IN	A	60.60.60.1
www1			IN	A	50.50.50.1
www2			IN	A	50.50.50.2
www3			IN	A	60.60.60.2
```

El serial (2026100601) hay que subirlo cada que se edita el archivo o a veces BIND no detecta el cambio.

### 4. Zona inversa (ip -> nombre)

```
nano /etc/bind/rev.60.60.60
```

Contenido:
```
$TTL 86400
@	IN	SOA	dns.networking.mx. root.networking.mx. (
			2026100601; numero de serie
			604800;tiempo de referesco
			86400;tiempo entre reintentos
			604800;tiempo expira
			86400);ttl
@	IN	NS	dns.networking.mx.
3	IN	PTR	dns.networking.mx.
1	IN	PTR	dhcp.networking.mx.
2	IN	PTR	www3.networking.mx.
```

Nomás va el último octeto de la ip (el 3 es por el .3). Si hay equipos en otra red (aquí www1 y www2 están en 50.50.50.0/28) esos necesitarían su propia zona inversa aparte, no los metí porque no los pidieron.

## Permisos

```
chown root:bind /etc/bind/db.networking.mx /etc/bind/rev.60.60.60
chmod 644 /etc/bind/db.networking.mx /etc/bind/rev.60.60.60
```

## Checar que no haya errores antes de arrancar

```
named-checkconf /etc/bind/named.conf
named-checkzone networking.mx /etc/bind/db.networking.mx
named-checkzone 60.60.60.in-addr.arpa /etc/bind/rev.60.60.60
```

Las zonas tienen que decir OK. El checkconf si no dice nada ya está bien.

## Arrancar el servicio - aquí fue donde batallé

Estos contenedores no tienen systemd:

`service bind9 start` -> da error "unrecognized service", no sirve
`systemctl start bind9` -> da error de que no hay systemd, tampoco sirve

Lo que sí sirve:
```
named -u bind
```

Para checar que sigue vivo:
```
ps aux | grep named
```

Si no arranca y quiero ver el error real:
```
named -g -u bind
```
Se queda mostrando los logs, si no se queja de nada está bien, le doy Ctrl+C y lo vuelvo a correr normal (sin -g) para que quede en background.

Cosa importante: cada que se reinicia el nodo o se cierra y abre el proyecto en GNS3, el proceso se muere y hay que correr otra vez `named -u bind` a mano, no hay forma de dejarlo enabled sin systemd.

## Otro problema que me salió: la ip no se aplicaba

Al editar la interfaz con:
```
nano /etc/network/interfaces
```
el cambio no se aplica solo, hay que forzarlo:
```
ifdown eth0
ifup eth0
ip addr show eth0
```
Si no hago esto el server se queda con otra ip (la de docker por default tipo 192.168.122.x) y named ni siquiera puede levantar porque la ip que puse en named.conf.options no existe en la interfaz. Da el error "failed to create listening socket: Cannot assign requested address"

## El error tonto que me quitó rato

Al editar named.conf.options con nano, en vez de borrar lo viejo pegué encima y quedó duplicada la línea dnssec-validation. named-checkconf marca:
```
'dnssec-validation' redefined near 'dnssec-validation'
```

Mejor manera de arreglarlo, en vez de buscar la línea duplicada a mano, vaciar el archivo primero:
```
echo "" > /etc/bind/named.conf.options
nano /etc/bind/named.conf.options
```
y ahí sí pegar todo de nuevo limpio.

## Probar que resuelve

```
dig @localhost networking.mx
dig @localhost dhcp.networking.mx
dig @localhost www1.networking.mx
```

Si dice connection refused es que named no está corriendo, regresar al paso de arriba.

## Para que los clientes usen este DNS

En el dhcpd.conf del server DHCP, dentro de cada subnet:
```
option domain-name-servers 60.60.60.3;
option domain-name "networking.mx";
```

En el cliente checar que agarró el dns bien:
```
cat /etc/resolv.conf
```
Tiene que decir nameserver 60.60.60.3. Si no, forzar que pida ip de nuevo:
```
udhcpc -i eth0
```

Y ya:
```
ping dns.networking.mx
ping dhcp.networking.mx
ping www1.networking.mx
```

## Resumen de pasos para la próxima vez que monte esto

1. apt install bind9 bind9utils dnsutils -y
2. poner ip estática en /etc/network/interfaces, luego ifdown/ifup
3. editar los 4 archivos cambiando dominio/ips/zona inversa
4. chown y chmod a los dos archivos de zona
5. named-checkconf y named-checkzone x2, no debe haber error
6. named -u bind (nada de service ni systemctl)
7. ps aux | grep named para confirmar
8. dig @localhost para probar local
9. checar dhcpd.conf que mande el domain-name-servers correcto
10. probar desde cliente con ping después de renovar ip

si se reinicia el nodo, repetir el paso 6 a mano siempre.
