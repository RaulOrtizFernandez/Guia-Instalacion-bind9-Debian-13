# Guia-Instalacion-bind9-Debian-13

**Autor:** Raúl Ortiz  
**Sistema:** Debian GNU/Linux 13 (Trixie) en VirtualBox  
**Objetivo:** Resolver nombres de la zona `haven.local` y realizar consultas inversas de la red `192.168.1.0/24`.

**Estado del informe:** Se documenta la configuración trabajada y las comprobaciones que deben ejecutarse. Durante la práctica se confirmó que `named` escuchaba en `192.168.1.100:53` por UDP y TCP. No se ha mostrado todavía una salida final de las consultas `nslookup`/`dig` que confirme que las dos zonas responden; no se da esa validación por realizada.

## 1. Entorno de la práctica

La máquina virtual cuenta con dos interfaces observadas mediante `ip addr`:

| Interfaz | Configuración observada | Uso |
| --- | --- | --- |
| `enp0s3` | `10.0.2.15/24`, obtenida por DHCP | Conectividad NAT de la VM. |
| `enp0s8` | `192.168.1.100/24`, estática | Dirección utilizada por BIND9 para atender consultas DNS. |

La zona directa se llama `haven.local`. El nombre elegido para el servidor DNS es `raul.haven.local`, asociado a `192.168.1.100`. La zona inversa de `192.168.1.0/24` es `1.168.192.in-addr.arpa`.

**Nota de red:** En una captura de `ip addr`, `enp0s8` mostraba `brd 192.168.6.255` pese a tener `192.168.1.100/24`. Conviene comprobar que, tras aplicar la configuración, el broadcast sea `192.168.1.255`. El modo exacto del segundo adaptador de VirtualBox no quedó acreditado: tener NAT en el primero no demuestra por sí mismo que un cliente pueda llegar a `enp0s8`.

## 2. Instalación de BIND9

Se utiliza BIND9 como servidor DNS, las utilidades para validar archivos de configuración y, opcionalmente, las herramientas de consulta:

```bash
sudo apt update
sudo apt install bind9 bind9-utils bind9-dnsutils
```

`bind9-dnsutils` proporciona herramientas como `dig` y `nslookup`; si ya están instaladas, no es necesario reinstalarlas. Para revisar el servicio:

```bash
sudo systemctl status bind9
```

## 3. Dirección de red del servidor

Archivo: `/etc/network/interfaces`.

```conf
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug enp0s3
iface enp0s3 inet dhcp

allow-hotplug enp0s8
iface enp0s8 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    broadcast 192.168.1.255
    dns-nameservers 192.168.1.100
```

`enp0s3` conserva la configuración DHCP de la red NAT. `enp0s8` usa la IP fija del DNS. No se añade una segunda puerta de enlace a `enp0s8`, porque no se confirmó que exista un router en esa red.

Para comprobar la IP y las rutas realmente activas:

```bash
ip addr show enp0s8
ip route
```

La línea `dns-nameservers` no garantiza por sí sola que el sistema cambie `/etc/resolv.conf`: depende de cómo gestione Debian la resolución DNS. Por eso las primeras pruebas consultan el servidor explícitamente mediante `192.168.1.100`.

## 4. Opciones generales de BIND9

Archivo: `/etc/bind/named.conf.options`.

```conf
acl "safeclients" {
    localhost;
    192.168.1.0/24;
};

options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { safeclients; };
    allow-query { safeclients; };
    allow-query-cache { safeclients; };

    listen-on port 53 { 127.0.0.1; 192.168.1.100; };
    listen-on-v6 { none; };

    allow-transfer { none; };

    forwarders {
        1.1.1.1;
        9.9.9.9;
    };
};
```

La lista `safeclients` restringe las consultas a la propia máquina y a la red `192.168.1.0/24`. `listen-on` indica dónde atiende BIND. Los `forwarders` se emplean para consultas externas que el servidor no responde desde sus propias zonas. Se deshabilitan las transferencias de zona en este ejemplo porque no hay un servidor secundario configurado.

En la configuración mostrada durante la práctica, BIND escuchaba solo en `192.168.1.100`; se incluye `127.0.0.1` en esta versión para poder probarlo también desde el propio servidor. No es necesario modificar `/etc/default/named` para forzar `-4` en esta práctica.

## 5. Declaración de las zonas

Archivo: `/etc/bind/named.conf.local`.

```conf
// Zona directa
zone "haven.local" {
    type master;
    file "/etc/bind/zones/db.haven.local";
};

// Zona inversa para 192.168.1.0/24
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.1.168.192";
};
```

Esta declaración se observó en la captura de la práctica. `type master;` es válido para una zona primaria. Antes de crear los dos archivos de zona hay que crear su directorio:

```bash
sudo mkdir -p /etc/bind/zones
```

**Atención al orden de los octetos:** Para la red `192.168.1.0/24` la zona inversa es `1.168.192.in-addr.arpa`. El nombre del archivo es libre, siempre que coincida exactamente con el valor de `file`.

## 6. Archivo de zona directa

Archivo: `/etc/bind/zones/db.haven.local`.

La captura inicial de esta zona mostraba un servidor llamado `ns.haven.local`, con su registro `A` apuntando a `192.168.1.100`. Después se decidió sustituir ese nombre por `raul.haven.local`. La versión coherente con el nombre finalmente elegido es:

```dns
$TTL 604800
@   IN  SOA raul.haven.local. hostmaster.haven.local. (
        2026092803 ; serial: incrementar al modificar la zona
        12h        ; refresh
        15m        ; retry
        3w         ; expire
        2h         ; TTL de caché negativa
)

@       IN  NS  raul.haven.local.
raul    IN  A   192.168.1.100
```

