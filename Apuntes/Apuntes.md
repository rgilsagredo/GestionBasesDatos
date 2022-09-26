# Apuntes GBD 1º ASIR

## MySQL Server en un contenedor de Docker:
Requisitos:
- Docker 
- Usuario con privilegios (?) **Veremos si esto es cierto**  

Estamos en OS *Windows 10* **PDTE ver versión, en mi personal es home version**

Abrimos la CMD, introduce `docker pull mysql/mysql-server:latest`. Esto hace que Docker
se descargue la última imagen de MySQL server. Si deseas una vaesión concreta, cambia *latest*
por el número de la versión.  

Chequeamos con `docker images` que se ha descargado.  

La activamos con `docker run -p 3306:3306 --name=[container_name] -d mysql/mysql-server:latest`  

**PDTE** entender mejor Docker. La opción `-p 3306:3306` mapea los puertos del contenedor y de la máquina host,
el puerto 3306 es el default del server. `--name` da un nombre al contenedor de Docker, yo lo llamaré *mysql-server*.
`-d` hace que el contenedor funcione como un server en el background, así simulamos que MySQL server es un server.  

Para ver que el servicio está, ejecutamos `docker ps`. Debería aparecer listado. También podemos ir mirando la
GUI de Docker Desktop y usar su CLI.  

Una vez que el servicio está corriendocomo contenedor, desde la CMD ejecutamos `docker logs [container_name]` (también podemos mirar Docker).
Buscamos el password que ha generado para el user *root* (pondrá, en algún sitio de la info, Entrypoint GENERATED ROOT PASSWORD).
Guardamos ese password. Lo siguientepodemos hacerlo via CMD o con la CLI de Docker. Si vamos por CMD, ejecutamos
`docker exec -it [container_name] bash`, y nos abre un terminal bash, si vamos por la CLI de Docker, estamos
directamente en el bash. Una vez en el bash, ejecutamos `mysql -uroot -p`. Nos pedirá el password que
hemos guardado antes. Tras introducirlo, estaremos conectados al server logeados como 'root', que es el user por defecto.
Tiene todos los privilegios, no queremos trabajar en este usuario en general.  

Lo primero que hay que hacer es cambiar el password de root, para ellos, desde el command line del client
ejecutamos `ALTER USER 'root'@'localhost' IDENTIFIED BY [password];`, y metemos el password que queramos. 
**NOTA: para el curso voy a usar como password el mismo nombre que el user. No hace falta explicar por qué esto es una malísima idea en un ambiante laboral**
**NOTA: lo que estamos haciendo son tareas administrativas que se enseñan en la asignatura de 2ASIR, pero ir cogiendo buenas prácticas siempre es bueno**  

Para ver que todo está bien, ejecutamos, ya como cliente del server, `SELECT user, host FROM mysql.user`. Debe devolvernos una
tabla con los users y host desde donde se pueden conectar.  

Queremos usar *MySQL Workbench* para trabajar con el server, ya que es mas amable que ir por linea de comandos (aunque no nos quitaremos de hacer
cosas por CLI), así que necesitamos un poco más de configuración. Es debido a que el server está en un contenedor de Docker, no en la máquina en 
la que estamos trabajando. El resultado de la consulta anterior, nos dice que *root* sol se puede conectar al server si estamos en localhost, pero
no permitirá conecxiones como *root* desde otra máquina. Hay que cambiar eso, ya que a ojos del server, la máquina que tiene el contenedor es 
otra máquina.  

**PDTE: necesito entender mejor cómo funciona Docker para no hacer lo que vamos a hacer ahora, ie, hacerlo más *limpio***  

Para poder conectarnos al server desde Workbench, necesitamos decirle al server que *root* se puede conectar desde cualquier máquina.
Para ello, cambiamos el *host* del user *root* a *%*, que quiere decir *cualquier máquina*. Para ello, hacemos
`UPDATE mysql.user SET host='%' WHERE user='root';`. Si volvemos a ejecutar `SELECT user, host FROM mysql.user` 
veremos el cambio. Salimos del client (con `exit`) y reseteamos el contenedor, desde la CMD, con `docker restart [container_name]`.  

**Pequeño desvío**: si en algún momento tuviésemos que volver a empezar por hacer algo irreparable, tenemos los comandos (desde CMD)
`docker stop [container_name]` y `docker rm [container_name]` para parar y borrar por completo en contenedor reps.  

Con esto ya deberíamos ser capaces de conectarnos desde Workbench al server. Abrimos Workbench y pulsamos en el '+' para generar
una nueva conexión (aunque las que hay por defecto debería funcionar también). Los parámetros para la conexión son:
- Method: Standard TCP/IP
- Hostname: localhost o 127.0.0.1
- Port: 3306

Arriba debemos darle un nombre a la conexión, lo llamaré *root-docker*. Testeamos conexión, y debería dar mensaje de ok. Si no, 
hemos hecho algo mal. Supongo que lo más sensato es elimniar el contenedor y volver a repatir pasos.  

