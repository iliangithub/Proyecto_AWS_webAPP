# Proyecto_AWS_2

multi tier web application stack [vprofile]
host and run for production
lift and shift strategy

tenemos servicios de app que está corriendo o son ejecutados en maquinas virtuales o físicas, servicios de base de datos; Postgre, Oracle, aplicaciones como; TomCat, LAMP stack y servicios DNS.

Tenemos este Workload, todos eçtodos estos servidores corriendo de forma local, almacenar todo de forma local pues supone costes y no está automatizado, perdemos tiempo. Es decir, pagamos por la infraestructura como servicio, realmente pagamos del AWS pues la infraestructura, los procesadores, la RAM. IaaS. Es escalable, reducir o aumentar para reducir costes y rendimiento o aumentar el rendimiento pero también aumentar los costes y también podemos autimatizar.

Utilizamos evidengemente AWS Cloud Computing.
- Utilizaremos las instancias EC2, que serán nuestras máquinas virtuales, para nuestro TomCat, RabbitMQ, MemCache y MySQL.
- ELB para hacer un balanceador de carga, es decir, remplazaremos el Nginx como nuestro balanceador de cargas, ya no lo vamos a utilizar.
- El servicio de autoescalado, que automáticamente escalará o reducirá, según nuestras necesidades, que controlará nuestros recursos y nuestros costes.
- Y como almacenamiento, utilizaré el S3 y el EFS
- Por último, Route 53, para nuestro servicio DNS privado.
- También otros servicios como IAM, ACM o EBS.

entonces, queremos que sea flexible, escalable, e ir pagando de poco a poco por lo que vamos utilizando. Y también queremos automatización de la infra, IaaC.

https://www.udemy.com/course/decodingdevops/learn/lecture/26464654#overview
en este explican los puertos y tal...
# El diseño arquitectura:
EC2 instances
Elb
Auto scaling
efs s3 para el almacenamiento
amazon certificate mana
route 53

el diseño es ahora:
Nuestros usuarios, van a acceder al sitio web, utilizando una URL y esa URL, va a apuntar al balanceador de carga, utilizaremos, https, y habrá un certificado que será respaldado por el ACM de amazon (Amazon Certificate Manager).
El balanceador de carga, va a estar en un Grupo de Seguridad (Firewall) y solo aceptará peticiones https, solo tráfico https.
Entonces, el balanceador carga, mandará a la instancia TomCat, este TOmCat, en realidad, tendrá varias instancias, y serán pues manejadas por el grupo de autoescalado. Entonces, dependiendo del tráfico y las peticiones y recursos, pues escalaremos o reduciremos. Estas instancias, pues estarán en otro grupo de de seguridad y solo aceptarán tráfico en el puerto 8080 y solo del balanceador de carga, ahora.

Necesitamos el backend, el MySQL, MemCache y RabbitMQ, las IP de estos servidores del backend, o los servicios, van estar pues en Route 53, en el DNS privado.

Lo primero que vamos a hacer, es crear los pares-clave, crear los grupos de seguridad, y vamos a iniciar las instancias con sus respectivos scripts, para que se inicien y configuren.
Vamos a actualizar la IP al name mapping en Route 53.Y finalmente, vamos a crear nuestra aplicación en código fuente, esto en nuestra máquina local (Terraform).
Y finalmente vamos a subir neustro artefacto a S3 bucket. Desde S3, vamos a descargarlo a la instancia EC2.
Luego, el ELB con HTTPS el certificado. EL balanceador de carga, irá pues a la página.

