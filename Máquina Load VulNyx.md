Una vez tengamos la máquina descargada y funcionando, lo que tenemos que hacer es, desde la máquina atacante, ver que IP tiene la máquina víctima.

![[Pasted image 20240112165424.png]]

La herramienta que hemos usado se llama "arp-scan". Esta herramienta reporta las direcciones IP de la red a la que esté conectada. El comando se compone del nombre de la herramienta, el parámetro "-I", en el cuál indicamos la interfaz que está conectada a la red y "--localnet" para que analice la red a la que está conectada la máquina. Como se puede ver, la IP 192.168.1.118 es la máquina víctima, esto se sabe por la dirección MAC, ya que empieza por 08:00...., puesto que todas las máquinas de virtual box tienen la misma MAC.

Lo siguiente que tenemos que hacer es verificar de que sistema operativo se trata, para ello, enviamos un ping a la máquina víctima y nos fijamos en el valor de la "ttl". Si la "ttl" es un valor próximo a 64, se trata de una máquina Linux, y si es próximo a 125, se trata de una máquina Windows.

![[Pasted image 20240112170025.png]]

Como podemos ver, el valor de la ttl es de 64, con lo cuál, estamos ante una máquina Linux.

Ahora, tenemos que ver que puertos de la máquina están abiertos, para ello, haremos uso de la herramienta "Nmap".

![[Pasted image 20240112170424.png]]

El comando de nmap está dividido en varios parámetros, el primero "-p-" sirve para abarcar todo el rango de puertos abiertos, "-sS" sirve para que el escaneo vaya más rápido pero a la vez sea sutil, "--open" es para mostrar puertos que estén abiertos, "-sC" es para hacer uso de scripts que tiene nmap para realizar el escaneo, "-sV" sirve para ver la versión de cada servicio que corre en los puertos que estén abiertos, "--min-rate" para que vaya aún más rápido, "-n" para que no haga resolución DNS (esto lleva mucho tiempo), "-vvv" para que vaya mostrando por pantalla todo lo que se vaya encontrando en el escaneo y "-Pn" para no utilizar el comando "ping" en el reconocimiento.

![[Pasted image 20240112171210.png]]

Como podemos ver, es escáner ha encontrado que el puerto 22 (ssh) y el puerto 80 (http) están abiertos, con lo cuál, al tener el puerto 80 abierto significa que la máquina aloja una página web. Accedamos a ella.

![[Pasted image 20240112171402.png]]

Al acceder a la IP de la máquina, nos encontramos con la página por defecto de Apache. Hasta aquí todo normal, pero, esta web puede contener directorios/archivos ocultos, ¡vamos a hacer Fuzzing! El término "Fuzzing" hace referencia a hacer una búsqueda de los directorios que puede tener ocultos una página web, esto se realiza con la herramienta "gobuster".

![[Pasted image 20240112171919.png]]

Este comando se compone de varios parámetros, "dir" es para indicar que queremos obtener posibles directorios de la web, "-u" para indicar la URL a la que queremos atacar y "-w" para usar diccionarios que ya vienen preparados para esto con los directorios más comunes.

![[Pasted image 20240112172357.png]]

Aparentemente no ha encontrado ningún posible directorio pero, si volvemos a ver el escaneo, veremos un archivo llamado "robots.txt" y un directorio llamado "/ritedev".  Sabiendo esto, volveremos a ejecutar el comando anterior, pero esta vez, añadiendo a la URL el directorio mencionado.

![[Pasted image 20240112174306.png]]

El resultado de la búsqueda es el siguiente:

![[Pasted image 20240112174400.png]]

La búsqueda nos ha reportado algo que nos puede interesar, el archivo "admin.php". Accedamos a él "https://192.168.1118/ritedev/admin.php".

![[Pasted image 20240112174528.png]]

Al acceder a la URL nos lleva a un panel de login pero, tenemos un problema, desconocemos el usuario y la contraseña. En este caso, usaré un poco la intuición y probaré con admin:admin.

![[Pasted image 20240112174742.png]]

¡Funcionó! Como podemos observar, vemos los ajustes de administración del cms. De todos estos apartados hay uno que me llama bastante la atención, "Files Manager".

![[Pasted image 20240112175049.png]]

Si entramos dentro, vemos que podemos subir archivos, lo cual nos da la posibilidad de subir un archivo malicioso con un código para ejecutar una reverse shell. Vamos a buscar un exploit que nos permita alcanzar el objetivo. 

![[Pasted image 20240112222653.png]]

