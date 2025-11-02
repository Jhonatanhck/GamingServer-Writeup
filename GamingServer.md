
# Writeup: GamingServer

Empezamos con la el escaneo de puertos

![Nmap scan](images/Pasted%20image%2020251102154836.png)

Obtenemos estos resultados

![Nmap results](images/Pasted%20image%2020251102154913.png)

Agregamos la pagina al /etc/hosts

![Hosts edit](images/Pasted%20image%2020251102155326.png)

y accedemos 

![Web access](images/Pasted%20image%2020251102155344.png)

vemos los recursos de la pagina 

![Page resources](images/Pasted%20image%2020251102155541.png)

vemos un mensaje de un usuario llamado john, el nombre tal vez nos sirva despues si queremos logearnos en algun lado

se me ocurre hacer un ataque de fuerza bruta de directorios 

![ffuf run](images/Pasted%20image%2020251102160929.png)
 
`ffuf` probó 9,228 nombres de la lista y te mostró solo los que obtuvieron una respuesta interesante

# Explicacion del comando:

### `ffuf`

- **Qué es:** Es el nombre del programa que estás ejecutando: **F**uzz **F**aster **U** **F**ool.
    
- **Propósito:** Le dice a tu terminal que "ejecute la aplicación `ffuf`".


### `-u http://10.10.195.192/FUZZ`


- **`-u` (URL):** Es un "flag" o interruptor que le dice a `ffuf`, "lo que viene a continuación es la URL que quiero atacar".
    
- **`http://10.10.195.192/`**: Esta es la URL base de tu objetivo.
    
- **`FUZZ`**: Esta es la parte más importante. Es una palabra clave especial que `ffuf` reconoce. Significa: "Aquí es donde quiero que inyectes las palabras". `ffuf` tomará cada línea de tu lista de palabras (`-w`) y la pondrá en el lugar de `FUZZ`.
    
    - Intento 1: `http://10.10.195.192/`**`index.html`**
        
    - Intento 2: `http://10.10.195.192/`**`robots.txt`**
        
    - Intento 3: `http://10.10.195.192/`**`secret`**
        
    - ...etc.

### `-w /usr/share/wordlists/dirb/common.txt`

- **`-w` (Wordlist):** Es otro "flag" que le dice a `ffuf`, "lo que viene a continuación es la ruta al archivo de texto que quiero usar como mi diccionario de adivinanzas".
    
- **`/usr/share/wordlists/dirb/common.txt`**: Esta es la ruta completa en tu sistema de archivos (probablemente Kali Linux) que apunta a un archivo de texto muy popular (`common.txt`) que viene con la herramienta `dirb`. Contiene una lista de miles de nombres comunes de archivos y directorios (`admin`, `login`, `uploads`, `config`, etc.).


### `-e .txt`

- **`-e` (Extension):** Es un "flag" que significa "extensión".
    
- **`.txt`**: Le dice a `ffuf`: "Por cada palabra en mi _wordlist_, quiero que la pruebes dos veces:
    
    1. Una vez tal como está (ej. `secret`).
        
    2. Y una segunda vez añadiendo esta extensión (ej. `secret.txt`)".
        
    
    _Es por eso que en tus resultados aparecieron `.htaccess` (el intento normal) y también `.htaccess.txt` (el intento con la extensión `-e`)._

Continuando con el ataque 

![ffuf results](images/Pasted%20image%2020251102161407.png)

vemos que el comando nos devolvio directorios ocultos que antes no veiamos 

accedemos a /robots.txt y vemos esto

![robots.txt](images/Pasted%20image%2020251102161731.png)

Cuando accedemos a /uploads/ vemos esto

![uploads listing](images/Pasted%20image%2020251102161858.png)

obtenemos 3 archivos 

Una lista de contrasenas

![password list](images/Pasted%20image%2020251102161943.png)  

Una especie de carta 

![note file](images/Pasted%20image%2020251102162016.png)

Y un MEME xd

![meme](images/Pasted%20image%2020251102162051.png)

Cuando queremos extraer datos de la imagen vemos que nos pide contrasena

![image password prompt](images/Pasted%20image%2020251102162820.png)

ahora vamos al directorio http://IP/secret 

![secret page](images/Pasted%20image%2020251102163051.png)

