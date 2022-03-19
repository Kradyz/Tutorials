## Cómo crear una instancia de lemmy en la nube

Video asociado con este tutorial: https://www.youtube.com/watch?v=h50M6jYZ8YU

La documentación oficial de Lemmy la puedes encontrar en este sitio:
https://join-lemmy.org/docs/es/about/about.html


(1) Elige el nombre de tu domino y regístralo con alguna empresa registradora de dominios (ejempo: namecheap.com). Costo aproximado: entre 1 y 10 euros por un año.

(3) Renta un servidor en la nube que corra Ubuntu (20.04 x64) al que puedar acceder por medio de SSH (ejemplo: serverspace.io), costo: entre 5 y 20 euros al mes. Recomiendo comenzar con opciones modestas para que el costo sea lo más bajo posible. (Por ejemplo: 1 GB ram, 30 GB memoria, 10 Mbps). Si más adelante te das cuenta de que no es suficiente, es fácil cambiar la configuración. 
Anota el ip del servidor. 

(4) En el panel de control de tu dominio, encuentra las opciones de DNS. Crea un "A record" para "@" que apunte al ip de tu servidor, y un "A record" para "www" que apunte al mismo ip. Esto es para hacer que tu dominio apunte al servidor. Nota: Cualquier cambio que hagas aquí puede tardar varios minutos en reflejarse. 

(5) Accessa al servidor por ssh desde la terminal (para Windows, usa putty). Actualiza las fuentes del gestionador de paquetes (apt) y todos los programas instalados por defecto.  Instala vim (o tu editor de texto favorito) para poder editar los archivos de configuración.

```
ssh root@ip-del-servidor

apt -y update && apt -y upgrade

apt -y install vim
```


(6) Instala docker y docker-compose. Docker es una aplicación que permite crear contenedores virtuales utilizando las instrucciones detalladas en un archivo. Los desarrolladores han generado estos archivos, con los cuales podremos construir un contenedor virtual con lemmy de forma automática más adelante.

Las instrucciones para docker pueden encontrarlas aquí:
https://docs.docker.com/engine/install/ubuntu/

Y para docker-compose aquí:
https://docs.docker.com/compose/install/

Estos son los comandos que deben correr dentro de su servidor:

```

apt -y install ca-certificates curl gnupg lsb-release
 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg 
 
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null 
 
 
apt -y update
 
apt -y install docker-ce


curl -L "https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose


```

