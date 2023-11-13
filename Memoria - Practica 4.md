# Practica 6 - WordPress, MySQL, Apache

## Prerrequisitos

```sh
ufw allow 'Apache Full'
```

## Declaración de los Servidores Virtuales

```sh
cat 000-default.conf > sitio1.conf
cat 000-default.conf > sitio2.conf

# Generar un sitio virtual nuevo para cada web para https
# apuntando al mismo contenido que su version http
# Serán configurados en el punto siguiente
cat default-ssl.conf > sitio1SSL.conf
cat default-ssl.conf > sitio2SSL.conf
```

```xml
<VirtualHost *:80>
        
            ...

        ServerName sitio1.com
        Serveralias www.sitio1.com
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/sitio1

            ...

</VirtualHost>
```

```xml
<VirtualHost *:80>
        
            ...

        ServerName sitio2.com
        Serveralias www.sitio2.com
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/sitio2

            ...

</VirtualHost>
```

## Habilitacion del Módulo SSl para HTTPS

```sh
CRT_DIR=/etc/ssl/cert
KEY_DIR=/etc/ssl/private

# Generar clave privada para el certificado autofirmado del servidor
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout $CRT_DIR/aso-apache-selfsigned.key -out $KEY_DIR/aso-apache-selfsigned.crt

a2enmod ssl
```

![alt](./Capturas%20de%20pantalla/Captura%20desde%202023-11-10%2011-34-10.png)

```xml
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>

                    ...

                ServerAdmin webmaster@localhost
                DocumentRoot /var/www/sitio1

                    ...

                #   Habilitar SSL para el servidor.
                SSLEngine on

                    ...

                #   Se requiere de certificados y claves ssl para la negociacion de claves durante la conexión al servidor
                #   Para este caso se usarán certificados autofirmados

                SSLCertificateFile      /etc/ssl/certs/aso-apache-selfsigned.crt
                SSLCertificateKeyFile   /etc/ssl/private/aso-apache-selfsigned.key

                    ...

        </VirtualHost>
</IfModule>
```

```xml
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>

                    ...

                ServerAdmin webmaster@localhost
                DocumentRoot /var/www/sitio2

                    ...

                #   Habilitar SSL para el servidor.
                SSLEngine on

                    ...

                #   Se requiere de certificados y claves ssl para la negociacion de claves durante la conexión al servidor
                #   Para este caso se usarán certificados autofirmados

                SSLCertificateFile      /etc/ssl/certs/aso-apache-selfsigned.crt
                SSLCertificateKeyFile   /etc/ssl/private/aso-apache-selfsigned.key

                    ...

        </VirtualHost>
</IfModule>
```

```sh
sudo a2enmod ssl

sudo a2ensite sitio1.conf
sudo a2ensite sitio2.conf
sudo a2ensite sitio1SSL.conf
sudo a2ensite sitio2SSL.conf

# Apache avisa de usar reload, pero puede no ser suficiente
# Aunque es más agresivo, siempre que se pueda, pude resultar 
# mejor usar restart en su lugar. Para evitar problemas futuros
#
#       sudo systemctl restart apache2.service
#
sudo systemctl reload apache
```

### Configurar fichero hosts del anfitrion de la MV

```sh
# Como sudo 
echo -e "
    127.0.0.1   sitio1.com\n
    127.0.0.1   www.sitio1.com\n
    127.0.0.1   sitio2.com\n
    127.0.0.1   www.sitio2.com\n
" >> /etc/hosts
```

![](./Capturas%20de%20pantalla/Captura%20desde%202023-11-10%2015-39-49.png)

## Instanciar Base de Datos y Crear Usuarios para las Conexiones