`@` representa el origen de la zona, `haven.local`. El primer nombre del SOA es el servidor principal (`raul.haven.local.`); `hostmaster.haven.local.` representa la dirección de contacto `hostmaster@haven.local`. El NS declara el servidor autoritativo y el A le asigna su IP. El punto final en los nombres completos evita que BIND les añada de nuevo el sufijo de la zona.

**Importante:** Si el serial real del archivo ya es igual o superior a `2026092803`, debe aumentarse; no se debe reducir al copiar el ejemplo. El `@` al inicio del SOA de la captura original era correcto: lo que cambió fue el nombre elegido para el servidor.

## 7. Archivo de zona inversa

Archivo: `/etc/bind/zones/db.1.168.192`.

No se mostró en la conversación una captura del contenido original de este archivo. Esta es la configuración propuesta para que la IP del servidor resuelva al nombre finalmente elegido:

```dns
$TTL 604800
@   IN  SOA raul.haven.local. hostmaster.haven.local. (
        2026092803 ; serial: incrementar al modificar la zona
        12h        ; refresh
        15m        ; retry
        3w         ; expire
        2h         ; TTL de caché negativa
)

@       IN  NS  raul.haven.local.
100     IN  PTR raul.haven.local.
```

El registro `100 IN PTR` usa el último octeto de `192.168.1.100`. Al consultar esa IP de forma inversa, el resultado esperado es `raul.haven.local.`. En cada zona se debe incrementar su propio serial cuando se cambie el archivo correspondiente; no es obligatorio que los seriales de ambas zonas coincidan.

## 8. Comprobación de sintaxis y servicio

Antes de reiniciar BIND, comprobar los dos archivos y la carga de las zonas:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.1.168.192
sudo named-checkconf -z
```

Cada `named-checkzone` debería terminar en `OK`. Si falla alguno, corregir el error antes de continuar. Después:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
sudo ss -lntup | grep ':53'
```

En una captura de la práctica, `ss` mostró a `named` escuchando en `192.168.1.100:53` tanto en UDP como en TCP. Las entradas de `avahi-daemon` en `:5353` pertenecen a mDNS, no al servicio DNS de BIND en el puerto 53. Si el reinicio falla, consultar:

```bash
sudo journalctl -u bind9 --no-pager -n 50
```

## 9. Pruebas de resolución

**Consulta directa** — debe devolver `192.168.1.100`:

```bash
nslookup raul.haven.local 192.168.1.100
```

**Consulta inversa** — debe devolver `raul.haven.local`:

```bash
nslookup 192.168.1.100 192.168.1.100
```

Alternativamente, con `dig`:

```bash
dig @192.168.1.100 raul.haven.local A +short
dig @192.168.1.100 -x 192.168.1.100 +short
```

Las salidas esperadas de las dos consultas son, respectivamente, `192.168.1.100` y `raul.haven.local.`. Para comprobar que los reenviadores y la salida a Internet funcionan:

```bash
nslookup example.com 192.168.1.100
```

También puede hacerse en modo interactivo:

```text
nslookup
> server 192.168.1.100
> raul.haven.local
> 192.168.1.100
> exit
```

Indicar la IP del servidor es importante porque en la primera prueba del ejercicio `nslookup` estaba usando `192.168.111.1`, no `192.168.1.100`. Además, se escribió `ns` como consulta; eso buscó un nombre llamado `ns` y produjo `NXDOMAIN`. Esa respuesta no demuestra que BIND estuviese apagado.

Si las consultas explícitas funcionan pero `nslookup raul.haven.local` **sin indicar servidor** falla, revisar qué DNS usa Debian:

```bash
cat /etc/resolv.conf
resolvectl status
```

`resolvectl` puede no estar disponible si no se utiliza `systemd-resolved`. Un cliente externo también necesita conectividad hasta `192.168.1.100` y configurarlo como DNS. La red NAT de una interfaz de VirtualBox y la red configurada en la segunda interfaz deben evaluarse por separado.

## 10. Incidencias y decisiones

1. **Consulta al DNS equivocado:** La prueba inicial mostraba como servidor `192.168.111.1`. Para aislar el problema se especificó `192.168.1.100` en cada consulta.
2. **Diferencia con la guía:** La guía de referencia usaba un archivo inverso relacionado con `192.168.6.0/24`, mientras que la interfaz activa del servidor tenía `192.168.1.100/24`. Se adaptó la zona inversa a `1.168.192.in-addr.arpa`.
3. **Nombre del servidor:** La zona directa mostrada inicialmente usaba `ns.haven.local`. Se eligió `raul.haven.local`, que debe aparecer de forma coherente en `SOA`, `NS`, `A` y `PTR`.
4. **Broadcast a revisar:** Una salida de `ip addr` mostraba `192.168.6.255` para `192.168.1.100/24`. Se recomienda verificar la configuración activa de `enp0s8`; no se atribuye automáticamente a esto el fallo de DNS.
5. **Validación pendiente:** La captura del puerto 53 acredita que BIND escucha, pero no que `raul.haven.local` resuelva. Para cerrar el informe con resultado positivo hay que incorporar las salidas reales de las pruebas del apartado 9.

## 11. Resultado y alcance

La configuración descrita permite servir la zona directa `haven.local` y la inversa de `192.168.1.0/24`, con `raul.haven.local` asociado a `192.168.1.100`. La escucha de BIND en UDP y TCP 53 quedó comprobada; la resolución efectiva de los registros queda pendiente de documentar con las salidas de las consultas.

> **Nota sobre el nombre de dominio:** `.local` se utiliza aquí por seguir el enunciado o la guía de laboratorio, pero tiene un uso especial para mDNS. En otros entornos podría causar comportamientos diferentes entre aplicaciones. Para esta práctica se consulta BIND directamente especificando `192.168.1.100`.