(7) Instala nginx. Cuando alguien se conecta con tu servidor a través de un navegador, esta es la aplicación que especifica cómo manejar la conexión . [https://nginx.org/en/]

```
apt -y install nginx
```

Ahora hay que probar si funciona. Inicia el servicio de nginx usando:

```
sudo systemctl enable nginx

sudo systemctl start nginx
```
Si te sale un error, utiliza el siguiente comando para ver qué es lo que ocasionó el error:

```
sudo systemctl start nginx
```

Si te aparece un error cómo el siguiente:
```
nginx: [emerg] socket() [::]:443 failed (97: Address family not supported by protocol)
```

Es por que la máquina que estás usando no tiene una dirección de IPv6.
Entra al archivo 

/etc/nginx/sites-enabled/default

y comenta la linea:
```
#       listen [::]:80 default_server;

```

De nuevo, intenta correr
```
sudo systemctl start nginx
```


Con el navegador, entra a tu dominio. Deberías ver la siguiente página:

![Pasted image 20211211002801](https://user-images.githubusercontent.com/81911574/145698065-7d0bee26-ff54-4d39-8755-dea60d78f867.png)


(8) Instala certbot. Esta aplicación te permite generar los certificados SSL, que son necesarios pare establecer una conección segura mediante https. Es necesario tenerlos para que activar la federación con otras instancias. 

```
apt install certbot python3-certbot-nginx
```

(9) Ahora puedes generar los certificados SSL.

Para asegurarte de que todo está bien configurado, primero corre:

```
certbot certonly --nginx --dry-run -d dominio.xyz,www.dominio.xyz  

```

Si te da un error, es posible que el DNS no esté bien configurado, o que sea necesario configurar los puertos manualmente. En el caso de que no te de un error, puedes quitar la opción de --dry-run para generar los certificados. 

```
certbot certonly --nginx -d domain.xyz,www.domain.xyz  
```

Esto generará los certificados de SSL para el dominio, y los guardará en:

/etc/letsencrypt/live/domain.xyz


(9) Configura nginx

Ahora necesitamos configurar la configuración de nginx. 

Debemos agregar la configuración en el siguiente folder:

```
cd /etc/nginx/sites-enabled
```

Un ejemplo de cómo debe ser la configuración se puede encontrar aquí, en su github: 
https://github.com/LemmyNet/lemmy-ansible/blob/main/templates/nginx.conf#L63
 
Hacemos una copia de este archivo con el siguiente comando:

```
wget https://raw.githubusercontent.com/LemmyNet/lemmy-ansible/main/templates/nginx.conf -O lemmy.conf
```

Abre el archivo usando vim

```
vim lemmy.conf
```

Necesitas reemplazar las siguientes variables, incluyend los {{  }}.

```
{{domain}} => dominio.xyz
{{lemmy_port}} => 8536
{{lemmy_ui_port}} => 1235
```

Alternativamente, puedes utilizar el siguiente comando para reemplazar las variables usando sed, sólo tienes que reemplazar dominio.xyz por tu dominio en el comando:
```
sed 's/{{domain}}/dominio.xyz/g;s/{{lemmy_ui_port}}/1235/g;s/{{lemmy_port}}/8536/g' lemmy.conf -i
```

Si tuviste un error de nginx por no tener una dirección IPv6 en la sección anterior, aquí también tendrás que comentar las siguientes líneas:
```
#    listen [::]:80;

#    listen [::]:443 ssl http2;
```

Guarda los cambios.

Ahora tienes que reiniciar el servicio de nginx para que se cargue la nueva configuración, esto se hace con el siguiente comando:

```
systemctl restart nginx
```

(10) Configuración de Lemmy

Primero, crearemos la carpeta an la instalaremos lemmy. Por ejemplo, en /var/www/dominio

```
mkdir /var/www/dominio


cd /var/www/dominio

```


En el github de Lemmy, tienen un folder llamado "docker" (https://github.com/LemmyNet/lemmy/tree/main/docker). Esta carpeta contiene un archivo de configuración cómo ejemplo, llamado "lemmy.hjson".

También contiene varias carpetas con configuraciones para construir el contenedor de docker. Les recomiendo generar el contenedor de "producción", que se encuentro en el folder "prod".  El archivo con las instrucciones se llama docker-compose.yml. 

Para descargar estos dos archivos, usamos:

```
wget https://raw.githubusercontent.com/LemmyNet/lemmy/main/docker/prod/docker-compose.yml

wget https://raw.githubusercontent.com/LemmyNet/lemmy/main/docker/lemmy.hjson

```

```
mkdir -p volumes/pictrs

chown -R 991:991 volumes/pictrs

```



En el docker-compose.yml

Elije la configuracion para tu base de datos de postgres.

Tambien hay que cambiar 
LEMMY_EXTERNAL_HOST=localhost:8536 => LEMMY_EXTERNAL_HOST=dominio.xyz:8536
Ahora ya podemos la instancia de lemmy usando:

Ahora abre el archivo lemmy.hjson para configurar la instancia. Debes cambiar los detalles del administrador, establecer nombre del sitio, cambiar el hostname a 'dominio.xyz', y cambiar los detalles de la base de datos de postgres.

Para activar la federación, agreguen lo siguiente arriba de la linea que dice "slur_filter":
```
  federation: {
        enabled: true
  }
```

Para comenzar la instancia, utiliza el siguiente comando:


```
docker-compose up -d
```

Entra de nuevo a tu dominio en tu navegador y deberías poder ver tu instancia ya lista! 

Para apagar la instancia temporalmente, puedes usar el siguiente comando desde la carpeta /var/www/dominio.xyz:

```
docker-compose down
```