```mysql
# Como usuario root de la BD

# Crear un usuario por cada sitio web
CREATE USER 'sitio1'@'localhost' IDENTIFIED WITH 'caching_sha2_password' BY '@aso_sitio1';
CREATE USER 'sitio2'@'localhost' IDENTIFIED WITH 'caching_sha2_password' BY '@aso_sitio2';

# Crear la base de datos de cada sitio
CREATE DATABASE sitio1DB;
CREATE DATABASE sitio2DB;

# Dar privilegios a los usuarios sobre su correspondiete base de datos
# Por comodida se usará ALL PRIVILEGIES, aunque no es recomendable
# En su lugar se debería usar:
# 
#    GRANT connect,
#          select, 
#          insert,
#          update,
#          delete,
#          drop,
#          create,
#          alter,
#          index,
#          reference 
#    ON sitioXDB.* TO 'sitioX'@'localhost';
#
GRANT ALL PRIVILEGES ON sitio1.* TO 'sitio1'@'localhost';
GRANT ALL PRIVILEGES ON sitio2.* TO 'sitio2'@'localhost';
```

```sh
mysql -u <sitioX> -p
```

```sh
# Sitio http
wget http://sitio1.com:8888
```

## Instalación de WordPress

### Contenido de los sitios web

```sh
wget https://wordpress.org/latest.tar.gz
tar -xvf latest.tar.gz wordpress/ 
sudo rm -rf /var/www/sitio1/index.html && sudo cp wordpress/* /var/www/sitio1
sudo rm -rf /var/www/sitio2/index.html && sudo cp wordpress/* /var/www/sitio2
rm -rf latest.tar.gz
```

```sh
# Renombrar el archivo de configuracion por defecto de wordpress 
# Para evitar problemas con el instalador de wordpress
sudo mv /var/www/sitio1/wp-config-sample.php /var/www/sitio1/wp-config.php
sudo mv /var/www/sitio1/wp-config-sample.php /var/www/sitio2/wp-config.php
```

Reeditar la configuracion de los sitios virtuales de apache siguiendo el siguiente esquema:

```xml
<VirtualHost *:80>
        ServerName <sitioX>.com
        Serveralias www.<sitioX>.com
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/<sitioX>
                <Directory /var/www/<sitioX>>
                        Options FollowSymLinks
                        AllowOverride Limit Options FileInfo
                        DirectoryIndex index.php
                        Require all granted
                </Directory>
                <Directory /var/www/<sitioX>/wp-content>
                        Options FollowSymLinks
                        Require all granted
                </Directory>
</VirtualHost>
```

### Conexión a la Base de Datos de cada Sitio Web

