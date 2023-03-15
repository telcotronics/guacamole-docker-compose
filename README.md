# Guacamole con docker-compose
Esta es una pequeña documentación sobre cómo ejecutar una instancia de **Apache Guacamole (incubando)** totalmente funcional con docker (docker-compose). El objetivo de este proyecto es facilitar la prueba de Guacamole.

## acerca de Guacamole
Apache Guacamole (en incubación) es una puerta de enlace de escritorio remoto sin cliente. Admite protocolos estándar como VNC, RDP y SSH. Se llama sin cliente porque no se requieren complementos ni software de cliente. Gracias a HTML5, una vez que Guacamole está instalado en un servidor, todo lo que necesita para acceder a sus escritorios es un navegador web.

Es compatible con RDP, SSH, Telnet y VNC y es la puerta de enlace HTML5 más rápida que conozco. Consulta la [página de inicio](https://guacamole.incubator.apache.org/) de los proyectos para obtener más información.

## requisitos previos
Necesita una instalación **docker** funcional y **docker-compose** ejecutándose en su máquina.

## Inicio rápido
Clona el repositorio GIT e inicia guacamole:

~~~bash
git clone "https://github.com/boschkundendienst/guacamole-docker-compose.git"
cd guacamole-docker-compose
./prepare.sh
docker-compose up -d
~~~

Su servidor de guacamole ahora debería estar disponible en `https://ip de su servidor: 8443/`. El nombre de usuario predeterminado es `guacadmin` con la contraseña `guacadmin`.

## Detalles
Para comprender algunos detalles, echemos un vistazo más de cerca a partes del archivo `docker-compose.yml`:

### Networking
La siguiente parte de docker-compose.yml creará una red con el nombre `guacnetwork_compose` en modo `puenteado`.
~~~python
...
# networks
# create a network 'guacnetwork_compose' in mode 'bridged'
networks:
  guacnetwork_compose:
    driver: bridge
...
~~~

### Services

#### guacd
La siguiente parte de docker-compose.yml creará el servicio guacd. guacd es el corazón de Guacamole que carga dinámicamente soporte para protocolos de escritorio remoto (llamados "complementos de cliente") y los conecta a escritorios remotos según las instrucciones recibidas de la aplicación web. El contenedor se llamará `guacd_compose` basado en la imagen acoplable `guacamole/guacd` conectada a nuestra red `guacnetwork_compose` creada previamente. Además, mapeamos las 2 carpetas locales `./drive` y `./record` en el contenedor. Podemos usarlos más tarde para mapear unidades de usuario y almacenar grabaciones de sesiones.

~~~python
...
services:
  # guacd
  guacd:
    container_name: guacd_compose
    image: guacamole/guacd
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./drive:/drive:rw
    - ./record:/record:rw
...
~~~

#### PostgreSQL
La siguiente parte de docker-compose.yml creará una instancia de PostgreSQL usando la imagen oficial de docker. Esta imagen es altamente configurable usando variables de entorno. Por ejemplo, inicializará una base de datos si se encuentra un script de inicialización en la carpeta `/docker-entrypoint-initdb.d` dentro de la imagen. Dado que mapeamos la carpeta local `./init` dentro del contenedor como `docker-entrypoint-initdb.d`, podemos inicializar la base de datos para guacamole usando nuestro propio script (`./init/initdb.sql`). Puede leer más sobre los detalles de la imagen oficial de postgres [aquí] (http://).

~~~python
...
  postgres:
    container_name: postgres_guacamole_compose
    environment:
      PGDATA: /var/lib/postgresql/data/guacamole
      POSTGRES_DB: guacamole_db
      POSTGRES_PASSWORD: ChooseYourOwnPasswordHere1234
      POSTGRES_USER: guacamole_user
    image: postgres
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./init:/docker-entrypoint-initdb.d:ro
    - ./data:/var/lib/postgresql/data:rw
...
~~~

#### Guacamole
La siguiente parte de docker-compose.yml creará una instancia de guacamole usando la imagen acoplable `guacamole` de docker hub. También es altamente configurable usando variables de entorno. En esta configuración, está configurado para conectarse a la instancia de postgres creada previamente utilizando un nombre de usuario y contraseña y la base de datos `guacamole_db`. ¡El puerto 8080 solo está expuesto localmente! Adjuntaremos una instancia de nginx para el público en el siguiente paso.

~~~python
...
  guacamole:
    container_name: guacamole_compose
    depends_on:
    - guacd
    - postgres
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRES_DATABASE: guacamole_db
      POSTGRES_HOSTNAME: postgres
      POSTGRES_PASSWORD: ChooseYourOwnPasswordHere1234
      POSTGRES_USER: guacamole_user
    image: guacamole/guacamole
    links:
    - guacd
    networks:
      guacnetwork_compose:
    ports:
    - 8080/tcp
    restart: always
...
~~~

#### nginx
La siguiente parte de docker-compose.yml creará una instancia de nginx que asigna el puerto público 8443 al puerto interno 443. Luego, el puerto interno 443 se asigna a guacamole usando `./nginx/templates/guacamole.conf.template `archivo. El contenedor utilizará el certificado autofirmado previamente generado (`prepare.sh`) en `./nginx/ssl/` con `./nginx/ssl/self-ssl.key` y `./nginx/ssl/self .cert`.

~~~python
...
  # nginx
  nginx:
   container_name: nginx_guacamole_compose
   restart: always
   image: nginx
   volumes:
   - ./nginx/templates:/etc/nginx/templates:ro
   - ./nginx/ssl/self.cert:/etc/nginx/ssl/self.cert:ro
   - ./nginx/ssl/self-ssl.key:/etc/nginx/ssl/self-ssl.key:ro
   ports:
   - 8443:443
   links:
   - guacamole
   networks:
     guacnetwork_compose:
...
~~~

## prepare.sh
`prepare.sh` es un pequeño script que crea `./init/initdb.sql` para descargar la imagen docker `guacamole/guacamole` y inicia esta:(a and start it like this):

~~~bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > ./init/initdb.sql
~~~

Crea el archivo de inicialización de base de datos necesario para postgres.

`prepare.sh` tambien crea self-signed certificate `./nginx/ssl/self.cert` y la: private key `./nginx/ssl/self-ssl.key` la cual usará nginx para https.

## reset.sh
Para restablecer todo al principio, simplemente ejecute `./reset.sh`.

## WOL

Wake on LAN (WOL) does not work and I will not fix that because it is beyound the scope of this repo. But [zukkie777](https://github.com/zukkie777) who also filed [this issue](https://github.com/boschkundendienst/guacamole-docker-compose/issues/12) fixed it. You can read about it on the [Guacamole mailing list](http://apache-guacamole-general-user-mailing-list.2363388.n4.nabble.com/How-to-docker-composer-for-WOL-td9164.html)

Wake on LAN (WOL) no funciona y no lo arreglaré porque está fuera del alcance de este repositorio. Pero [zukkie777](https://github.com/zukkie777) que también presentó [este problema](https://github.com/boschkundendienst/guacamole-docker-compose/issues/12) lo solucionó. Puede leer sobre esto en la [lista de correo de Guacamole] (http://apache-guacamole-general-user-mailing-list.2363388.n4.nabble.com/How-to-docker-composer-for-WOL-td9164 .html)


##### implementacion y puesta en marcha ######
## RUTAS
edit: /etc/netplan/50-cloud-init.yaml 
routes:
-to: 10.13.13.0/24 #network wireguard
via: 172.18.0.2 #ip docker wireguard

## FIREWALL
sudo ufw allow 8000/tcp


**Descargo de responsabilidad**
Descargar y ejecutar scripts de Internet puede dañar su computadora. ¡Asegúrese de verificar la fuente de los scripts antes de ejecutarlos!
