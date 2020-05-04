# OpenWebinars_Replica_BD_MariaDB_Docker

Este repositorio contiene los comando utilizados durante el taller **Réplica de Base de BD con MariaDB y Docker**.

Cada uno de los comandos viene acompañado por una pequeña explicación y el objetivo es que se puedan copiar y pegar en vuestra consola mientras estáis desarrollando el taller.

#### Listado de imágenes disponibles

```sh
docker images
```


#### Descarga la imagen de MariaDB desde DockerHub

```sh
# Descargar la última versión (latest)
docker pull mariadb

# Descargar una versión determinada (10.1)
docker pull mariadb:10.1
```

Si no especificamos una versión siempre obtendremos la última versión disponible en [DockerHub](https://hub.docker.com/).


#### Comandos para la gestión de redes-Docker

```sh

#Listado de redes disponibles
docker network ls

#Borrado de una red con nombre red_replica
docker network rm red_replica

#Creación de una red Docker llamada red_replcia con las características por defecto.
docker network create --subnet=172.25.0.0/16 red_replica

#Ver de manera detallada la información de una red en docker (red_replica)
docker network inspect red_replica

```

#### Comandos para arrancar los contenedores maestro y esclavo

```sh
#Arrancar el servidor maestro
docker run -it -d --net red_replica --hostname maestro --add-host esclavo:172.25.0.3 --ip 172.25.0.2 --name maestro -p 3336:3306 -e MYSQL_ROOT_PASSWORD=maestro mariadb

#Arrancar el servidor esclavo
docker run -it -d --net red_replica --hostname esclavo --add-host maestro:172.25.0.2 --ip 172.25.0.3 --name esclavo -p 3346:3306 -e MYSQL_ROOT_PASSWORD=esclavo mariadb

```

A continuación vamos a describir brevemente el significado de todos los parámetros de _docker run_

* _**-it**_ arranca el contenedor con la entrada abierta para poder interactuar con él mediante el terminal.
* _**-d**_  arranca el contenedor en modo _dettached_. Se usa para contenedores que tienen un servicio en ejecución para que al arrancarlos desde el terminal no se bloquee el terminal y nos vuelva a salir el prompt de sistema.
* _**--net**_ conectar el contenedor a una determinada red docker (red_replica en este caso).
* _**--hostname**_ para nombre al host que estamos arrancando (maesto o esclavo).
* _**--add-host**_ para añadir resolución de nombres estática al contenedor que estamos arranchando. Dos parámetros separados por  : , la ip y el nombre asociado a esa ip.
* _**--ip**_ para arranchar el contenedor con una ip asignada manualmente. Tiene que ser de la red que le hemos asignado.
* _**--name**_ por si queremos darle un nombre concreto al contenedor en ejecución (maestro o esclavo). Si no le asignamos nada docker le da un nombre de manera aleatoria.
* _**-p**_ para la redirección de puertos. Dos parámetros separados por _:_ . El primero indica el puerto local y el segundo el puerto del contenedor,
* _**-e**_ para fijar una variable de entorno que se usará para la configuración del contenedor. En nuestro caso estamos fijando la contraseña del administrador de la base de datos (root) dando un valor a la variable de entorno MYSQL_ROOT_PASSWORD.


#### Listado de los contenedores en ejecución

```sh

# Muestra los contenedores en ejecución
docker ps

# Muestra todos los contenedores
docker ps -a

```

#### Inspección de un contenedor para conocer todas sus características

```sh
# Inspeccionar el maestro
docker inspect maestro

# Inspeccionar el esclavo
docker inspect esclavo

```

#### Conexión a la base de datos desde nuestra máquina

Tenemos dos formas posibles (las dos tienen el mismo efecto) , usando la ip que hemos dado al contenedor o bien usando la redirección de puertos que hemos establecido anteriormente.

```sh

# Usando la ip del contenedor. Conectarnos al maestro
mysql -u root -pmaestro -h 172.25.0.2 

# Usando la redirección de puertos. Conectar con el esclavo
mysql -u root -pesclavo -h 127.0.0.1 -P 3346
```

* _**-u**_ para especificar el usuario.
* _**-p**_ para especificar la contraseña.
* _**-h**_ para especificar el host donde se encuentra la base de datos.
* _**-P**_ para especificar el puerto en el que escucha la base de datos. Si no se indica nada será 3306.

#### Restaurar una copia de seguridad desde línea de comandos

Supondremos que el fichero de la copia de seguridad se llama backup.sql y está en el directorio desde el que ejecutamos la orden.

Hay que asegurarse de que el script lleva el __CREATE DATABASE__ o crear la base de datos destino de manera manual.

```sh

# Restauro en el maestro
mysql -u root -pmaestro -h 172.25.0.2 < backup.sql

```

#### Conectarse a un contenedor para ejecutar comandos en él

```sh

docker exec -it maestro /bin/bash

```

#### Contenido de los fichero de configuración del maestro y del esclavo

Configuración del maestro **/etc/mysql/my.cnf**

```sh
[mariadb]
log-bin
server-id=1
log-basename=maestro
```

Configuración del esclavo **/etc/mysql/my.cnf**

```sh
[mariadb]
server-id=2
```
#### Reinicio contenedores

```sh
docker restart maestro
docker restart esclavo
```

#### Configuración del servidor maestro

Debemos estar conectados al servidor maestro.

```sql

# Creamos el usuario replica con contraseña replica y con permiso para conectarse desde el escalvo
CREATE USER 'replica'@'172.25.0.3' IDENTFIED BY 'replica';

# Damos al usuario permiso para realizar réplicas de todas las bases de datos desde el servidor esclavo
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'172.25.0.3';

# Recargar los permisos de los usuarios
FLUSH PRIVILEGES;

# Bloqueo de las tablas para escritura
FLUSH TABLES WITH READ LOCK;

# Obtención de los datos del log que son necesarios para la sincronización
SHOW MASTER STATUS;

# Desbloque de las tablas
UNLOCK TABLES;

```

#### Configuración del servidor esclavo

Debemos estar conectados al servidor esclavo.

```sql

#Establezco los datos del maestro
CHANGE MASTER TO  MASTER_HOST='172.17.0.2',
MASTER_USER='replica',
MASTER_PASSWORD='replica',
MASTER_LOG_FILE='maestro1-bin.000003',
MASTER_LOG_POS=345;

# Inicio el esclavo, a partir de ahora si todo está correcto estará sincronizado con el maestro
start slave;

# Muestro el estado
show slave status;

```


Taller creado por @pekechis para @OpenWebinars
# Replica_BD_MariaDB_Docker