```php
/** The name of the database for WordPress */
define( 'DB_NAME', 'sitioXDB' );

/** Database username */
define( 'DB_USER', 'sitioX' );

/** Database password */
define( 'DB_PASSWORD', '@aso_sitio1' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

//  ...

define('AUTH_KEY',         '~:9^,jOMO1k&D]X3J<3Z:>9I{CX5~3O*c|WrwOce1-F7uR#<iK`CrnVU:: [zP+I');
define('SECURE_AUTH_KEY',  'l@G.-o-6Bw;c~Vi|+SG<IZ_LIUsO77aHfiKu^us`Wj@Tg-FsUl3OpR]30$_W$+i]');
define('LOGGED_IN_KEY',    'I(c?To)LOQ?8-]@ip(d(5H,g2f?@O}zBJ~iME$EPvCm_{!-`G6aRet9H<61vS.pV');
define('NONCE_KEY',        ',TQbwiU#-7jj}+*J_=x=0+Z?qS;JH_(en+>^a;) F|UG@Cy{Pu!uU[R $i#`6NIL');
define('AUTH_SALT',        'EJdvYEzZQ?W|FfiKZ:Q}VkD-In239-F&),|@I&pvz8c~M!3Wj}4&l*q)2])mt8+$');
define('SECURE_AUTH_SALT', 'jn)#~4Nm@]55dswVMVo?[fZ&7_ui-!XrL5cr[,ByjG]qG`4eGf>85-_L5[6u}8uo');
define('LOGGED_IN_SALT',   '*eU4|8W,#mhryl-Are&cg!HThwkBpV=/vp#|yO4lsstgKF!a3=_|tax4.w@j?0]F');
define('NONCE_SALT',       'Kz+*7yU<$adV$`;( T*D>:GVK)@e`<osW-|b@CChGQgzf-$-L1H#2wZY5o}Q>7&g');

```

### Instalacion y personalización de Wordpress

![alt](./Capturas%20de%20pantalla/Captura%20desde%202023-11-11%2012-29-02.png)

![alt](././Capturas%20de%20pantalla/Captura%20desde%202023-11-11%2012-29-48.png)

### Alternativa de instalación con CLI

```sh
wp core download [--path=<path>] [--locale=<locale>] [--version=<version>] [--skip-content] [--force]
```

## Personalización de WordPress

Para editar y personalizar el contenido de un sitio wordpress, basta con utilizar los plugins y los temas correctos. Los más recomendables y los más usados, son elementor y hello elementor. Estos permiten editar la estructura del contenido web de forma interactiva  sin la necesidad de programar ni una sola línea de html, adémas existen una gran variedad de plantillas predefinidas que podría reducir el trabajo.

La instalación de plugins y temas en wordpress es bastante sencilla, basta con descargar el comprimido con el plugin/tema y descomprimiro en el directorio correspondiente dentro de wp-content/ (plugins/ o themes/)

```sh
wget https://downloads.wordpress.org/theme/hello-elementor.2.9.0.zip
wget https://downloads.wordpress.org/plugin/elementor.3.17.3.zip

sudo unzip hello-elementor.2.9.0.zip -d /var/www/sitio1/wp-content/themes/
sudo unzip hello-elementor.2.9.0.zip -d /var/www/sitio2/wp-content/themes/

sudo unzip elementor.3.17.3.zip -d /var/www/sitio1/wp-content/plugins
sudo unzip elementor.3.17.3.zip -d /var/www/sitio2/wp-content/plugins
```

Una vez descomprimidos, deben aparecer dentro del panel de adminstración, en sus respectivos aparatados, listos para se activados, y comenzar su uso:

![alt](./Capturas%20de%20pantalla/Captura%20desde%202023-11-13%2008-05-38.png)

![alt]()

![alt](https://)

## Acceso HTTP y HTTPS a los contenidos

Para atender peticiones http y https de un mismo sitio web, todas aquellas peticiones que lleguen al servicio http del sitio virtual, ha de ser redireccionada al servicio seguro (https), con una respuesta http del tipo 301. Para ello será necesario configurar el nombre del sitio web y su URL, para que sea accesible mediante peticiones https:

+ Opción 1: Dentro del panel de control de wordpress

En el panel de administración, en el apartado settings, modificar los campos Site Address y Wordpress Address

![](./Capturas%20de%20pantalla/Captura%20desde%202023-11-11%2012-29-02.png)

+ Opción 2: Modificando la tabla de configuración de wordpres dentro de MySQL

```sql
UPDATE wp_options SET option_value = 'http://<sitioX.com>:8889' WHERE option_name = 'siteurl' OR option_name = 'home';
```

Ahora solo queda configurar el servicio http dentro del servidor para que redireccione de forma permanente las peticiones que recive al servicio https. Para ello se ha de añadir un nuevo parámetro en el fichero configuracion del sitio virtual dentro de /etc/apache/sites-available:

```xml
<VirtualHost *:80>
    ...

    Redirect permanent / https://<sitioX.com>:8889/
</VirtualHost>
```

De esta forma cada vez que se lanze una petición al servidor http, esta será resuelta con un 301 a la url del servicio seguro. Y de esta forma será accesible el sitio web con peticiones http o https.

Es posible que para que el redireccionamiento funcione, sea necesario activar un módulo de apache: 

```sh
sudo a2enmod rewrite
sudo systemctl restart apache2.service
```


## Instalacion de certificados de una CA pública

```sh
sudo apt update
sudo apt install certbot
```

```sh
sudo certbot certonly --apache -d <sitioX.com>
```

```xml
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>

                ...

            SSLCertificateFile /etc/letsencrypt/live/<sitioX>/fullchain.pem
            SSLCertificateKeyFile /etc/letsencrypt/live/<sitioX>/privkey.pem

                ...

        </VirtualHost>
</IfModule>
```

```sh
sudo certbot certificates
```
