
## 1) Escaneo de puertos

Empezamos con el escaneo de puertos.

![Nmap scan](images/pasted-20251102154836.png)

Obtenemos estos resultados:

![Nmap results](images/pasted-20251102154913.png)

---

## 2) Añadir la página al `/etc/hosts` y acceder

Agregamos la página al `/etc/hosts` y accedemos.

![Hosts edit](images/pasted-20251102155326.png)

![Web access](images/pasted-20251102155344.png)

Vemos los recursos de la página:

![Page resources](images/pasted-20251102155541.png)

> Nota: aparece un mensaje de un usuario llamado **john**. Ese nombre puede ser útil más adelante si necesitamos intentar un login.

---

## 3) Fuerza bruta de directorios con `ffuf`

Se me ocurre hacer un ataque de fuerza bruta de directorios.

![ffuf run](images/pasted-20251102160929.png)

`ffuf` probó 9,228 nombres de la lista y te mostró solo los que obtuvieron una respuesta interesante.

### Explicación del comando

#### `ffuf`
- **Qué es:** Es la herramienta `ffuf` (Fuzz Faster U Fool).
- **Propósito:** Ejecutar ataques de fuzzing / brute-force en URLs.

#### `-u http://10.10.195.192/FUZZ`
- **`-u` (URL):** indica la URL objetivo.
- **`FUZZ`**: marcador donde `ffuf` reemplaza con cada entrada de la wordlist (ej. `index.html`, `robots.txt`, `secret`).

#### `-w /usr/share/wordlists/dirb/common.txt`
- **`-w` (wordlist):** ruta al archivo con las palabras a probar.

#### `-e .txt`
- **`-e` (extension):** también prueba cada palabra con la extensión `.txt` (p.ej. `secret` y `secret.txt`).

Continuando con el ataque:

![ffuf results](images/pasted-20251102161407.png)

Vemos que el comando nos devolvió directorios escondidos.

---

## 4) Revisar `/robots.txt` y `/uploads`

Accedemos a `/robots.txt`:

![robots.txt](images/pasted-20251102161731.png)

Accedemos a `/uploads/` y vemos 3 archivos:

![uploads listing](images/pasted-20251102161858.png)

- Una lista de contraseñas:

![password list](images/pasted-20251102161943.png)

- Una especie de carta:

![note file](images/pasted-20251102162016.png)

- Un meme:

![meme](images/pasted-20251102162051.png)

Al intentar extraer datos de la imagen nos pide contraseña:

![image password prompt](images/pasted-20251102162820.png)

---

## 5) Directorio `/secret` y la clave

Visitamos `http://IP/secret` y vemos que contiene una clave:

![secret page](images/pasted-20251102163051.png)

Podemos usar la lista de contraseñas obtenida en `/uploads/` para intentar descifrar la clave.

---

## 6) Preparar la `private key` y usar `john`

Creamos un archivo llamado `id_rsa` que contiene la clave privada encontrada.

Usamos `ssh2john.py` (u otro script) para convertir la llave a un formato que John pueda leer:

![ssh2john](images/pasted-20251102164759.png)

Esto genera la llave hasheada:

![id_rsa hash](images/pasted-20251102164846.png)

Luego usamos `john` para intentar crackear la contraseña usando la lista extraída de `/uploads/`:

![john cracking](images/pasted-20251102164943.png)

Al principio recibimos un error porque `id_rsa` no tenía permisos correctos. Cambiamos los permisos:

```bash
chmod 600 id_rsa

## 7) Conexión SSH, enumeración y descubrimiento del grupo `lxd`

Tras ajustar permisos y usar la clave (id_rsa) con su contraseña `letmein`, iniciamos sesión como **john**:

![login john](images/pasted-20251102165233.png)

Dentro del home de john obtenemos la primera flag:

![first flag](images/pasted-20251102165324.png)

Al revisar los grupos a los que pertenece el usuario vemos varios, entre ellos `lxd`:

![groups john](images/pasted-20251102165425.png)

---

## 8) Escalada a root aprovechando LXD

Investigué LXD y confirmé que estar en el grupo `lxd` permite crear y gestionar contenedores —si se abusa de eso, puede conducir a una escalada de privilegios. Decidimos crear un contenedor malicioso para obtener acceso completo a la máquina.

Primero descargamos la herramienta / payload que usaremos para la imagen del contenedor:

![download tool](images/pasted-20251102173328.png)

Clonamos / revisamos la rama en GitHub (según la guía usada):

![github clone](images/pasted-20251102173505.png)  
![github repo](images/pasted-20251102173601.png)

Abrimos un servidor HTTP temporal en nuestra máquina atacante y descargamos el payload desde la víctima:

![http server download](images/pasted-20251102173731.png)

Registramos el archivo como una nueva imagen en LXD llamada `myimage`:

![lxd image import](images/pasted-20251102173831.png)

Creamos un nuevo contenedor `ignite` usando `myimage`:

![create container](images/pasted-20251102174038.png)

Configuramos el contenedor como **privilegiado** (flag `security.privileged=true`) para que el root dentro del contenedor tenga privilegios equivalentes a root fuera:

![set privileged](images/pasted-20251102174350.png)

El paso clave (el exploit): montamos el disco raíz de la víctima (`source=/`) dentro del contenedor en `/mnt/root` para acceder al sistema de archivos host desde dentro del contenedor.

Encendemos el contenedor:

![start container](images/pasted-20251102174806.png)

Ejecutamos `/bin/sh` dentro del contenedor para obtener una shell interactiva:

![exec sh](images/pasted-20251102174916.png)

Ahora tenemos una shell dentro del contenedor:

![container shell](images/pasted-20251102175149.png)

Desde ahí elevamos privilegios a **root** (acceso al filesystem montado) y obtenemos la flag final:

![root flag](images/pasted-20251102175221.png)

> La flag se encuentra en `/mnt/root/` porque montamos el disco raíz de la máquina víctima en ese punto dentro del contenedor.
