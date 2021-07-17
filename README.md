# OpenVPN con Easy-RSA

Este repositorio pretende simplificar los pasos indicados por la [documentación oficial de OpenVPN](https://openvpn.net/community-resources/setting-up-your-own-certificate-authority-ca/)

## Índice
<!--ts-->
* [Servidor](#servidor)
  * [Instalar los paquetes necesarios](#instalar-los-paquetes-necesarios)
  * [Configurar Easy-RSA](#configurar-easy-rsa)
  * [Generar los certificados](#generar-certificados)
    * [CA (ca.crt)](#ca-cacrt)
    * [Servidor (server.crt)](#servidor-servercrt)
    * [Cliente (client.crt)](#cliente-clientcrt)
    * [Autenticación TLS (ta.key)](#autenticación-tls-takey)
  * [Configurar OpenVPN](#configurar-openvpn)
* [Cliente](#cliente)
<!--te-->

## Servidor

### Instalar los paquetes necesarios

```
$ apt install openssl openvpn easy-rsa
```

### Configurar Easy-RSA

1. Copiar los ejemplos que incluye [Easy-RSA](https://openvpn.net/community-resources/rsa-key-management/).

   ```
   $ mkdir /etc/openvpn/easy-rsa
   $ cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
   $ cd /etc/openvpn/easy-rsa/
   ```

   A partir de este momento, trabajaremos casi siempre en el directorio `/etc/openvpn/easy-rsa/`

2. Variables de entorno

   ```
   $ cp vars.example vars
   $ vim vars
   ```

   Dejar todos los valores por defecto, excepto:

   ```sh
   export KEY_SIZE=2048                     # Valores más bajos serán más rápidos, a costa de una menor seguridad
   ...
   export KEY_COUNTRY="US"                  # "ES" para España
   export KEY_PROVINCE="California"         # Admite cualquier cadena de texto: Madrid, Barcelona, etc.
   export KEY_CITY="San Francisco"          # Igual que el anterior: Logroño, Mérida, etc.
   export KEY_ORG="Copyleft Certificate Co" # La compañía (real o ficticia) de la CA (autoridad de certificación)
   export KEY_EMAIL="me@example.net"        # E-mail de contacto
   export KEY_OU="My Organizational Unit"   # Departamento dentro de la compañía responsable de la CA (autoridad de certificación)
   ...
   # export KEY_CN="CommonName"             # Es mejor no fijar este valor y que cada clave generada tenga su propio nombre único
   ```

   Aplicar los valores configurados y limpiar el directorio antes de empezar a trabajar:

   ```
   $ source ./vars
   $ ./clean-all
   ```

   > En algunos sistemas se puede tener el siguiente error:
   >
   > ```
   > **************************************************************
   >   No /etc/openvpn/easy-rsa/openssl.cnf file could be found
   >   Further invocations will fail
   > **************************************************************
   > ```
   >
   > En estos casos hay que indicar de manera explícita dónde está en nuestra distribución el fichero openssl.cnf
   >
   > ```
   > export KEY_CONFIG=`$EASY_RSA/whichopensslcnf $EASY_RSA`
   > ...
   > export KEY_CONFIG=/etc/openvpn/easy-rsa/openssl-1.0.0.cnf
   > ```

Y con esta configuración preliminar ya se puede empezar a generar contenido real (certificados, claves y demás).

### Generar los certificados

#### CA (ca.crt)

```
$ ./build-ca 
Country Name (2 letter code) [US]:
State or Province Name (full name) [California]:
Locality Name (eg, city) [San Francisco]:
Organization Name (eg, company) [Copyleft Certificate Co]:
Organizational Unit Name (eg, section) [My Organizational Unit]:
Common Name (eg, your name or your server's hostname) [Copyleft Certificate Co CA]:
Name [EasyRSA]:
Email Address [me@example.net]:
```

#### Servidor (server.crt)

```
$ ./build-key-server <server_name>
```

#### Cliente (client.crt)

```
$ ./build-key-pass <client_name>
```

Nos pedirá introducir una contraseña PEM (Privacy-Enhanced Mail), que deberemos memorizar para su uso en el cliente.

#### Autenticación TLS (ta.key)

La autenticación TLS dará un plus de seguridad a nuestra configuración. Para ello hay que seguir estos pasos:

1. Cifrar la clave de cliente:

   ```
   $ cd keys/
   $ openssl rsa -in <client_name>.key -des3 -out <client_name>.3des.key
   $ cd ..
   ```

2. Parámetros de Diffie-Hellman

   Este es, con diferencia, el paso más largo de todo el proceso:

   ```
   $ ./build-dh 
   ```

   Puede tardar desde 5/10 minutos hasta varias horas. Personalmente, nunca he visto que tarde más de 5/10 minutos.

3. Clave para autenticación TLS (ta.key)

   ```
   $ openvpn --genkey --secret keys/ta.key
   ```

¡Y por fin tenemos todas las claves y certificados que vamos a necesitar!

### Configurar OpenVPN

```
$ vim /etc/openvpn/server.conf
```

En [nuestro ejemplo](https://github.com/lopecillo/openvpn-easyrsa/blob/master/openvpn/server.conf) hemos utilizado la red `10.8.0.0/24` y el servidor está en la dirección `192.168.1.100`, obviamente se puede cambiar por cualquier otra.

1. Reglas de firewall para OpenVPN

   ```sh
   #!/bin/sh
   iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o enp2s0 -j SNAT --to-source 192.168.1.100
   ```

   ```
   $ vim /etc/firewall-openvpn-rules.sh
   $ chmod 700 /etc/firewall-openvpn-rules.sh
   ```

2. Activar firewall en el arranque

   Configurar [las interfaces de red](https://github.com/lopecillo/openvpn-easyrsa/blob/master/openvpn/interfaces) para que las reglas de firewall se activen automáticamente al levantar la interfaz:

   ```
   pre-up /etc/firewall-openvpn-rules.sh
   ```

   ```
   $ vim /etc/network/interfaces
   ```

¡Y ya hemos terminado con el servidor! Ya solo queda preparar la configuración del cliente.

## Cliente

Copiar las claves para el cliente:

```
/etc/openvpn/easy-rsa/keys/ca.crt
/etc/openvpn/easy-rsa/keys/<client_name>.crt
/etc/openvpn/easy-rsa/keys/<client_name>.3des.key
/etc/openvpn/easy-rsa/keys/ta.key
```

Y volcar en un fichero `.ovpn`. Se puede tomar como modelo [este ejemplo](https://github.com/lopecillo/openvpn-easyrsa/blob/master/openvpn/client-example.ovpn).

Salvando las distancias, el proceso es algo parecido a:

```
$ cat openvpn_header.txt ca.crt <client_name>.crt <client_name>.3des.key ta.key > <client>.ovpn
```
