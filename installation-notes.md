# Notas de instalación de Kobotoolbox para el INE

Este archivo tiene como propósito documentar los cambios que se hicieron en los archivos y en las configuraciones para poder llevar a cabo la instalación de **[KoboToolbox](https://www.kobotoolbox.org/)** en los servidores del [**INE**](https://www.ine.gob.gt/ine/).

## Arquitectura 
La instalación se realizó utilizando una arquitectura monolítica dentro de la infraestructura del INE. La aplicación se despliega con docker compose, creando los contenedores descritos en el repositorio  ["kobo-docker"](https://github.com/desarrolloinegt/kobo-docker.git). 

# Cambios en los archivos de instalación.
La instalación de kobotoolbox presentó algunos problemas al utilizar los repositorios por defecto, por lo que se procedió a realizar actualizaciones y colocarlas en los repositorios del INE. A continuación se describen los cambios y ajustes que permitieron instalar la aplicación.

### Docker-compose
En las versiones más recientes de [**docker**](https://docs.docker.com/engine/install/ubuntu/),  el comando "docker-compose" se instala como un plugin de docker y no como otra aplicación aplicación independiente. Sin embargo, los archivos de instalación verifican que esté instalado el comando "docker-compose" por lo que luego de instalar docker se creó un alias de la aplicación únicamente ser detectado por el instalador. Para realizar dicho proceso se creó el archivo `/bin/docker-compose`  y se insertó el siguiente código:
```bash
docker compose "$@" 
```
Luego le damos permisos de ejecución con `sudo chmod +x /bin/docker-compose`. 

Podemos verificar su funcionamiento con el comando: `docker-compose version` 



### Archivos de python
 
Aunque se creó el archivo docker-compose, se seguían presentando incovenientes durante la instalación por lo que se cambiaron las llamadas a docker-compose por "docker compose" en los archivos de python, un ejemplo de dicho cambio es el siguiente: 

```diff
 frontend_command = [
-          'docker-compose', 
+          'docker',
+          'compose', 
           '-f', 'docker-compose.frontend.yml',
           '-f','docker-compose.frontend.override.yml',
           '-p', config.get_prefix('frontend'),
           'up', '-d',
]

```

### Network
Se actualizó la sintaxis de los archivos docker-compose.yml 
```diff
networks:
  kobo-fe-network:
-   external:
-     name: ${DOCKER_NETWORK_FRONTEND_PREFIX}_kobo-fe-network
+   external: true
+   name: ${DOCKER_NETWORK_FRONTEND_PREFIX}_kobo-fe-network
```


# Configuración

A continuación se describen algunas consideraciones que hay que tomar en cuenta al configurar la aplicación.

## DNS

El archivo de instalación no logró crear el certificado ssl automáticamente, por lo que se indicó en la configuración que la aplicación estaría detras de un servidor reverse-proxy propio y se utilizó el puerto 8080 para alojar la aplicación. Como servidor reverse-proxy se utilizó NGINX y se agregó la configuración para el proxy server.

```nginx
server{
      server_name kobo.ine.gob.gt kc.ine.gob.gt ee.ine.gob.gt;
      listen 80;
      client_max_body_size 100M;
      location / {
                 proxy_set_header        Host $host;
                 proxy_set_header        X-Real-IP $remote_addr;
                 proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                 proxy_set_header        X-Forwarded-Proto $scheme;
                 proxy_pass              http://127.0.0.1:8080$request_uri;
                 proxy_read_timeout   	 90;
	   }
}
```
Tambien se agregaron los prefijos kobo, kc y ee en los registros DNS del proveedor y se apuntaron al servidor de la aplicación. 

Para no cambiar el código de kobo-kpi se configuró el proxy pass para reemplazar los logotipos del sitio. 
```nginx
	location /static/images/kobologo.svg {
		proxy_pass             http://127.0.0.1/logoINE.png;
	}
	location /static/favicon.png{
		proxy_pass             http://127.0.0.1/favicon.png;
	}
```
## Repositorio [kobo-docker](https://github.com/desarrolloinegt/kobo-docker.git)

Con el fin de personalizar el sitio se modificaron algunos configuraciones en los archivos docker-compose.yml, entre ellos se cambiaron algunos logotipos, pero tambien se agregarón algunos volúmenes que contienen archivos de configuración y archivos css. 
