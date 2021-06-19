# recipe-app-api

### primero creamos el dockerfile
> La primera linea del dockerfile es la imagen de la que vamos a heredar los atributos. Hay que buscar una imagen que se adapte y tenga todo lo que se necesita, y despues la podes ir modificando para adaptarla. 

> vamos a hub.docker y buscamos python, buscamos el que mas se adapte: en este caso usamos python3.7 alpine (es el mas basico y liviano)

```
FROM python:3.9-alpine
MAINTAINER Mainque-labs developements

ENV PYTHONUNBUFFERED 1

COPY ./requierements.txt /requirements.txt
RUN pip install -r /requirements.txt

RUN mkdir /app
WORKDIR /app
COPY ./app /app

RUN  adduser -D user
USER user
```

Cosas importantes del archivo anterior:
> setear el UNBUFFERED to 1 evita problemas con python y la documentacion a la hora de correr el programa
> En el home del directorio tenemos que crear la carpeta app, porque eso es lo que le estamos diciendo. 
> IMPORTANTE: la parte de crear el usuario, es para crear un nuevo usuario que solo le permita correr el programa y no tener acceso al root, son temas de seguridad.

Tenemos que crear el requirements.txt:
>django
>djangorestframework 

fijarse en el archivo la forma correcta de setear las versiones correctas. 

### Iniciamos el dockerfile (crea la imagen a partir del dockerfile que le estamos pansando):

en linea de comandos en el directorio principal del repositorio:

```
docker build .
```

### Configuramos el dockercomposeconfiguration file:

Nos permite correr la docker image facilmente desde la direccion del proyecto. Manejar facilmente los distintos servicios por ejemplo: database, python application etc. 

El docker-compose.yml es el archivo que contiene todas las configuraciones que decoran el proyecto. Permite que puedan correr en entornos aislados. 

> primero definimos la version de docker-compuse que vamos a usar. 

```
version: "3"

services:
    app: 
        build:
            context: .
        ports: 
            - "8000:8000"
        volumes:
            - ./app:/app
        command: >
            sh -C "python manage.py runserver 0.0.0.0:8000"
```


Cosas importantes del docker-compose:
> en context ponemos un punto (.) para indicar que es el directorio actual
> el volumes sirve para que cuando hagamos una modificacion de un archivo en la app se actualice automaticamente la imagen del docker y no tenganos que volver a correr para actualizar. 
> command: sh "use Shell"  corre el django devolepment server disponible en todas las ips disponibles en el docker y el 8000 es para conectarlo con los puertos en la local machine. 

corremos el comando

``` 
docker-compose build
```

### vamos a crear el django project!!!