Este es el exploit que usaremos para poder recibir la reverse shell. Para hacerlo, debemos de capturar la petición de subida del archivo, pegar el código anterior y mandar la petición.

![[Pasted image 20240112223017.png]]

Una vez capturada la petición, la mandamos al Repeater y sustituimos estas líneas por las del exploit.

![[Pasted image 20240112223054.png]]
![[Pasted image 20240112223139.png]]

Como podemos ver, el archivo se ha subido correctamente.

![[Pasted image 20240112223746.png]]

Como se puede ver, ha funcionado, ya tenemos el archivo con código malicioso dentro de la máquina. Después de esto, tenemos que decodificar el binario en base 64 que tenemos en el archivo malicioso que acabamos de subir.

![[Pasted image 20240112224120.png]]

Después de esto, vemos si podemos ejecutar comandos de manera remota.

![[Pasted image 20240112225006.png]]

Excelente, ya podemos enviar a nuestra máquina atacante una reverse shell. Lo primero que tenemos que hacer es ponernos en escucha por un puerto que nosotros queramos, en mi caso he usado el puerto 1234. La herramienta que usaremos será "NetCat".

![[Pasted image 20240112225502.png]]

Lo siguiente que tenemos que hacer es codificar el código que vamos a ejecutar para obtener la reverse shell.

![[Pasted image 20240112225651.png]]
![[Pasted image 20240112225722.png]]

Una vez ejecutado esto, volvemos al terminal donde hemos ejecutado el comando de NetCat y podemos comprobar que tenemos acceso remoto a la máquina.

![[Pasted image 20240112225832.png]]

Si tratamos de ejecutar algún comando, nos dará error, con lo cuál, tenemos que hacer el tratamiento de la tty.

![[Pasted image 20240113000948.png]]

Hecho el tratamiento de la tty, comprobamos si podemos elevar nuestros privilegios.

![[Pasted image 20240112230504.png]]

Vemos que hemos encontrado un usuario llamado "travis" y lo podemos hacer con crash. Crash es una herramienta de interacción  de análisis de estado de Linux mientras esté corriendo.

![[Pasted image 20240112230930.png]]

Al final de la ejecución de este comando, veremos que abajo a la izquierda aparecen dos puntos, esto hay que cambiarlo por "!sh" para poder realizar la reverse shell.

![[Pasted image 20240112231231.png]]

Una vez hecho esto, navegamos hacia el directorio home del usuario "travis", allí nos encontraremos un fichero llamado "user.txt" el cual contiene la flag.

![[Pasted image 20240112231430.png]]
![[Pasted image 20240112231444.png]]

Una vez obtenida esta flag, debemos de conseguir la del usuario "root", para ello, veremos si podemos seguir elevando nuestros privilegios.

![[Pasted image 20240112231637.png]]

Como podemos observar, sí podemos seguir escalando privilegios hasta convertirnos en root. Lo podemos hacer mediante "xauth". Xauth es una herramienta que permite visualizar un entorno gráfico.

Como podemos ver, no podemos visualizar el entorno ya que no disponemos del archivo .Xauthority.

![[Pasted image 20240112232205.png]]

Si volvemos a fijarnos en el escaneo, vemos que el puerto 22 está abierto, vamos a tirar por aquí. Intentemos iniciar sesión como root en el servicio ssh.

![[Pasted image 20240112232516.png]]

No podemos iniciar sesión, pero, si nos fijamos, habla de la clave pública, con lo cual, vamos a capturar las claves públicas y privadas del usuario root.

![[Pasted image 20240112232752.png]]

Vemos que el archivo que contiene las claves públicas no existe, probemos si existe el archivo que contiene las claves privadas.

![[Pasted image 20240112232957.png]]

Ya tenemos la clave del usuario root, ahora lo que tenemos que hacer es limpiarla y añadir "RSA PRIVATE KEY".

![[Pasted image 20240113002841.png]]

Habiendo quedado lista la clave privada, editamos el archivo id_rsa de la máquina víctima y pegamos todo.

![[Pasted image 20240113002918.png]]

Ya tenemos el archivo, démosle permisos al usuario para utilizar el archivo.

![[Pasted image 20240113001215.png]]

Ahora, iniciamos sesión como root en ssh utilizando la clave privada que acabamos de generar.

![[Pasted image 20240113003031.png]]

Una vez dentro de la máquina, listamos los archivos que se encuentran en el directorio "/root" y localizamos la última flag.

![[Pasted image 20240113003241.png]]






