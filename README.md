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

## Por último, las claves-valor:
![image](https://github.com/user-attachments/assets/3c7ea6d4-b3e4-492b-ade8-9d3cc359d0c3)

![image](https://github.com/user-attachments/assets/fbe23bdb-11c3-4317-b52a-d6b26d6b41ab)

# Crear las instancias.
Vamos a clonar el source code

Para la base de datos:
**Nombre y etiquetas:**
| Clave    | Valor |
| -------- | ------- |
| Name  | proyecto-db01 |
| Project | db01     |

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