Vamos a comprobar que todo va bien, para ellos, nos conectamos desde Workbench al server. En la tab de *administration* vemos que 
*server status* es *running* (tiene que coincidir con el estado del Docker container). Probamos a pararlo, vamos a 
CMD y ejecutamos `docker stop [container_name]`. Esto lo para, y se refleja en Workbench. Lo 
volvemos a poner en marche desde CMD con `docker start [container_name]`, vemos que se refleja en Workbench.  

Probamos a crear una DB/tabla e introducir datos, tanto desde Workbench como desde el client de Docker, tenemos que ver
que todo funciona correctamente. Empezamos en Workbench, para ello, creamos una nueva tab de ejecución de QUERIES
(es el botón de la izquierda de la barra de herramientas) e introducimos
		CREATE DATABASE IF NOT EXISTS test;
		USE test;
		CREATE TABLE IF NOT EXISTS test(
			col1 INT,
			col2 VARCHAR(255)
		);
		INSERT INTO test(col1, col2) VALUES (1, "uno"), (2, "dos"), (3, "tres");
		

Deberíamos (tras pulsar el botón de refresh) ver que se ha creado una DB, con una tabla, con datos. Podemos hacer `SELECT * FROM test.test;`
para ver que es así. Por otro lado, si vamos al CLI de Docker, podemos ver que efectivamente estos cambios se reflejan en el server
ejecutando `SHOW DATABASES;` y `USE test;` `SELECT * FROM test;`.  

Se puede comprobar que el camino 'al revés' funciona, ie, crear DB en CLI de Docker, crear tabla, meter datos, y lo vemos reflejado en Workbench 
(no puede ser de otra manera pues desde Workbench estamos conectados, como user *root*, al servidor).  

Para terminar de comprobar que todo funciona bien, dado que tenemos "trabajo guardado" en el server (hemos creado una DB, una tabla en esa DB, y hemos
introducido datos), estaría bien comprobar que el Docker container mantiene el estado del server incluso si lo paramos. Para ello, desde la CMD
ejecutamos `docker stop [container_name]`. Eso para el contenedor (el server ya no está corriendo). Si lo reactivamos con `docker start [container_name]`
vemos que no hemos perdido nada del "trabajo hecho".  

Lo siguiente será aprender a hacer usuarios (desde el CLI de Docker) y darles los permisos necesarios para trabajar. En la asignatura de primero (GBD) esto no
es necesario saberlo pero hacer las cosas bien es gratis, y tampoco es tanto esfuerzo. Esto lo hacemos porque *root* es el user por defecto que crea el server
y tiene muchos más permisos de los que vamos a necesitar, y no queremos romper nada.  

Antes de hacer los usuarios, vamos a crear BBDD con las que vamos a ir trabajando, y después nos crearemos un user con (todos) los privilegios sobre esas BBDD.  

Cada BBDD la vamos a crear usando un *backup lógico*, esto es, un fichero .sql con las sentencias que crean la BD, sus tablas, y los datos de las tablas. Simplemente hay
que ejecutar desde Workbench los ficheros .sql (estarán en el repo).  

Ahora creamos un usuario para trabajar en las BBDD que hemos creado desde *root*. Para crear un usuario, vamos al CLI de Docker 
y ejecutamos `CREATE USER 'user_name'@'host_id' IDENTIFIED BY 'password'`
donde pondré como user_name y password *raul*, y host_id será '%' para que nos permita conectarnos desde cualquier host.  
Si todo ha ido bien, podremos hacer una conexión desde Workbench al server con este usuario. El user creado no tiene ningún privilegio, vamos a 
concederle todos sobre las DDBB que hemos creado antes para poder empezar a trabajar. Para ello, desde el CLI de Docker, ejecutamos 
`GRANT ALL PRIVILEGES ON db_name.* TO 'user_name'@'host_name';`, especificando los parámetros para cada DB. El user y host siempre serán los mismos.  

Con esto estamos dando al usuario poderes administrativos sobre las DDBB; no es lo ideal, a nivel de seguridad, pero para este curso nos vale.  

Si todo ha ido bien, podremos ver las DDBB desde la conexión del usuario que hemos creado, y podremos ver también privilegios desde Workbench. 
A partir de este punto, nos olvidamos de momento del usuario *root*, y vamos a trabajar siempre con el que hemos creado.



## Biblio

### Más general
- Documentación oficial de MySQL: https://dev.mysql.com/doc/
- MySQL Tutorial: https://www.mysqltutorial.org/
- W3Schools Tutorial: https://www.w3schools.com/mysql/
- Material: https://josejuansanchez.org/bd/

### Más específica
- Instalar mysql-server en un contenedr de Docker: https://hevodata.com/learn/docker-mysql/
- Conectar mysql workbench con el contenedor: https://stackoverflow.com/questions/33827342/how-to-connect-mysql-workbench-to-running-mysql-inside-docker
- Admin Best Practices: http://download.nust.na/pub6/mysql/tech-resources/articles/mysql-administrator-best-practices.html
