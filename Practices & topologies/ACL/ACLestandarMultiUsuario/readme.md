![Topologia](img/multiACL.img)


### Router R1



```
conf term
enable secret cisco123

interface g1/0
no shut

interface g1/0.60
encapsulation dot1Q 60
ip add 172.16.16.254 255.255.255.0

interface g1/0.90
encapsulation dot1Q 90
ip add 172.17.17.254 255.255.255.0

interface fa0/0
ip add 192.168.8.1 255.255.255.252
no shut

router ospf 1
network 172.17.17.0 0.0.0.255 area 0
network 172.16.16.0 0.0.0.255 area 0
network 192.168.8.0 0.0.0.3 area 0

ip domain-name networking.mx
username admin secret admin123
username tux123 secret tux123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 15
login local
transport input ssh
end
wr
```

### Router R2

```text
conf term
enable secret cisco123

interface fa0/0
ip add 192.168.8.2 255.255.255.252
no shut

interface s2/0
ip add 192.168.8.5 255.255.255.252
clock rate 64000
no shut

router ospf 1
network 192.168.8.0 0.0.0.3 area 0
network 192.168.8.4 0.0.0.3 area 0

ip domain-name networking.mx
username admin secret admin123
username tux123 secret tux123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 15
login local
transport input ssh
end
wr
```

### Router R3

```text
conf term
enable secret cisco123

interface s2/0
ip add 192.168.8.6 255.255.255.252
no shut

interface fa0/0
ip add 192.168.8.9 255.255.255.252
no shut

router ospf 1
network 192.168.8.4 0.0.0.3 area 0
network 192.168.8.8 0.0.0.3 area 0

ip domain-name networking.mx
username admin secret admin123
username tux123 secret tux123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 15
login local
transport input ssh
end
wr
```

### Router R4

```text
conf term
enable secret cisco123

interface fa0/0
ip add 192.168.8.10 255.255.255.252
no shut

interface g1/0
ip add 13.13.13.254 255.255.255.0
ip helper-address 172.17.17.1
no shut

interface g3/0
ip add 14.14.14.254 255.255.255.0
ip helper-address 172.17.17.1
no shut

interface g4/0
ip add 15.15.15.254 255.255.255.0
ip helper-address 172.17.17.1
no shut

router ospf 1
network 192.168.8.8 0.0.0.3 area 0
network 13.13.13.0 0.0.0.255 area 0
network 14.14.14.0 0.0.0.255 area 0
network 15.15.15.0 0.0.0.255 area 0

ip dns server
ip name-server 172.17.17.2

ip domain-name networking.mx
username admin secret admin123
username tux123 secret tux123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 15
login local
transport input ssh
end
wr
```

---

##  Servidores



### dhcp-server



```text
auto eth0
iface eth0 inet static
address 172.17.17.1
netmask 255.255.255.0
gateway 172.17.17.254
up echo "nameserver 172.17.17.2" > /etc/resolv.conf
```

### dns-server



```text
auto eth0
iface eth0 inet static
address 172.17.17.2
netmask 255.255.255.0
gateway 172.17.17.254
up echo "nameserver 172.17.17.2" > /etc/resolv.conf
```

### nginx www1-server



```text
auto eth0
iface eth0 inet static
address 172.16.16.1
netmask 255.255.255.0
gateway 172.16.16.254
up echo "nameserver 172.17.17.2" > /etc/resolv.conf
```

### nginx www2-server


```text
auto eth0
iface eth0 inet static
address 172.16.16.2
netmask 255.255.255.0
gateway 172.16.16.254
up echo "nameserver 172.17.17.2" > /etc/resolv.conf
```

---

## Servidor DHCP

Instalar el servicio o usa el contenedor que ya tene todo:
https://github.com/IsaiRamirez11/Escalabilidad_de_redes/tree/main/Resources/Containers/ServidoresDHCP

```
apt update
apt install isc-dhcp-server -y
```

### Configuración del servicio

**Ubicación**

```text
/etc/dhcp/dhcpd.conf
```

**Comandos**

```
echo "" > /etc/dhcp/dhcpd.conf
nano /etc/dhcp/dhcpd.conf
```

**Contenido**

```text
authoritative;

default-lease-time 600;
max-lease-time 7200;
option domain-name "networking.mx";

subnet 172.17.17.0 netmask 255.255.255.0
{
}

subnet 13.13.13.0 netmask 255.255.255.0
{
range 13.13.13.11 13.13.13.100;
option routers 13.13.13.254;
option domain-name-servers 172.17.17.2;
option broadcast-address 13.13.13.255;
}

subnet 14.14.14.0 netmask 255.255.255.0
{
range 14.14.14.11 14.14.14.100;
option routers 14.14.14.254;
option domain-name-servers 172.17.17.2;
option broadcast-address 14.14.14.255;
}

subnet 15.15.15.0 netmask 255.255.255.0
{
range 15.15.15.11 15.15.15.100;
option routers 15.15.15.254;
option domain-name-servers 15.15.15.254;
option broadcast-address 15.15.15.255;
}
```

### Levantar servicio

```
service isc-dhcp-server restart
service isc-dhcp-server status
```



---

## Servidor DNS

Instalar bind o usa el cotenedor que tiene todo:
https://github.com/IsaiRamirez11/Escalabilidad_de_redes/tree/main/Resources/Containers/Servidores%20DNS

```
apt update
apt install bind9 bind9utils dnsutils -y
```

### Declaracion de zonas

**Ubicación**

```text
/etc/bind/named.conf.local
```

**Comandos**

```
echo "" > /etc/bind/named.conf.local
nano /etc/bind/named.conf.local
```

**Contenido**