# Grupos de Seguridad y Pares-Clave.
## Grupos de Seguridad (Creación).
![image](https://github.com/user-attachments/assets/9e1fb253-1f6d-4a87-8c02-25030b642cff)
![image](https://github.com/user-attachments/assets/9a77f185-f499-4dfa-b4c4-a420e66af7f2)

Las de salida, como de costumbre no las vamos a tocar.

Ahora voy a crear las de la instancia TomCat:
![image](https://github.com/user-attachments/assets/654f48da-04dd-4e2f-a6b4-49987f44ca1d)

![image](https://github.com/user-attachments/assets/679b901a-31a4-48d4-94b2-a34206c58a09)

y solo falta un último grupo de seguridad, para las aplicciones del backend:
![image](https://github.com/user-attachments/assets/67d876f6-cdea-4320-b16e-afa90dd2928d)

**Y una vez creada, volvemos a editar esa misma**
Porque queremos añadir otra regla de entrada que admita el tráfico, de si mismo.

![image](https://github.com/user-attachments/assets/e1ef8053-d2a6-4bcb-8821-f6d7e6e2eff6)

Y también tendríamos que haber añadido el puerto 22 SSH, en el TomCat app:
![image](https://github.com/user-attachments/assets/65de6d2c-41e6-41d5-9d1f-962580bffb60)

Con el backend, exatamente igual:
![image](https://github.com/user-attachments/assets/378cfaee-e630-4e79-96e1-9691889dc147)

## Pares-Claves (Creación).
![image](https://github.com/user-attachments/assets/3c7ea6d4-b3e4-492b-ade8-9d3cc359d0c3)

![image](https://github.com/user-attachments/assets/fbe23bdb-11c3-4317-b52a-d6b26d6b41ab)

# Crear las instancias.
Vamos a clonar el source code

## Para la base de datos:

**Nombre y etiquetas:**
| Clave    | Valor |
| -------- | ------- |
| Name  | proyecto-db01 |
| Project | delta |

**Imagenes de Aplicaciones y Sistemas Operativos (AMI)**
Amazon Linux, 2023, 64 bits (x86)

**Tipo de instancia**
t2.micro

**Pares clave**
Las que creamos `proyecto-produccion-pares-clave`

**Configuraciones de red**
Seleccionamos un grupo de seguridad existente: `proyecto-backend-SG`

**Detalles Avanzados > Datos de usuario**
```
#!/bin/bash
DATABASE_PASS='admin123'
sudo yum update -y
#sudo yum install epel-release -y
sudo yum install git zip unzip -y
sudo dnf install mariadb105-server -y
# starting & enabling mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
cd /tmp/
git clone -b main https://github.com/hkhcoder/vprofile-project.git
#restore the dump file for the application
sudo mysqladmin -u root password "$DATABASE_PASS"
sudo mysql -u root -p"$DATABASE_PASS" -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '$DATABASE_PASS'"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-project/src/main/resources/db_backup.sql
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

sudo systemctl restart mariadb
```

## Para el memcached:

**Nombre y etiquetas:**

| Clave    | Valor |
| -------- | ------- |
| Name  | proyecto-mc01 |
| Project | delta |

**Imagenes de Aplicaciones y Sistemas Operativos (AMI)**
Amazon Linux, 2023, 64 bits (x86)

**Tipo de instancia**
t2.micro

**Pares clave**
Las que creamos `proyecto-produccion-pares-clave`

**Configuraciones de red**
Seleccionamos un grupo de seguridad existente: `proyecto-backend-SG`

**Detalles Avanzados > Datos de usuario**
```
#!/bin/bash
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached
sudo yum install firewalld -y
sudo systemctl start firewalld
sudo systemctl enable firewalld
firewall-cmd --add-port=11211/tcp
firewall-cmd --runtime-to-permanent
firewall-cmd --add-port=11111/udp
firewall-cmd --runtime-to-permanent
sudo memcached -p 11211 -U 11111 -u memcached -d
```

## Para el RabbitMQ:

**Nombre y etiquetas:**

| Clave    | Valor |
| -------- | ------- |
| Name  | proyecto-rmq01 |
| Project | delta |

**Imagenes de Aplicaciones y Sistemas Operativos (AMI)**
Amazon Linux, 2023, 64 bits (x86)

**Tipo de instancia**
t2.micro

**Pares clave**
Las que creamos `proyecto-produccion-pares-clave`

**Configuraciones de red**
Seleccionamos un grupo de seguridad existente: `proyecto-backend-SG`

**Detalles Avanzados > Datos de usuario**
```
#!/bin/bash
## primary RabbitMQ signing key
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc'
## modern Erlang repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key'
## RabbitMQ server repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key'
curl -o /etc/yum.repos.d/rabbitmq.repo https://raw.githubusercontent.com/hkhcoder/vprofile-project/aws-LiftAndShift/al2023rmq.repo
dnf update -y
## install these dependencies from standard OS repositories
dnf install socat logrotate -y
## install RabbitMQ and zero dependency Erlang
dnf install -y erlang rabbitmq-server
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo systemctl restart rabbitmq-server
```

## Para la TomCat-app01:

**Nombre y etiquetas:**

| Clave    | Valor |
| -------- | ------- |
| Name  | proyecto-app01 |
| Project | delta |

**Imagenes de Aplicaciones y Sistemas Operativos (AMI)**
Ubuntu Server 24.04 LTS, 64 bits (x86)

**Tipo de instancia**
t2.micro

**Pares clave**
Las que creamos `proyecto-produccion-pares-clave`

**Configuraciones de red**
Seleccionamos un grupo de seguridad existente: `proyecto-TomCat-APP-SG`

**Detalles Avanzados > Datos de usuario**
```
#!/bin/bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-11-jdk -y
#sudo apt install tomcat9 tomcat9-admin tomcat9-docs tomcat9-common git -y
# no funciona Package tomcat9 is not available, but is referred to by another package.
#This may mean that the package is missing, has been obsoleted, or
#is only available from another source

#este si:
sudo apt install tomcat10 tomcat10-admin tomcat10-docs tomcat10-common git -y

sudo systemctl start tomcat10
```
# Comprobaciones (podría fallar o haber errores).
## Máquina MySQL.
```
sudo systemctl status mariadb
```
Evidentemente, tiene que mostrar que está running.
```
mysql -u admin -padmin123 accounts
```

Y deberías de ver esto:
```
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 3
Server version: 10.5.25-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [accounts]>
```

```
SHOW tables;
```

Debería devolver esto:
| Tables_in_accounts |
| -------- |
| role  |
| user |
| user_role |

Para salir del MariaDB:

```
quit
```
A por la siguiente máquina.
## Máquina MemCache.

```
[ec2-user@ip-172-31-xx-xx ~]$ ss -tunlp | grep 11211
tcp   LISTEN 0      1024                           0.0.0.0:11211      0.0.0.0:*
tcp   LISTEN 0      1024                             [::1]:11211         [::]:*
```


## Máquina RabbitMQ-server.

```
[ec2-user@ip-172-31-38-8 ~]$ systemctl status rabbitmq-server
● rabbitmq-server.service - RabbitMQ broker
     Loaded: loaded (/usr/lib/systemd/system/rabbitmq-server.service; enabled; >
     Active: active (running) since Tue 2024-10-15 09:19:03 UTC; 37min ago
   Main PID: 26089 (beam.smp)
      Tasks: 26 (limit: 1112)
     Memory: 68.1M
        CPU: 5.748s
     CGroup: /system.slice/rabbitmq-server.service
             ├─26089 /usr/lib64/erlang/erts-14.2.5.4/bin/beam.smp -W w -MBas ag>
             ├─26102 erl_child_setup 32768
             ├─26117 sh -s disksup
             ├─26119 /usr/lib64/erlang/lib/os_mon-2.9.1/priv/bin/memsup
             ├─26120 /usr/lib64/erlang/lib/os_mon-2.9.1/priv/bin/cpu_sup
             ├─26121 /usr/lib64/erlang/erts-14.2.5.4/bin/inet_gethost 4
             ├─26122 /usr/lib64/erlang/erts-14.2.5.4/bin/inet_gethost 4
             ├─26133 /usr/lib64/erlang/erts-14.2.5.4/bin/epmd -daemon
             └─26152 /bin/sh -s rabbit_disk_monitor
```

## Máquina TomCat-app01.

```
systemctl status tomcat10
```

```
● tomcat10.service - Apache Tomcat 10 Web Application Server
     Loaded: loaded (/usr/lib/systemd/system/tomcat10.service; enabled; preset:>
     Active: active (running) since Tue 2024-10-15 10:10:20 UTC; 2min 26s ago
       Docs: https://tomcat.apache.org/tomcat-10.0-doc/index.html
    Process: 5223 ExecStartPre=/usr/libexec/tomcat10/tomcat-update-policy.sh (c>
   Main PID: 5228 (java)
      Tasks: 28 (limit: 1130)
     Memory: 104.9M (peak: 112.4M)
        CPU: 6.126s
     CGroup: /system.slice/tomcat10.service
             └─5228 /usr/lib/jvm/java-11-openjdk-amd64/bin/java -Djava.util.log>

Oct 15 10:10:46 ip-172-31-40-172 tomcat10[5228]: At least one JAR was scanned f>
Oct 15 10:10:46 ip-172-31-40-172 tomcat10[5228]: Deployment of deployment descr>
Oct 15 10:10:46 ip-172-31-40-172 tomcat10[5228]: Deploying deployment descripto>
Oct 15 10:10:46 ip-172-31-40-172 tomcat10[5228]: The path attribute with value >
Oct 15 10:10:46 ip-172-31-40-172 tomcat10[5228]: At least one JAR was scanned f>
Oct 15 10:10:46 ip-172-31-40-172 tomcat10[5228]: Deployment of deployment descr>
Oct 15 10:10:56 ip-172-31-40-172 tomcat10[5228]: Deploying deployment descripto>
Oct 15 10:10:56 ip-172-31-40-172 tomcat10[5228]: The path attribute with value >
Oct 15 10:10:57 ip-172-31-40-172 tomcat10[5228]: At least one JAR was scanned f>
Oct 15 10:10:57 ip-172-31-40-172 tomcat10[5228]: Deployment of deployment descr


LUEGO


ls /var/lib/tomcat10/
conf  lib  logs  policy  webapps  work


Y WEBAPPS ES DONDE VAN A ESTAR PUES LAS APPS WEB.
```
Este es nuestro archivo /etc/hosts:

![image](https://github.com/user-attachments/assets/ab69e7ee-5c3c-46e1-a8db-d5e086788791)

> [!IMPORTANT]
> El archivo /etc/hosts es útil en situaciones locales o para configuraciones muy específicas en servidores individuales, ya que permite mapear nombres de dominio a direcciones IP de forma local en la máquina. Sin embargo, no es una solución viable para entornos de producción o en la nube, especialmente en AWS.
>
> (Mapear = asociar un nombre de dominio a una dirección IP)
>
> ### ¿Por qué solo sirve de forma local el archivo /etc/hosts?
>El archivo /etc/hosts es un archivo de configuración presente en sistemas Unix y Linux, que permite hacer este mapeo de manera local en una máquina específica. Es local porque solo afecta esa máquina en particular; cualquier otro dispositivo (como el de un cliente externo) no verá ni usará los cambios que realices en ese archivo.
>
>**Cada máquina tiene su propio archivo /etc/hosts: Cada vez que una máquina necesita traducir un dominio a una IP, primero revisa su propio archivo /etc/hosts antes de hacer consultas a servidores DNS. Esto significa que cualquier cambio en este archivo solo será visible para esa máquina en particular.**
>
>No se comparte a nivel de red. **Los cambios que realices en el archivo /etc/hosts no se propagan ni se comunican a otras máquinas en la red o en internet**. Si otras máquinas o usuarios quieren resolver el mismo nombre de dominio, **deberán tener sus propios mapeos o depender de un servidor DNS.**
>
>Limitado a resoluciones locales: En redes pequeñas o entornos de desarrollo, el archivo /etc/hosts puede ser útil si necesitas probar algo rápidamente o hacer una configuración temporal. Sin embargo, en entornos de producción (como en AWS), donde muchos usuarios deben acceder a tu instancia o servicio, necesitas un sistema centralizado de resolución de nombres como Route 53, que es un servidor DNS.
>
>### ¿Por qué necesitas un servidor DNS como Route 53?
>Cuando quieres que tu dominio sea accesible para cualquier persona en internet (porque no tienen tu archivo /etc/hosts/), los navegadores y sistemas operativos de los usuarios no van a consultar tu archivo /etc/hosts. **En su lugar, consultan servidores DNS distribuidos por todo el mundo**. Route 53 es un servicio de DNS que permite gestionar estos mapeos de forma centralizada y global, permitiendo que cualquier persona pueda acceder a tu dominio, no solo una máquina local.
>

# Route 53 (DNS Server).

Así que buscamos en la consola de AWS, el servicio Route 53:

![image](https://github.com/user-attachments/assets/db0cf6e3-1750-410c-864b-f247e9fa31c7)

Tenemos que crear una zona, y allí estará nuestro nombre de dominio. Y en ese dominio, tendremos diferentes Hosts. Y esos registros de Hosts, tendrán la IP o el CNAME.

## Crear Zona.

> [!TIP]
> Una zona DNS es una porción del espacio de nombres DNS que se administra de manera independiente. Puede contener uno o varios registros DNS y puede abarcar un dominio completo o una subparte de él.
> 

![image](https://github.com/user-attachments/assets/871e91e6-5f06-42c9-9bc2-c58b1609ff2f)

Y vamos a ponerle nombre de dominio: `delta.es`

NO VA A SER RESUELTO DESDE EL INTERNET, DESDE FUERA.

![image](https://github.com/user-attachments/assets/4e5a95d6-624c-4c79-839f-2c17dd6ad2a7)

VPC, es la Red, de esa región:

![image](https://github.com/user-attachments/assets/117855f3-fe28-4a13-9ea0-a6ab74b5d3b5)

## Crear Registros.
Y ahora creamos un "record", un registro:

![image](https://github.com/user-attachments/assets/059b9a75-25a6-4a98-a072-56ef2dfa9c9b)

necesito saber la IP privada de mis instancias:
![image](https://github.com/user-attachments/assets/0ee843bd-d6c1-4ccc-ac48-623c4d03ba1c)

Y lo copio en el registro:
![image](https://github.com/user-attachments/assets/b5aa5726-0bc9-43e7-b163-05da26eee62f)

Y creamos el registro. Ahora hacemos lo mismo con el resto.

> [!TIP]
>
> En mi caso, la interfaz me aparece así:
>
>![image](https://github.com/user-attachments/assets/7b984d3b-f0b5-4241-a259-533d09f185c9)
>
> Si cambiamos el asistente, se verá de esta forma más compleja:
>
> ![image](https://github.com/user-attachments/assets/bafd79c1-4d98-4fb9-874f-0eff7e82aeae)
> 
> ![image](https://github.com/user-attachments/assets/addd5d5d-2045-4960-a8ed-dbf3f6763fbe)
> 
> Y le damos a definir un registro simple.
> 
> ![image](https://github.com/user-attachments/assets/24743759-e067-4bb4-b9b8-bac0188377f7)

Realmente, solamente necesitamos estos 3 registros, no más. Porque la aplicacción se va a conectar a estos servicios backend.

![image](https://github.com/user-attachments/assets/6e3ee6b0-3c3b-4b91-af6e-cf79decf06ec)

# Construir y Desplegar el artefacto.

> [!TIP]
> En DevOps, un artefacto se refiere a cualquier archivo generado durante el proceso de desarrollo de software que puede ser usado en las siguientes fases del ciclo de vida del software. Los artefactos son los resultados tangibles del proceso de construcción y despliegue de una aplicación.
> 

![image](https://github.com/user-attachments/assets/f65130c4-24e0-4774-8b0a-8d801523117a)

![image](https://github.com/user-attachments/assets/6e0c4437-a94f-4143-b668-924fe73eae60)

Y seleccionamos GitBash.

![image](https://github.com/user-attachments/assets/f544e8b2-85f7-4ac0-b3ec-f36932244636)

> [!WARNING]
> GitBash ahora es el terminal por defecto.
> 

Como podemos ver, ahora, si intento abrir la terminal, pues me abre la de GitBash:

![image](https://github.com/user-attachments/assets/aa70aad4-e842-44ba-8170-269085479495)

tenemos que modificar esta parte:

![image](https://github.com/user-attachments/assets/b8929f0c-d425-4d00-abeb-6d558af32276)

y así pues añadir el registro DNS. Este es el resultado, lo tenemos que hacer con los 3:

![image](https://github.com/user-attachments/assets/a836477a-2e6f-45e0-95cd-c231a117a91e)

Y ahora, vamos a montar / construir el artefacto.

## Construir el artefacto.

Prerequisitos:
- Maven
  ```
  choco install maven -y
  ```

  ```
  mvn --version
  ```
  
- JDK
  ```
  choco install corretto11jdk -y
  ```
Son herramientas esenciales que se utilizan para montar y construir artefactos en el desarrollo de software, especialmente en proyectos de Java.

- AWS CLI
  ```
  choco install awscli -y
  ```

>[!WARNING]
>Hay que reiniciar el ordenador, si los acabas de instalar. Por que, muy probablemente, no funcionen.
>

Ahora, vamos a construir el artefacto:

Tenemos que estar en:
![image](https://github.com/user-attachments/assets/3b1dff38-a825-4319-a0ab-396dcf230b35)

```
mvn install
```
>
>### Pasos que realiza `mvn install`:
>**Limpieza previa (si es necesario):**
>Si has configurado pasos de limpieza, como eliminar los artefactos anteriores, esto puede suceder antes de compilar.
>
>**Compilación (compile):**
>Maven compila el código fuente del proyecto (normalmente dentro del directorio src/main/java) y genera los archivos .class.
>
>**Pruebas (test):**
>Se ejecutan las pruebas unitarias (si tienes tests configurados, como en el directorio src/test/java). Maven usará frameworks de pruebas como JUnit o TestNG.
>
>**Empaquetado (package):**
>Maven empaqueta el proyecto en un archivo ejecutable, generalmente un JAR o WAR (dependiendo del tipo de proyecto). Este archivo contiene el código compilado y las dependencias.
>
>**Instalación (install):**
>En este paso, el artefacto generado (el JAR, WAR u otro empaquetado) se coloca en el repositorio local de Maven, que se encuentra en tu máquina, en el directorio ~/.m2/repository (o en la carpeta de >configuración de Maven si has cambiado la ruta). Esto permite que otros >proyectos que dependan de este artefacto lo puedan utilizar.
>

> [!IMPORTANT]
>**Maven guarda el artefacto compilado y empaquetado (por ejemplo, un archivo .jar o .war) en el subdirectorio target dentro del directorio raíz de tu proyecto.**
>

![image](https://github.com/user-attachments/assets/88c678ab-ec5e-42bd-9242-9ee849783a5a)

Ahora, voy a subir todo a S3 bucket, y no es posible sin la autenticación.

## Crear un usuario IAM y S3 bucket.

![image](https://github.com/user-attachments/assets/96038202-4470-4f40-921c-4fd3244fe0ef)

Este va a ser el nombre:
![image](https://github.com/user-attachments/assets/aaa75c60-f920-409d-90aa-28d81503a534)

No necesitamos, ningun acceso a la consola, solo necesitamos la clave de acceso y la clave privada.

![image](https://github.com/user-attachments/assets/db70ef15-b854-4d57-b4c7-08544b1b9bd6)

![image](https://github.com/user-attachments/assets/17e56d00-27aa-407c-a726-7f37433202f2)

![image](https://github.com/user-attachments/assets/41e0cbee-dd6f-42a9-a711-b1454b63151a)

> [!WARNING]
> **¿Cuál es el riesgo?**
>

![image](https://github.com/user-attachments/assets/d94dd8a3-fbb1-4f94-886e-6570572fbc69)

descargo el .csv.

![image](https://github.com/user-attachments/assets/d08e9eb6-2f81-48cd-9e9f-99c2f2570f13)

y ahora en el CLI del Visual Studio:

```
aws configure
```

![image](https://github.com/user-attachments/assets/d4f985fb-8a7d-46fb-ab8a-f8271230d42f)

El S3 bucket, el nombre tiene que ser 100% único, no como el mio, si no, como el de ningún otro:
```
aws s3 mb s3://forilianprojectbucket-delta
```

![image](https://github.com/user-attachments/assets/7bd06f01-e856-4e8c-bfdc-45353b125c70)

Y en teoría, con este comando, ya debería de estar:
```
aws s3 cp target/vprofile-v2.war s3://forilianprojectbucket-delta
```
![image](https://github.com/user-attachments/assets/da4610ca-1087-4c88-bcbf-2d8899fd901e)

Y ya debería haberse subido, vamos a comprobarlo, vamos a la barra de búsqueda de AWS y buscamos ``S3`

![image](https://github.com/user-attachments/assets/70685b4c-03c4-4e23-9dd0-9326edb94181)

como podemos ver, aparece:

![image](https://github.com/user-attachments/assets/cd4be745-cb6a-4090-94d1-091811697b8a)

dentro, está el artefacto:
![image](https://github.com/user-attachments/assets/8002a733-7836-4ed2-80be-466641007579)