y vemos que guarda una clave, la podemos usar haciendo fuerza bruta con la lista de contrasenas que obtuvimos anterior mente

ojo a estos pasos que me pusieron a pensar y a investigar varias cosas

Nos creamos un archivo con la private key yo lo llame "id_rsa" que contenga la clave de arriba 

ahora utilizamos un script de la herramienta john

![ssh2john](images/Pasted%20image%2020251102164759.png)

esto lo que hace es que convierte nuestro archivo id_rsa a un idioma que john lo pueda leer como lo es hash

![id_rsa hash](images/Pasted%20image%2020251102164846.png)

esta es la llave hasheada 

ahora el siguiente paso es volver a utilizar la herramienta john para crackear la contrasena 

![john cracking](images/Pasted%20image%2020251102164943.png)

le indicamos el archivo hasheado y -w para la lista de contrasenas que queremos que prueba, en nuestro caso fue la que extraimos de /uploads/

![john wordlist](images/Pasted%20image%2020251102165042.png)

aqui nos da un error pero ya tenemos la contrasena que es letmein 

el error es debido a que no le dimos los permios requeridos a id_rsa que es la clave con la que nos conectaremos a john que fue el usuario que vimos anterior mente

para solucionarlo solo tenemos que hacer esto chmod 600 id_rsa

ya con eso podremos ejecutar el comando

![ssh login](images/Pasted%20image%2020251102165233.png)

ya estamos dentro de john utilizando la llave y su contrasena letmein

aqui tenemos la primera flag

![first flag](images/Pasted%20image%2020251102165324.png)

vemos que estamos en muchos grupos interesante 

![groups](images/Pasted%20image%2020251102165425.png)

investigando un poco descubri que lxd es un proceso que puede ser corrido como root

despues de investigar mucho y preguntarle a chatgpt y a gemini muchas cosas

descubri que estar en el grupo lxd te da casi permisos de ser root porque te da todos los permisos de crear contenedores entonces vamos a crear un contenedor malicioso que nos de acceso a toda la maquina 

![download tool](images/Pasted%20image%2020251102173328.png)

Descargamos la herramienta que es lo primero que tenemos que haces para crear nuestro contenedor

Entramos a la rama de github para crear el contenedor como nos dice en la pagina de github

![github clone](images/Pasted%20image%2020251102173505.png)

![github repo](images/Pasted%20image%2020251102173601.png)

ahora abrimos un servidor temporal para llevarnos el archivo a la maquina victima, esto hace que nuestra maquina atacante se convierta en un servidor de descargas temporal

nos descargamos el archivo desde el servidor que abrimos

![http server download](images/Pasted%20image%2020251102173731.png)

aqui le dijimos a lxd que tenga ese archivo y registralo como una nueva imagen de contenedor y registralo con el nombre myimage

![lxd image import](images/Pasted%20image%2020251102173831.png)

aqui cree un nuevo contenedor llamado ignite usando myimage

![create container](images/Pasted%20image%2020251102174038.png)

aqui le decimos que -c security.privileged=true o sea quiero que este contenedor se ejecute en modo privilegiado esto significa que el usuario root dentro del contenedor sera tratado com el usuario root fuera del contenedor 

![set privileged](images/Pasted%20image%2020251102174350.png)

aqui cambiamos la configuracion del contenedor ignite (que estaba apagado todavia)

![container config](images/Pasted%20image%2020251102174350.png)

y el TRUCO es este le dijimos a lxc toma el disco duro raiz "souce=/" de la maquina victima y montalo dentro del contenedor ignite en la carpeta /mnt/root

aqui encendemos el contenedor

![start container](images/Pasted%20image%2020251102174806.png)

el contenedor ahora esta corriendo con el disco duro completo de la maquina victima montado en su interior

aqui ejecutamos el comando /bin/sh dentro del contenedor

![exec sh](images/Pasted%20image%2020251102174916.png)

esto nos dio una terminal shell dentro del contenedor 

![container shell](images/Pasted%20image%2020251102175149.png)
y aqui nos convertimos en root

![root flag](images/Pasted%20image%2020251102175221.png)
aqui esta la flag

sabiamos que estaba en /mnt/root/ porque fue el directorio que le pasamos anteriormente
