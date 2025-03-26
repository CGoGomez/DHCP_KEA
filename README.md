# DHCP_KEA

En el ámbito DHCP, el servicio más utilizado suele ser "isc-dhcp-server" pero hay otros métodos para otorgar IPs a otros equipos, 
en este arhivo veremos la configuración de KEA; un servicio de direcciones IP dinámicas (o estáticas) para sistemas Linux, el programa ha sido instalado en
una máquina Debian 12. Estos son los pasos a seguir para que funcione. Este repositorio mostrará su realización en un par de máquinas virtuales; además del Debian 12
también se configurará un Windows como nuestro cliente.

1.  Actualización de paquetes.

Lo habitual antes de realizar cualquier cosa con cualquier programa es actualizar los repositorios del sistema, podría ser que la versión que utilizamos acaba de actualizar.
En tal caso hacemos los siguientes comandos:

``` bash
apt update
apt upgrade
```
El primero dirá cuántos paquetes neceitan de actualizarse, el segundo actualizará todos los programas y repositorios (puede llevar un rato dependiendo de la cantidad)

2.  KEA

Una vez todo se haya actualizado, es recomendable comprobar que el paquete no está instalado por defecto.
``` bash
apt policy kea
```
En caso de que esté instalado, la terminal sacará un texto que indicará la versión que tenemos instlada, en caso contrario, indicará que no hay ningún paquete instalado
en su última versión, es el turno del usuario para instalar kea en el ordenador, a saber que no es necesario indicar ninguna versión a la hora de instalarlo, el propio
debian tomará la versión más reciente de los repositorios que hemos actualizado anteriormente.
Para instalarlo se usa el siguiente comando:
``` bash
apt install kea
```
Y tras esperar un rato, KEA estará instalado y activo en nuestro ordenador.

3.  Las Tarjetas de Red

Este es un paso importante, a tener en cuenta de que nuestro objetivo es que el otorgar una salida a internet por medio de KEA, pero para ello primero hemos de asegurarnos que
el sistema que usamos tiene salida a internet y además pueda tener su propia red interna con la cual poder unir diferentes máquinas. Por ese motivo es recomendable que este experimento
se realice antes en máquinas virtuales que en la máquina en la cual trabajas de diário.
Para la primera máquina, nuestro servidor, necesitarás un total de dos tarjetas de red, una es la red principal que por defecto tendrá salida a internet, la otra tarjeta estará configurada como una
red interna que permita todas las conexiones. Posteriormente, es recomendable que tu máquina tenga IPs estáticas, si tienes un debian gráfico es recomendable que desinstales el Network Manager,
pues a la hora de reiniciar el servicio de internet puede dar conflicto con la configuración del estado gráfico; si tienes un debian no gráfico entonces no hay problema.
``` bash
apt purge network-manager
```
¿Qué IPs has de configurar? Para ello, has de identificar los nombres de la tarjetas de red y que IPs tienen por defecto, la primera tarjeta de red que es la que por defecto ya tiene salida a internet
porque es adaptador puente es fácil de reconocer pues tiene dirección IP; la otra tarjeta probablemente al ser una red interna no tenga ningún tipo de IP asignada, tendrás que asignársela tu en el
archivo **/etc/network/interfaces** algo más o menos así:

``` bash
# The Loopback network interface
auto lo
iface lo inet loopback

# Configuración de la tarjeta de red base con salida a internet.
auto **Nombre Tarjeta 1**
iface **Nombre Tarjeta 1** inet static
      address IP estática
      netmask Máscara de Red de la salida a internet
      gateway Puerta de enlace

# Confifuración de la tarjeta que usaremos como parte del servicio
auto **Nombre Tarjeta 2**
iface **Nombre tarjeta 2** inet static
      address IP estática de servicio
      netmask Máscara de red de la red del servicio
```
Tras esto, están configuradas ambas tarjetad de red, pero los cambios no están aplicados hasta que no reiniciemos el servico de red.
``` bash
systemctl restart networking.service
```
Ahora sí, al realizar el comando **ip a** debería mostrarte las dos interfaces de red con las IPs que has asignado. Lo que quedaría por hacer antes de empezar a configurar KEA es 
comprobar que nuestra máquina tiene acceso a internet, para eso es tan sencillo como realizar un ping a cualquier página web, si todo está bien configurado la interfaz principal debería
de darnos conexión y el resultado en milisegundos.
Lo último, debemos añadir en nuestra máquina cliente una tarjeta de red, exclusivamente una con configuración de red interna que permita todo tipo de conexiones. Para asegurarnos de que
está en la misma red, es recomendable ponerle un nombre que sea reconocible dentro de la propia red interna, cualquier nombre que sea distintivo con respecto a la red interna por defecto de
VirtualBox para agilizar además de darle un toque personal y reconocible.

4. Configurando KEA
KEA tiene una configuración curiosa con respecto a otros servicios DHCP, su configuración se encuentra en el directorio **/etc/kea** en donde encontraremos configuración base para
IPv4 e IPv6, con los respectivos nombres **kea-dhcp4.conf** y **kea-dhcp6.conf** pero a nosotros solo nos interesa el primero, para la configuración IPv4, que centra la configurción
de la tarjeta de red que hemos configurado.
Los archivos de configuración de KEA están configurados en formato json, un formato que si bien es bastante tedioso de escribir; se puede comprender. En la documentación oficial de
KEA te otorgan lo que es la estructura principal del arhivo, se ve algo más o menos de la siguiente manera:

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [
        "Tarjeta de Red 2"
      ],
      "dhcp-socket-type": "raw"
    },
    "reservations-global": false,
    "reservations-out-of-pool": true,
    "valid-lifetime": 4000,
    "renew-timer": 1000,
    "rebind-timer": 2000,
    "subnet4": [
      {
        "subnet": "192.168.100.0/24",
        "match-client-id": false,
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.100.1"
          },
          {
            "name": "domain-name-servers",
            "data": "9.9.9.9"
          },
          {
            "name": "ntp-servers",
            "data": "192.168.100.1"
          },
          {
            "name": "domain-name",
            "data": "dominio-100.test"
          }
        ],
        "pools": [
          {
            "pool": "192.168.100.100-192.168.100.199"
          }
        ],
        "reservations": [
          {
            "hw-address": "Dirección MAC de la tarjeta de red del cliente",
            "ip-address": "192.168.100.11"
          }
        ]
      }
    ],
    "loggers": [
      {
        "name": "*",
        "severity": "DEBUG"
      }
    ]
  }
}
```