```text
zone "networking.mx" {
type master;
file "/etc/bind/db.networking.mx";
};

zone "17.17.172.in-addr.arpa" {
type master;
file "/etc/bind/rev.172.17.17";
};

zone "16.16.172.in-addr.arpa" {
type master;
file "/etc/bind/rev.172.16.16";
};
```

### Zona directa

**Ubicacion**

```
/etc/bind/db.networking.mx
```

**Comandos**

```
echo "" > /etc/bind/db.networking.mx
nano /etc/bind/db.networking.mx
```

**Contenido**

```text
$TTL 86400
@ IN SOA dns.networking.mx. root.networking.mx. (
2026102701 ; serie
604800 ; refresco
86400 ; reintentos
604800 ; expira
86400 ) ; ttl
@ IN NS dns.networking.mx.
@ IN A 172.17.17.2
dns IN A 172.17.17.2
dhcp IN A 172.17.17.1
www IN A 172.16.16.1
www1 IN A 172.16.16.1
www2 IN A 172.16.16.2
r1 IN A 192.168.8.1
r2 IN A 192.168.8.2
r3 IN A 192.168.8.6
r4 IN A 192.168.8.10
```

### Zona inversa 

**Ubicación**

```text
/etc/bind/rev.172.17.17
```

**Comandos**

```
echo "" > /etc/bind/rev.172.17.17
nano /etc/bind/rev.172.17.17
```

**Contenido**

```text
$TTL 86400
@ IN SOA dns.networking.mx. root.networking.mx. (
2026102701
604800
86400
604800
86400 )
@ IN NS dns.networking.mx.
1 IN PTR dhcp.networking.mx.
2 IN PTR dns.networking.mx.
```

### Zona inversa 

**Ubicación**

```text
/etc/bind/rev.172.16.16
```

**Comandos**

```
echo "" > /etc/bind/rev.172.16.16
nano /etc/bind/rev.172.16.16
```

**Contenido**

```text
$TTL 86400
@ IN SOA dns.networking.mx. root.networking.mx. (
2026102701
604800
86400
604800
86400 )
@ IN NS dns.networking.mx.
1 IN PTR www1.networking.mx.
2 IN PTR www2.networking.mx.
```

### Opciones generales

**Ubicación**

```text
/etc/bind/named.conf.options
```

**Comandos**

```
echo "" > /etc/bind/named.conf.options
nano /etc/bind/named.conf.options
```

**Contenido**

```text
options {
directory "/var/cache/bind";

listen-on port 53 { 127.0.0.1; 172.17.17.2; };
allow-query { any; };
recursion yes;

dnssec-validation auto;
listen-on-v6 { any; };
};
```

### Permisos

```
chown root:bind /etc/bind/db.networking.mx /etc/bind/rev.172.17.17 /etc/bind/rev.172.16.16
chmod 644 /etc/bind/db.networking.mx /etc/bind/rev.172.17.17 /etc/bind/rev.172.16.16
```

### Validar sintaxis

```
named-checkconf /etc/bind/named.conf
named-checkzone networking.mx /etc/bind/db.networking.mx
named-checkzone 17.17.172.in-addr.arpa /etc/bind/rev.172.17.17
named-checkzone 16.16.172.in-addr.arpa /etc/bind/rev.172.16.16
```

### Arrancar bind

```
pkill dnsmasq
service bind9 restart
service bind9 status
```

---


## ACL de cada router



### ACL Router R1

```text
conf term

! --- ACL 1: SSH solo desde Subred E
access-list 1 remark ACL1 - SSH solo Subred E
access-list 1 permit 14.14.14.0 0.0.0.255

! --- ACL 4: Subred D no accede a VLAN 60 (cerca del destino, salida)
access-list 4 remark ACL4 - Subred D bloqueada hacia VLAN 60
access-list 4 deny 13.13.13.0 0.0.0.255
access-list 4 permit any

! --- ACL 5: Subred F no accede a VLAN 90 (cerca del destino, salida)
access-list 5 remark ACL5 - Subred F bloqueada hacia VLAN 90
access-list 5 permit host 15.15.15.254
access-list 5 deny 15.15.15.0 0.0.0.255
access-list 5 permit any

interface g1/0.60
ip access-group 4 out

interface g1/0.90
ip access-group 5 out

line vty 0 15
access-class 1 in
end
wr
```

### ACL Router R2

```text
conf term

! --- ACL 1: SSH solo desde Subred E
access-list 1 remark ACL1 - SSH solo Subred E
access-list 1 permit 14.14.14.0 0.0.0.255

line vty 0 15
access-class 1 in
end
wr
```

### ACL Router R3

```text
conf term

! --- ACL 1: SSH solo desde Subred E
access-list 1 remark ACL1 - SSH solo Subred E
access-list 1 permit 14.14.14.0 0.0.0.255

line vty 0 15
access-class 1 in
end
wr
```

### ACL Router R4

```text
conf term

! --- ACL 1: SSH solo desde Subred E
access-list 1 remark ACL1 - SSH solo Subred E
access-list 1 permit 14.14.14.0 0.0.0.255

! --- ACL 2: Subred D no accede a Subred F (cerca del destino, salida g4/0)
access-list 2 remark ACL2 - Subred D bloqueada hacia Subred F
access-list 2 deny 13.13.13.0 0.0.0.255
access-list 2 permit any

! --- ACL 3: Subred F no accede a Subred D (cerca del destino, salida g1/0)
access-list 3 remark ACL3 - Subred F bloqueada hacia Subred D
access-list 3 deny 15.15.15.0 0.0.0.255
access-list 3 permit any

interface g1/0
ip access-group 3 out

interface g4/0
ip access-group 2 out

line vty 0 15
access-class 1 in
end
wr
```