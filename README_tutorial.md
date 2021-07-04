# recipe-app-api

### primero creamos el dockerfile
> La primera linea del dockerfile es la imagen de la que vamos a heredar los atributos. Hay que buscar una imagen que se adapte y tenga todo lo que se necesita, y despues la podes ir modificando para adaptarla. 

> vamos a hub.docker y buscamos python, buscamos el que mas se adapte: en este caso usamos python3.9 alpine (es el mas basico y liviano)

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

setear el UNBUFFERED to 1 evita problemas con python y la documentacion a la hora de correr el programa. Principalmente es para mostrar mensajes de python directamente en pantalla
En el home del directorio tenemos que crear la carpeta app, porque eso es lo que le estamos diciendo. 
IMPORTANTE: la parte de crear el usuario, es para crear un nuevo usuario que solo le permita correr el programa y no tener acceso al root, son temas de seguridad.

**Tenemos que crear el requirements.txt:**
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

El docker-compose.yml es el archivo que contiene todas las configuraciones que decoran el proyecto. Permite que puedan correr en entornos aislados. El docker compose es una herramienta que nos permite correr nuestra iamgen docker facilmente desde el directorio que tenemos el proyecto.  

> primero definimos la version de docker-compose que vamos a usar. 

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
> En context ponemos un punto (.) para indicar que es el directorio actual
> El volumes sirve para que cuando hagamos una modificacion de un archivo en la app se actualice automaticamente la imagen del docker y no tenganos que volver a correr para actualizar. 
> command: sh "use Shell"  corre el django devolepment server disponible en todas las ips disponibles en el docker y el 8000 es para conectarlo con los puertos en la local machine. 

corremos el comando

``` 
docker-compose build
```

### Vamos a crear el django project!!!

En linea de comandos:
```
docker-compose run app sh -c "django-admin.py startproject app ."
```

Comentarios:
> run a shell script el -c lo usa para ver exactamente el commando que esta corriendo. Podria correrlo directamente en la imagen. Pero asi es mas claro lo que estas haciendo. 

### Vamos a habilitar travis CI (Continue Integration) que automatiza los test y checks cada vez que pusheamos al repositorio.

Tenemos que ir la pagina travis-ci.com y vincular la cuenta de github. Una vez que veamos todos los repositorios tenemos que darle click en Trigger a build!

#### Ahora viene la parte de los archivos:

Primero tenemos que crear el archivo -travis.yml en el directorio principal para que travis reconozca y realice los tests:

```
language: python
python:
  - "3.6"

services: 
  - docker

before_scripts: pip install docker-compose

script:
  - docker-compose run app sh -c "python manage.py test && flake8"
  ```

el flake8 es para chequear el linting. 

lo siguiente es crear una archivo dentro de la carpeta app para el flake8: .flake8

```
[flake8]
exclude = 
    migrations,
    _pycache,
    manage.py,
    settings.py
```

Falta modificar el requirements.txt: agregamos el flake8

```
flake8>=3.9.2,<3.10.0
```

Excluimos esos archivos porque son los que genera django por default y tienen sus propias configuraciones, explotaria el linting sino. 

Ahora ya podemos pushear al repositorio y vamos a ver en la pagina de travis-ci el estado de los test. En cuanto a la app no tiene ningun test porque todavia no los creamos. Crea la imagen del docker y todo ok. 

### Lo basico de UNIT TEST

El django unit test framework busca todos los archivos que comiencen con test*

Para testear como funcionan los UNIT test creamos un app facil adentro de la carpeta app/app: calc.py

```python
def add(x, y):
    """Add two number together"""
    return x + y
```

despues creamos el archivo test.py: 

```python
from django.test import TestCase

from app.calc import add

class CalcTests(TestCase):

    def test_add_numbers(self):
        """Test that two numbers are added together"""

        self.assertEqual(add(3, 8), 11)
```

para correr el test hacemos lo siguiente:

> en linea de comandos:

```
docker-compose run app sh -c "python manage.py test"
```

devuelve lo sguiente:

```Creating recipe-app-api_app_run ... done
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.
----------------------------------------------------------------------
Ran 1 test in 0.001s

OK
Destroying test database for alias 'default'...
```

### comenzamos con la app:

Primero eliminamos los archivos que creamos de prueba para los test. 

en linea de comandos:
```
docker-compose run app sh -c "python manage.py startapp core"
```

Esto va a crear una carpeta llamada core con los archivos de la app. Por el momento hacemos lo siguiente:

> - Creeamos una carpeta que se llama tests (esta carpeta va a nuclear todos los tests de core)
> - Adentro de la carpeta creamos un __init__.py  
> - Borramos el archivo test.py y views.py que creo django. 

### Comenzamos a crear el modelo customizado (primero encarando los tests)

Lo primero que hay que hacer es modificar modificar el settings.py:

```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'core',
```

Lo unico que hicimos es agregar en INSTALLED_APPS la app que acabamos de crear, "core"

El siguiente paso es crear el archivo test_models.py en la carpeta tests q creamos anteriormente. 

```python
from django.test import TestCase
from django.contrib.auth import get_user_model

class ModelTests(TestCase):

    def test_create_user_with_email_successfull(self):
        """Test creating a new user with an email is successful"""
        email = 'test@mainquelabsdev.com'
        password = 'Testpass123'
        user = get_user_model().objects.create_user(
            email=email,
            password=password
        )

        self.assertEqual(user.email, email)
        self.assertTrue(user.check_password(password))
```
Cosas importantes:

> Se podria importar el módulo directo de django models pero no es recomendado por django. Usando la forma que se usa aca facilita mucho las cosas porque por cada parte que usa la funcion get_user_model solo tenes que modificar el settings.py para cambiarlo.
> Como la contraseña viene encriptada hay que usar la funcion check_password que devuelve True or False en funcion de la contraseña. 

### Ya podemos crear el user model

```python
from django.db import models
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, \
                                        PermissionsMixin
```

Primero creamos el userManager:

Cosas importantes:

> Aca estamos creando la clases que heredan los atributos de BaseUserManager, AbstractBaseUser, PermissionsMixin principalmente para modificarlas y que acepten que el usuario se logge con el email en lugar de solo con el user_name
> Es importante usar la funcion set_password para que siempre quede encriptada
> Usermanager es la clase que ayuda a crear un usuario o un superuser
> el **extra_field toma todas las funciones extra que le pasamos y hace mas flexible el agregar nuevos campos a user y no tener que modificar directamente el modelo. 
> el save(using=self._db) es utilizado para dar soporte a varias bases de datos, en este caso no es necesario pero es una buena practica.

Una vez que tenemos el manager creamos el modelo:

Cosas importantes:

> por default el user_field es username entonces tenemos que modificarlo a 'email'


**Buena información sobre la construccion de usuarios con AbtractBaseUser and AbstractUser:**

> https://testdriven.io/blog/django-custom-user-model/

```python
class UserManager(BaseUserManager):

    def create_user(self, email, password=None, **extra_fields):
        """Creates and saves a new user"""
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)

        return user


class User(AbstractBaseUser, PermissionsMixin):
    """Custom user model that supports using email instead of username"""
    email = models.EmailField(max_length=255, unique=True)
    name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = UserManager()

    USERNAME_FIELD = 'email'
```
Mas info por si es necesaria con respecto a lo anterior:

```
https://docs.djangoproject.com/en/3.2/topics/auth/customizing/
```

lo siguiente es modifcar el settings.py:

solo agregamos la siguiente linea al final de todo:


```
AUTH_USER_MODEL = 'core.User'
```
**Es importante hacer esto antes de correr el makemigrations!**

Si no lo hacemos antes de correr el migrations podemos tener problemas con los schemas. Siempre hay que hacer esto antes de correr el primer migration. 

> Para que lo anterior??

```
Substituting a custom User model¶
Some kinds of projects may have authentication requirements for which Django’s built-in User model is not always appropriate. For instance, on some sites it makes more sense to use an email address as your identification token instead of a username.

Django allows you to override the default user model by providing a value for the AUTH_USER_MODEL setting that references a custom model:
```

tenemos que hacer los migrations: 

En linea de comandos: 

```
docker-compose run app sh -c "python manage.py makemigrations core"

Ahora corremos los test:

```Creating recipe-app-api_app_run ... done
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.
----------------------------------------------------------------------
Ran 1 test in 0.149s

OK
Destroying test database for alias 'default'...
```

TODO OK POR EL MOMENTO

### Lo siguiente que hacemos crear un normalizador del email:

En test_models.py:

agregamos el siguiente test: 

```python
    def test_new_user_email_normalized(self):
        """Test the email for a new user is normalized"""
        email = 'test@MAINQUELABSDEV.com'
        user = get_user_model().objects.create_user(email, 'test123')

        self.assertEqual(user.email, email.lower())
```

Lo siguiente es correr el test y como esperamos falle!

Ahora modificamos el modelo y lo unico que tenemos que hacer es usar la funcion normalize_email que viene con BaseUserManager:

modificamos la linea correspondiente al email:

```python
def create_user(self, email, password=None, **extra_fields):
        """Creates and saves a new user"""
        user = self.model(email=self.normalize_email(email), **extra_fields)
        user.set_password(password)
        user.save(using=self._db)

        return user
```

Volvemos a correr el test y deberia estar todo bien!

### Ahora vamos a crear un validation para asegurar que el email es dado cuando se usa la funcion crear user

Bien para esto creamos un nuevo test: 

```python
  def test_new_user_invalid_email(self):
        """Test creating user with no email raises error"""
        with self.assertRaises(ValueError):
            get_user_model().objects.create_user(None, 'test123')
```

Aca lo nuevo que estamos usando es el assertRaises

Corremos el test y deberia fallar porque no implementamos la funcion

Ahora modificamos el models, y lo unico que tenemos que hacer es modicar la linea antes de asignar el email con lo siguiente:

```python
   def create_user(self, email, password=None, **extra_fields):
        """Creates and saves a new user"""
        if not email:
            raise ValueError('Users must have email address')
        user = self.model(email=self.normalize_email(email), **extra_fields)
        user.set_password(password)
        user.save(using=self._db)

        return user
```

volvemos a correr el test y deberia estar todo OK!

### Ahora vamos a crear la funcionalidad para permitir crear un superuser:

Tenemos que agregar la funcion create super user a BaseUserManager es una funcionalidad de django managment command que permite agregar superusers por la linea de comandos.

Primero como siempre, creamos el test:

```
def test_create_new_superuser(self):
        """Test creating a new superuser"""
        user = get_user_model().objects.create_superuser(
            'test@mainquelabsdev.com',
            'test123'
        )

        self.assertTrue(user.is_superuser)
        self.assertTrue(user.is_staff)
```

Siguiente despues de correr el test y que falle modificamos el models.py:

Agregamos la función createsuper user al UserManager

```python
class UserManager(BaseUserManager):

    def create_user(self, email, password=None, **extra_fields):
        """Creates and saves a new user"""
        if not email:
            raise ValueError('Users must have email address')
        user = self.model(email=self.normalize_email(email), **extra_fields)
        user.set_password(password)
        user.save(using=self._db)

        return user

    def create_superuser(self, email, password):
        """Creates a new super user"""
        user = self.create_user(email, password)
        user.is_staff = True
        user.is_superuser = True
        user.save(using=self._db)

        return user
```

Corremos nuevamente el test, pero en este caso usando el linting:

```
docker-compose run app sh -c "python manage.py test && flake8"
```

Va a fallar el linting porque tenemos una importacion que no estamos utilizando en admin.py, por ahora solo la comentamos y listo. 

Volvemos a correr el test y todo perfecto!

### Vamos a testear la actualizacion del Django Admin

Lo primero que vamos a hacer es crear el test: 

> Para esto creamos un nuevo archivo: test_admin en la carpeta tests:

```python
from django.test import TestCase, Client
from django.contrib.auth import get_user_model
from django.urls import reverse


class AdminSiteTests(TestCase):

    def setUp(self):
        self.client = Client()
        self.admin_user = get_user_model().objects.create_superuser(
            email = 'admin@mainquelabsdev.com',
            password = 'password123'
        )
        self.client.force_login(self.admin_user)
        self.user = get_user_model().objects.create_user(
            email='test@mainquelabsdev.com',
            password='password123',
            name='Test user full name'
        )
    
    def test_users_listed(self):
        """Test that users are listed on user page"""
        url = reverse('admin:core_user_changelist')
        res = self.client.get(url)

        self.assertContains(res, self.user.name)
        self.assertContains(res, self.user.email)
```


Cosas importantes de este archivo:


> Lo que hace principalmente es testear que los usuarios se listan correctamente en django admin, esto se tiene que hacer porque modificamos los valores por default (django admin espera el user_name como default y nosotros se lo modificamos por el email, entonces tenemos que tambien modificar el django admin para que acepte esto)

> Reverse es una helper function que nos permite generar urls para nuestra django admin page (Se usa de esta forma y no intrudiciendo la URL directamnete porque nos facilita que cuando mofiquemos la url no tengamos que modificar tambien los tests)

> El assertContains lo que hace es chequear es que la respuesta contiene cierto item, entiende el objeto, entiendo las respuestas, basicamente hace todo solo


El test client nos permite simular el web browser sin tener que correr el server, esto permite hacer el test mucho mas rapido

```
https://docs.djangoproject.com/en/2.2/topics/testing/tools/#overview-and-a-quick-example

The test client
The test client is a Python class that acts as a dummy Web browser, allowing you to test your views and interact with your Django-powered application programmatically.

Some of the things you can do with the test client are:

Simulate GET and POST requests on a URL and observe the response – everything from low-level HTTP (result headers and status codes) to page content.
See the chain of redirects (if any) and check the URL and status code at each step.
Test that a given request is rendered by a given Django template, with a template context that contains certain values.
Note that the test client is not intended to be a replacement for Selenium or other “in-browser” frameworks. Django’s test client has a different focus. In short:

Use Django’s test client to establish that the correct template is being rendered and that the template is passed the correct context data.
Use in-browser frameworks like Selenium to test rendered HTML and the behavior of Web pages, namely JavaScript functionality. Django also provides special support for those frameworks; see the section on LiveServerTestCase for more details.
A comprehensive test suite should use a combination of both test types.
```

Vamos a correr el test, y tiene que fallar:

```
RROR: test_users_listed (core.tests.test_admin.AdminSiteTests)       
Test that users are listed on user page
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/app/core/tests/test_admin.py", line 23, in test_users_listed 
    url = reverse('admin:core_user_changelist')
  File "/usr/local/lib/python3.9/site-packages/django/urls/base.py", line 86, in reverse
    return resolver._reverse_with_prefix(view, prefix, *args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/django/urls/resolvers.py", line 694, in _reverse_with_prefix
    raise NoReverseMatch(msg)
django.urls.exceptions.NoReverseMatch: Reverse for 'core_user_changelist' not found. 'core_user_changelist' is not a valid view function or pattern name.

----------------------------------------------------------------------
Ran 5 tests in 1.884s
```


### Ahora vamos a customizar el admin.py:

Basicamente lo que vamos a hacer es importar default user admin y vamos a cambiar algunas de las variables de la clase para que entienda el user model que nosotros creamos.

En admin.py bamos a extender la clase UserAdmin para que soporte lo que nosotros buscamos:

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin

from core import models


class UserAdmin(BaseUserAdmin):
    ordering = ['id']
    list_display = ['email', 'name']


admin.site.register(models.User, UserAdmin)
```

**Todavía no podemos probarlo en el navegador porque no creamos las bases de datos**


### Vamos a modificar el django admin para que pueda modifcar el user model:

Primero vamos a testear que la pagina para realizar cambios se renderice correctamente. 

Algo importante es que no testeamos que realice correctamente un POST a la pagina de cambios. Esto debe a que es parte de la django admin module y no es recomendado testear las dependencias del proyecto. No es necesario testear features especificas al framework o frameworks externos, solo nos mantenemos testeando nuestro propio codigo, y le dejamos el testo de esos modulos a los que los desarrollan. 

Creamos el test correspondiente:

```python
def test_user_change_page(self):
    """Test that the user edit page works"""
    url = reverse('admin:core_user_change', args=[self.user.id])
    res = self.client.get(url)

    self.assertEqual(res.status_code, 200)
```

Aca lo unico que estamos haciendo es chequear que la respuesta de el estado correcto. 

Pasamos a modificar el admin.py para que permita la modificacion de usuarios:

admin.py deberia quedar de la siguiente manera:

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.utils.translation import gettext as _

from core import models


class UserAdmin(BaseUserAdmin):
    ordering = ['id']
    list_display = ['email', 'name']
    fieldsets = (
        (None, {'fields': ('email', 'password')}),
        (_('Personal Info'), {'fields': ('name',)}),
        (
            _('Permissions'),
            {'fields': ('is_active', 'is_staff', 'is_superuser')}
        ),
        (_('Important dates'), {'fields': ('last_login',)})
    )


admin.site.register(models.User, UserAdmin)
```

Basicamente lo que hicimos es modificar el campo fieldsets de UserAdmin para que nos permita utilizar el user model que creamos nosotros. Lo que estamos haciendo es definir la secciones de nuestra pagina change_and_create_page. Cada parentesis es una nueva seccion, lo primero es el titulo y luego definimos los campos que se quieren mostrar. 

**Algo nuevo**

Otra cosa nueva es el gettext, esta es la convencion recomendada para convertir strings de python en algo "human readable". Al usar de esta forma pasa atraves del "translation engine" que no lo vamos a estar usando en este proyecto, pero si se quiere tener soporte para varios idiomas en el proyecto esta es la forma

### Lo último que tenemos que agregar al admin para que funcione con el user model que creamos es la add_page:

Creamos un nuevo test para verificar que la add_page renderice correctamente:

```python
    def test_create_user_page(self):
        """Test that the create user page works"""
        url = reverse('admin:core_user_add')
        res = self.client.get(url)

        self.assertEqual(res.status_code, 200)
```

Es bastante similar al test que creamos antes.

Testamos y probamos que el test falla con el error correspondiente, lo siguiente es modificar el admin.py para que la add_user page funcione con nuestro user model:

```python
class UserAdmin(BaseUserAdmin):
    ordering = ['id']
    list_display = ['email', 'name']
    fieldsets = (
        (None, {'fields': ('email', 'password')}),
        (_('Personal Info'), {'fields': ('name',)}),
        (
            _('Permissions'),
            {'fields': ('is_active', 'is_staff', 'is_superuser')}
        ),
        (_('Important dates'), {'fields': ('last_login',)})
    )
    add_fieldsets = (
        (None, {
            'classes': ('wide',),
            'fields': ('email', 'password1', 'password2')
        }),
    )
```

La sintaxis es muy similar a la de fieldsets. En este caso se agrega el classes (es una referencia a estilos CSS).

Con esto agregado ya podemos correr el test y debería estar todo OK.

### Agregamos y hacemos el setup para que funcione con postgres en lugar de SQLlite3

Lo que vamos a hacer primero es configurar el docker-compose para permitir crear un servicio para la base de datos y configurar la base de datos. 

modificamos el docker-compose.yml para que quede asi:

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
        environment:
            - DB_HOST=db
            - DB_NAME=app
            - DB_USER=postgres
            - DB_PASS=supersecretpassword
        depends_on:
            - db
        
    db:
        image: postgres:13-alpine
        environment:
            - POSTGRES_DB=app
            - POSTGRES_USER=postgres
            - POSTGRES_PASSWORD=supersecretpassword
```

Buscamos en hub-docker la imagen que queremos instalar de postgres, en este caso la 13-alpine version 13 de postgres y la imagen mas liviana de todas. 

Si vemos en hub-docker podemos ver todas las variables de entorno que tenemos para configurar. las variables de entorno tienen que estar en ese formato si o si!

Por su puesto, cuando se esta enviado el proyecto a produccion se debe encriptar la contraseña correctamente. 

Lo siguiente que hicimos es agregar algunas variables de entorno para la app. 

por ultimo el depends_on cuando corremos el docker-compose podemos indicarle que servicios dependen de otros servicios. Se puede agregar multiples servicios. 
En lo que hicimos nosotros principalmente significa 2 cosas:

> 1) El servicio db arranca antes que la app
> 2) El servicio db va estar disponible a la red cuando utiliza el host_name db. Cuando en adentro del servicio app nos conectamos al host db se conecta a cualquier que esta corriendo el servicio db.

### Vamos a configurar los archivos para instalar las dependecias necesarias y los modulos requeridos para dar soporte a la base de datos postgres:

Lo primero que tenemos que hacer es modificar el requirements.txt:

Agregamos el paquete que django recomienda para comunicar django con postgres es psycopg2. 

Agregamos los siguiente al archivo:

```
psycopg2>=2.9.1,<2.10.0
```

Ahora para instalar el paquete en el sistema, tenemos que instalar algunas dependencias para eso tenemos que modificar el dockerfile:

Lo modificamos para que quede asi:

```
FROM python:3.9-alpine
MAINTAINER Mainque-labs developements

ENV PYTHONUNBUFFERED 1

COPY ./requirements.txt /requirements.txt
RUN apk add --update --no-cache postgresql-client
RUN apk add --update --no-cache --virtual .tmp-build-deps \
            gcc libc-dev linux-headers postgresql-dev

RUN pip install -r /requirements.txt
RUN apk del .tmp-build-deps

RUN mkdir /app
WORKDIR /app
COPY ./app /app

RUN  adduser -D user
USER user
```
**mas info sobre esto en:**

https://www.mlr2d.org/contents/djangorestapi/06_dockertips_settingup_postgres_backend


Algunas cosas importantes: 

Vamos instalar cosas que deben ser permanentes postgresql-client y todo lo demas no es necesario despues de instalar las dependencias entonces lo vamos a eliminar. 

> El --no-cache es para que no se instale el registry index en el dockerfile de esta manera instalamos la menor cantidad de cosas extra que son incluidas en el docker container. No incluir nada que genere algun efecto inesperado. 

> Para instalar los archivos temporales usamos el --virtual y los eliminamos despues de instalar el requirements.txt

> La lista de dependencias necesarias no es tan simple de encontrar. 


### Cofigurar el Django Project para que use postgres

Primero moficamos el settings.py en la parte de DATABASES:

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': os.environ.get('DB_HOST'),
        'NAME': os.environ.get('DB_NANE'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASS')
    }
}
```

Esta es la forma correcta de trabajar ya que solo tenemos que cambiar las variables de entorno de donde estemos corriendo el servicio para que funcione. 


### Vamos a agregar test para wait_for_db command:

Este testeo es porque hay veces que django falla al iniciar por un error de la base de datos. Esto es porque postgres tiene que hacer unos setups antes de poder aceptar conecciones. Los errores surgen porque django intenta conectarse antes de que la base de datos este lista.

Creamos un nuevo archivo en tests:

```python
from unittest.mock import patch

from django.core.management import call_command
from django.db.utils import OperationalError
from django.test import TestCase


class CommandTest(TestCase):

    def test_wait_for_db_ready(self):
        """Test waiting for db when db is available"""
        with patch('django.db.utils.ConnectionHandler.__getitem__') as gi:
            gi.return_value = True
            call_command('wait_for_db')
            self.assertEqual(gi.call_count, 1)

    @patch('time.sleep', return_value=True)
    def test_wait_for_db(self, ts):
        """Test waiting for db"""
        with patch('django.db.utils.ConnectionHandler.__getitem__') as gi:
            gi.side_effect = [OperationalError] * 5 + [True]
            call_command('wait_for_db')
            self.assertEqual(gi.call_count, 6)
```

Cosas importantes:

Lo que se usa en esta parte es un mock de los servicios, esto es para no tener que estar por ejemplo enviando un email, cada vez que testeamos el codigo. la libreria unittest.mock provee las funciones necesarias. Con patch suscribimos el servicio que django usa.

En el segundo caso utiliza patch como un decorador. Las cosas necesarias para utilizarlo de esta forma es pasar el return value como argumento del decorador, he incluir el argumento en la funcion cumpliria la funcion del nombre del mock que estamos desarrollando. En este caso ts. Hacemos esto para realmente no tener que esperar realmente el tiempo entre intento de conexion y no ralentizar el test.

El side_effect nos permite hacer un raise una determinada cantidad de veces del operationerror simulando los intentos de conexion a la base de datos. 

Realizamos el test y deberia fallar porque no implementamos todavia el comando wait_for_db

### Construimos el comando wait_for_db

Para esto lo estandar es crear una carpeta en nuestra app core llamada management y dentro de esta otra que se llama commands. En esta carpeta creamos el comando que queremos crear (wait_for_db). Es importante agregar el __ini__.py a las carpetas que generamos.
El archivo wait_for_db.py queda asi:  

```python
import time

from django.db import connections
from django.db.utils import OperationalError
from django.core.management.base import BaseCommand


class Command(BaseCommand):
    """Django command to pause execution until database is available"""

    def handle(self, *args, **options):
        self.stdout.write('Waiting for database...')
        db_conn = None
        while not db_conn:
            try:
                db_conn = connections['default']
            except OperationalError:
                self.stdout.write('Database unavailable, waiting 1 second...')
                time.sleep(1)

        self.stdout.write(self.style.SUCCESS('Database available!'))
```
> Modulos:
El modulo time es el que vamos a utilizar para manejar los tiempos de espera entre los chequeos que hace la aplicacion de la base de datos. 
El modulo connectios es para testar que la base de datos se encuentre disponible
El operationalError es que django va a dar cuando la base de datos no se encuentre disponible.
Para crear nuestro propio comando partimos de BaseCommand. 

> Función:
La funcion handle establece que es lo que se ejecuta cuando corremos el commando que estamos creando. 
No hay cosas fuera de lo común dentro de la aplicacion. 

### Tenemos que intruir al docker-compose para que espere a la base de datos usando el comando que creamos:

Tenemos que modificar el archivo docker-compose.yml:

```
command: >
    sh -c "python manage.py wait_for_db && 
    python manage.py migrate && 
    python manage.py runserver 0.0.0.0:5000"
```

Modificando el docker-compose con lo anterior le estamos instruyendo que espere la base de datos este lista, luego realiza las migraciones correspondientes y recien despues de esto inicia el server. 

> Cosas importante:
>- Es importante realizar el migrate antes de levantar el server por primera vez ya que de otra forma se podrian generar problemas
>- En el docker-compose el comando sh -c el -c es con minuscula sino falla!!!


Cuando realizamos esta modificación tenemos que correr el siguiente comando en la Terminal:

```yml
docker-compose up
```

### Ya podemos testear nuestra app en el navegador:

siempre que querramos levantar el servidor: 

```
docker-compose up
```

vamos a ver el siguiente mensaje en la terminal: 

```
app_1  | System check identified no issues (0 silenced).
app_1  | June 29, 2021 - 15:01:25
app_1  | Django version 3.2.4, using settings 'app.settings'
app_1  | Starting development server at http://0.0.0.0:5000/
app_1  | Quit the server with CONTROL-C.
```

No utilizamos la direccion que se ve arriba ya que esta es la direccion interna del contenedor docker

Nos conectamos al Local Host que es el port 8000 dentro de nuestro contenedor para eso:

La direccion entonces es: 127.0.0.1/8000

Si vamos a esta direccion vamos a ver la landing page de django indicando que todo esta funcionando bien. 

Tenemos que ir al admin: 127.0.0.1/8000/admin

Pero antes tenemos que crear el superusuer para eso una nueva terminal corremos el siguiente comando:

```
docker-compose run app sh -c "python manage.py createsuperuser"
```

Seguimos las instrucciones y ya tendremos nuestro supersuser creado y ya podremos loggearnos en el django admin. 

 ### Vamos a crear los endpoints para agregar, modificar, borrar usuarios: 

Primero creamos la app user para eso:
En terminal:
```
docker-compose run --rm app sh -c "python manage.py user"

En esta variante una vez que finaliza el comando borra el contenedor. No algo tan importante porque si corrieramos docker-compose down seria lo mismo. 

Un vez que la app se inicio tenemos que limpiar la carpeta que se creo:

> - Borramos el migrations
> - Borramos el admin 
> - Borramos el models

Principalmente borramos lo anterior porque vamos a usar los que estan definidos en la core app. 

Lo siguiente es modificar el settings.py para agregar en installed app lo agregado y lo que vamos a usar:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework.authtoken',
    'core',
    'users',
]
```
### Vamos a crear la create_user API.

Comenzamos creando los tests necesarios:

```python
test_users_api.py: 

from django.test import TestCase
from django.contrib.auth import get_user_model
from django.urls import reverse

from rest_framework.test import APIClient
from rest_framework import status


CREATE_USER_URL = reverse('user:create')


def create_user(**params):
    return get_user_model().objects.create_user(**params)


class PublicUserApiTests(TestCase):
    """Test Users API (public)"""

    def setUP(self):
        self.client = APIClient()

    def test_create_valid_user_success(self):
        """Test creating user with valid payload is succesfull"""
    
        payload = {
            'email': 'test@mainqlabsdev.com',
            'password': 'testpass',
            'name': 'Test name',
        }
        res = self.client.post(CREATE_USER_URL, payload)

        self.assertEqual(res.status_code, status.HTTP_201_CREATED)
        user = get_user_model().objects.get(**res.data)
        self.assertTrue(user.check_password(payload['password']))
        self.assertNotIn('password', res.data)

    def test_user_exists(self):
        """Test creating a user that already exists fails"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'testpass'}
        create_user(**payload)

        res = self.client.post(CREATE_USER_URL, payload)

        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_password_too_short(self):
        """Test that the password must be more than 5 characters"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'pw'}
        res = self.client.post(CREATE_USER_URL, payload)

        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)
        user_exists = get_user_model().objects.filter(
            email=payload['email']
        ).exists()
        self.assertFalse(user_exists)
```

Del archivo lo mas importante es la diferenciación en el test para usuarios no autenticados y si autenticados.  

Para test_create_user verificamos que cuando creamos un usuario valido, la respuesta sea la correcta y luego se verifica que el usuario fue realmente creado. Para esto obtenemos el user model pasando la respuesta obtenida y luego se verifica que la contraseña es correcta. Luego se verifica que la contraseña no es mostrada en la respuesta. 

Luego se verifica que la respuesta sea correcta cuando se intenta crear un usuario que ya existe. Y que cuando se inserta una contraseña demasiado corta la respuesta sea la correcta. 

Corremos el test y deberiamos tener el error de que no hay respuesta para url ya que esta no fue creada. 

### Vamps a implementar la create_user API

> Primero creamos el serializer:

```python
from django.contrib.auth import get_user_model

from rest_framework import serializers


class UserSerializer(serializers.ModelSerializer):
    """Serializer for the users objects"""

    class Meta:
        model = get_user_model()
        fields = ('email', 'password', 'name')
        extra_kwargs = {'password': {'write_only': True, 'min_length': 5}}

    def create(self, validated_data):
        """Create a new user with encrypted password and return it"""
        return get_user_model().objects.create_user(**validated_data)
```


> Seguido tenemos que modificar el archivo views.py:

```python
from rest_framework import generics

from user.serializers import UserSerializer


class CreateUserView(generics.CreateAPIView):
    """Create a new user in the system"""
    serializer_class = UserSerializer
```

Para esto estamos usando modulo generics y este nos permite crear una view muy rapidamente utilizando lo que provee rest_framework solo con el serializer que creamos anteriormente.


> Tenemos que crear la url correspondiente a create_user

Creamos un nuevo archivo urls.py: 

```python
from django.urls import path

from user import views


app_name = 'user'

urlpatterns = [
    path('create/', views.CreateUserView.as_view(), name='create'),
]
```

> Tenemos que linkear la url creada a las urls de la app principal entonces en \recipe-app-api\app\app\urls.py

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/user/', include('user.urls')),
]
```

Para esto usamos la helper function include() para pasar las url como un string.

Con esto ya tenemos una API funcional para crear usuarios.

### Ahora vamos a crear la API para generar un nuevo token.

Con esta API a partir de un HTTP post vamos a generar un token temporal para usar despues para validar otros request que requieren autenticación. Esto sirve para que no tengamos que estar enviando le usuario y contraseña cada vez que hacemos un request. 

Vamos a generar los tests necesarios en el archivo test_user_api:

Agregamos los siguientes tests:

```python
    def test_create_token_for_user(self):
        """Test that token is created for the user"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'testpass'}
        create_user(**payload)
        res = self.client.post(TOKEN_URL, payload)

        self.assertIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_200_OK)
    
    def test_create_token_invalid_credentials(self):
        """Test that token is not created if invalid credentials are given"""
        create_user(email='test@mainqlabsdev.com', password='testpass')
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'wrongpass'}
        res = self.client.post(TOKEN_URL, payload)

        self.assertNotIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_create_token_no_user(self):
        """Test that token is not created if user doesn't exists"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'testpass'}
        res = self.client.post(TOKEN_URL, payload)

        self.assertNotIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_create_token_missing_field(self):
        """Test that email and password are required"""
        res = self.client.post(TOKEN_URL, {'email': 'one', 'password': ''})
        self.assertNotIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

```
tenemos que agregar la URL TOKEN_URL = reverse('user:token') a la que ya teniamos generados. 

No hay muchas cosas que aclarar en estos tests la mayoria esta explicito en los nombres de las funciones. No se testea que el token generado este bien se confia en que django se encuentra funcionando bien. 

Corremos el test y deberiamos tener el error de que el reverse para token no se encuentra ya que todavia no implementamos la función. 

### Tenemos que implemntar create_token API:

> Primero agregamos el serializer: 

```python
from django.contrib.auth import get_user_model, authenticate
from django.utils.translation import ugettext_lazy as _

from rest_framework import serializers


class UserSerializer(serializers.ModelSerializer):
    """Serializer for the users objects"""

    class Meta:
        model = get_user_model()
        fields = ('email', 'password', 'name')
        extra_kwargs = {'password': {'write_only': True, 'min_length': 5}}

    def create(self, validated_data):
        """Create a new user with encrypted password and return it"""
        return get_user_model().objects.create_user(**validated_data)


class AuthTokenSerializer(serializers.Serializer):
    """Serializer for the user authentication object"""
    email = serializers.CharField()
    password = serializers.CharField(
        style={'input_type': 'password'},
        trim_whitespace=False
    )

    def validate(self, attrs):
        """Validate and authenticate the user"""
        email = attrs.get('email')
        password = attrs.get('password')

        user = authenticate(
            request=self.context.get('request'),
            username=email,
            password=password
        )
        if not user:
            msg = _('Unable to authenticate with provided credentials')
            raise serializers.ValidationError(msg, code='authentication')

        attrs['user'] = user
        return attrs
```

En este caso creamos el serializer basandonos en el modulo estandar de django serializers. 

Utilizamos la funcion authenticate es un helper command que nos permite authenticar un request pasandole el usuario y contraseña. 

importamos el modulo de translation ya que queremos mostrar un mensaje y siempre lo optimo es hacerlo a traves del motor de traducciones. 
el trim whitespace es para que quite los espacios en el inicio y final de la contraseña, ya que se permiten espacios en el ingreso de una contraseña. 

Ahora utilizamos la funcion authenticate. El primer argumento es el request que queremos validar, despues tenemos que pasar el usuario y la contraseña. 

> Creamos la vista en views.py:

```python
from rest_framework import generics
from rest_framework.authtoken.views import ObtainAuthToken
from rest_framework.settings import api_settings

from user.serializers import UserSerializer, AuthTokenSerializer


class CreateUserView(generics.CreateAPIView):
    """Create a new user in the system"""
    serializer_class = UserSerializer


class CreateTokenView(ObtainAuthToken):
    """Create a new auth token for user"""
    serializer_class = AuthTokenSerializer
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES

```

Creamso una nueva clase basandonos en ObtainAuthToken:

Luego asignamos el serializer que creamos anteriormente  y agregamos el renderer_classes para poder ver este endpoint en el navegador. 

> Agregamos la url en urls.py dentro de la app:

```python
from django.urls import path

from user import views


app_name = 'user'

urlpatterns = [
    path('create/', views.CreateUserView.as_view(), name='create'),
    path('token/', views.CreateTokenView.as_view(), name='token'),
]
```

Esto lo hacemos igual que en el caso anterior solo agregamos la nueva url y la vista que creamos. 

**Ya podemos testear el autenticador en el navegador.**

### Lo siguiente es crear el manage user endpoint:

Este endpoint nos va a permitir modificar, borrar etc los usuarios ya creados.

> Primero los tests: 

```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from django.urls import reverse

from rest_framework.test import APIClient
from rest_framework import status


CREATE_USER_URL = reverse('user:create')
TOKEN_URL = reverse('user:token')
ME_URL = reverse('user:me')


def create_user(**params):
    return get_user_model().objects.create_user(**params)


class PublicUserApiTests(TestCase):
    """Test Users API (public)"""

    def setUP(self):
        self.client = APIClient()

    def test_create_valid_user_success(self):
        """Test creating user with valid payload is succesfull"""
        payload = {
            'email': 'test@mainqlabsdev.com',
            'password': 'testpass',
            'name': 'Test name',
        }
        res = self.client.post(CREATE_USER_URL, payload)

        self.assertEqual(res.status_code, status.HTTP_201_CREATED)
        user = get_user_model().objects.get(**res.data)
        self.assertTrue(user.check_password(payload['password']))
        self.assertNotIn('password', res.data)

    def test_user_exists(self):
        """Test creating a user that already exists fails"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'testpass'}
        create_user(**payload)

        res = self.client.post(CREATE_USER_URL, payload)

        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_password_too_short(self):
        """Test that the password must be more than 5 characters"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'pw'}
        res = self.client.post(CREATE_USER_URL, payload)

        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)
        user_exists = get_user_model().objects.filter(
            email=payload['email']
        ).exists()
        self.assertFalse(user_exists)

    def test_create_token_for_user(self):
        """Test that token is created for the user"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'testpass'}
        create_user(**payload)
        res = self.client.post(TOKEN_URL, payload)

        self.assertIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_200_OK)

    def test_create_token_invalid_credentials(self):
        """Test that token is not created if invalid credentials are given"""
        create_user(email='test@mainqlabsdev.com', password='testpass')
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'wrongpass'}
        res = self.client.post(TOKEN_URL, payload)

        self.assertNotIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_create_token_no_user(self):
        """Test that token is not created if user doesn't exists"""
        payload = {'email': 'test@mainqlabsdev.com', 'password': 'testpass'}
        res = self.client.post(TOKEN_URL, payload)

        self.assertNotIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_create_token_missing_field(self):
        """Test that email and password are required"""
        res = self.client.post(TOKEN_URL, {'email': 'one', 'password': ''})
        self.assertNotIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_retrieve_user_unathorized(self):
        """Test that authentication is requerided for users"""
        res = self.client.get(ME_URL)

        self.assertEqual(res.status_code, status.HTTP_401_UNAUTHORIZED)


class PrivateUserApiTests(TestCase):
    """Test API requests that require authentication"""

    def setUp(self):
        self.user = create_user(
            email='test@mainqlabsdev.com',
            password='testpass',
            name='name'
        )
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

    def test_retrieve_profile_success(self):
        """Test retrieving profile for logged in used"""
        res = self.client.get(ME_URL)

        self.assertEqual(res.status_code, status.HTTP_200_OK)
        self.assertEqual(res.data, {
            'name': self.user.name,
            'email': self.user.email
        })

    def test_post_me_not_allowed(self):
        """Test that POST is not allowed on the me URL"""
        res = self.client.post(ME_URL, {})

        self.assertEqual(res.status_code, status.HTTP_405_METHOD_NOT_ALLOWED)

    def test_update_user_profile(self):
        """Test updating the user profile for authenticated user"""
        payload = {'name': 'New Name', 'password': 'newpassword123'}

        res = self.client.patch(ME_URL, payload)

        self.user.refresh_from_db()
        self.assertEqual(self.user.name, payload['name'])
        self.assertTrue(self.user.check_password(payload['password']))
        self.assertEqual(res.status_code, status.HTTP_200_OK)
```

Primero agregamos el URL en la parte superior de la pagina. Luego agregamos el test en Public para verificar que devuelve el estado no autorizado cuando se intenta acceder al endpoint. 

Creamos una nueva clase para los Test para el caso que se requiera estar autenticado para realizar (ver, y modificar usuarios).

En setUp creamos el usuario, creamos un client reutilizable y luego forzamos la autenticacion del usuario.

El resto de los tests son bastante estandar, algo distinto surge en el test update: 

Cuando realizamos el request patch actualizamos el valor en la base de datos, pero el objeto sigue teniendo el mismo valor que antes, por esto tenemos que usar la helper function refresh_from_db antes de realizar las verificaciones. 

Testeamos y deberiamos tener el error ya que no tenemos implementadas las funciones. 
> Creamos el user manager endpoint: 

Vamos a usar el user serializer que ya tenemos, pero le vamso a gregar una fucion para hacer el update. Vamos a agregar una vista a partir del retrieveupdateview del modulo generics. 

generamos la vista necesaria: en views.py

```python
from rest_framework import generics, authentication, permissions
from rest_framework.authtoken.views import ObtainAuthToken
from rest_framework.settings import api_settings

from user.serializers import UserSerializer, AuthTokenSerializer


class CreateUserView(generics.CreateAPIView):
    """Create a new user in the system"""
    serializer_class = UserSerializer


class CreateTokenView(ObtainAuthToken):
    """Create a new auth token for user"""
    serializer_class = AuthTokenSerializer
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES


class ManageUserView(generics.RetrieveUpdateAPIView):
    """Manage the authenticated user"""
    serializer_class = UserSerializer
    authentication_classes = (authentication.TokenAuthentication,)
    permission_classes = (permissions.IsAuthenticated,)

    def get_object(self):
        """Retrieve and return authentication user"""
        return self.request.user
```

authentication es el mecanismo por el que ocurre la autenticacion, en este caso mediante token. 

permissions es el grado de acceso que tiene el usuario entonces lo unico que pedimos en este caso es que este autenticado. 

Luego vamos a agregar el get_object(). Normalmente en una vista lo que se hace es linkearla con el modelo como para devolver el modelo. Pero en este caso solo queremos devolver el modelo para el usuario que esta autenticado entonces tenemos que sobreescribir el metodo get_object(). cuando el get_object es llamado el request va a tener el user adjunto y esto se debe al authentication_classes que definimos anteriormente. La autentication class se encarga de autenticar el usuario y asignarselo al request. Cosas magicas de django. 

> Modificamos el serializer. 

en serializers vamso agregar el metodo update a nuestro user serializer: 

```python
from django.contrib.auth import get_user_model, authenticate
from django.utils.translation import ugettext_lazy as _

from rest_framework import serializers


class UserSerializer(serializers.ModelSerializer):
    """Serializer for the users objects"""

    class Meta:
        model = get_user_model()
        fields = ('email', 'password', 'name')
        extra_kwargs = {'password': {'write_only': True, 'min_length': 5}}

    def create(self, validated_data):
        """Create a new user with encrypted password and return it"""
        return get_user_model().objects.create_user(**validated_data)

    def update(self, instance, validated_data):
        """Update a user, setting the password correctly and return it"""
        password = validated_data.pop('password', None)
        user = super().update(instance, validated_data)

        if password:
            user.set_password(password)
            user.save()

        return user


class AuthTokenSerializer(serializers.Serializer):
    """Serializer for the user authentication object"""
    email = serializers.CharField()
    password = serializers.CharField(
        style={'input_type': 'password'},
        trim_whitespace=False
    )

    def validate(self, attrs):
        """Validate and authenticate the user"""
        email = attrs.get('email')
        password = attrs.get('password')

        user = authenticate(
            request=self.context.get('request'),
            username=email,
            password=password
        )
        if not user:
            msg = _('Unable to authenticate with provided credentials')
            raise serializers.ValidationError(msg, code='authentication')

        attrs['user'] = user
        return attrs
```

agregamos la funcion update debajo de la create: La principal finalidad de este es asegurarnos que la password es actualizada a traves de la set_password function en lugar de hacerlo a traves de la password que se esta dando. 

El argumento instance es el instance del modelo que es linkeado a nuestro model serializer. Lo primero que hacemos es guardar la password y eliminarla de la lista validated_data, en lugar de solo hacer un get es mejor usar el pop. 

A traves del super() hacemos un llamado al metodo update establecido por default en el modelserializer. Con esta podemos hacer uso de toda la funcionalidad de la clase por default y agregando algunas funcionalidades adicionales. 

Por ultimo hacemos el update de la contraseña y guardamos. 

> Tenemos que agregar la url:

```python
from django.urls import path

from user import views


app_name = 'user'

urlpatterns = [
    path('create/', views.CreateUserView.as_view(), name='create'),
    path('token/', views.CreateTokenView.as_view(), name='token'),
    path('me/', views.ManageUserView.as_view(), name='me')
]
```

Igual que en los casos anteriores. 

### Vamos a crear la recipe app

Para eso corremos al igual que antes el comando: 

```
docker-compose run --rm sh -c "python manage.py startapp recipe"
```

Luego borramos algunos archivos que no vamos a utilizar porque usaremos los que ya tenemos en la app core. 

> delete:
> - models.py
> - tests.py
> - admin
> - migrations folder

Creamos una nueva carpeta tests y agregamos el __init__.py 

Y agregamos la nueva appp al settings.py.

### Ahora vamos a crear un nuevo modelo para la base datos para agregar las tags

Primero creamos una helper function que nos permita crear facilmente un nuevo usuario para nuestros tests: 
En la parte superior de test_models.py en core app:


```python
def sample_user(email= 'test@mainqlabsdev.com', password='testpass'):
    """Create a sample user"""
    return get_user_model().object.create_user(email, password)
```

Siguiente a esto agregamos el test en test_models.py donde vamos a verificar que la representacion de la tag creada sea la correcta, en las ultimas lineas agregamos el test requerido:

```python
    def test_tag_str(self):
        """Test the tag string representation"""
        tag = models.Tag.objects.create(
            user=sample_user(),
            name='Vegan'
        )

        self.assertEqual(str(tag), tag.name)
```

Estamos verificando que al convertir la Tag a string no des de el nombre. 

Corremos el test y deberia fallar ya que no creamos el Tag model aun:

Vamos a crear el modelo entonces en models.py:

```python
from django.conf import settings # En la parte de los imports


class Tag(models.Model):
    """Tag to be used for a recipe"""
    name = models.CharField(max_length=255)
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
    )

    def __str__(self):
        return self.name
```

Lo nuevo aqui que para asignar la foreing key user no referenciamos el user method directamente sino que lo importamos del django.conf settings. Que es la forma recomendada de traer cosas desde el settings. 

Una vez que tenemos el modelo creado vamos a realizar el migrate:

```
docker-compose run app sh -c "python manage.py makemigrations"
```

Volvemos a correr el test y ya debiera estar todo ok. 

### Vamos a crear una app para la lista de tags: 

en test_tags_api.py : 

```python
from django.contrib.auth import get_user_model
from django.urls import reverse
from django.test import TestCase

from rest_framework import status
from rest_framework.test import APIClient

from core.models import Tag

from recipe.serializers import TagSerializer


TAGS_URL = reverse('recipe:tag-list')


class PublicTagsApiTests(TestCase):
    """Test the publicly available tags API"""

    def setUp(self):
        self.client = APIClient()

    def test_login_required(self):
        """Test that login is required for retrieving tags"""
        res = self.client.get(TAGS_URL)

        self.assertEqual(res.status_code, status.HTTP_401_UNAUTHORIZED)


class PrivateTagsApiTests(TestCase):

    def setUp(self):
        self.user = get_user_model().objects.create_user(
            'test@mainqlabsdev.com',
            'testpassword'
        )

        self.client = APIClient()
        self.client.force_authenticate(self.user)

    def test_retrieve_tags(self):
        """Test Retrieving Tags"""
        Tag.objects.create(user=self.user, name='Vegan')
        Tag.objects.create(user=self.user,name='Dessert')

        res = self.client.get(TAGS_URL)

        tags = Tag.objects.all().order_by('-name')
        serializer = TagSerializer(tags, many=True)
        self.assertEqual(res.status_code, status.HTTP_200_OK)
        self.assertEqual(res.data, serializer.data)

    def test_tags_limited_to_user(self):
        """Test that tags returned are for the authenticated user"""
        user2 = get_user_model().objects.create_user(
            'other@mainqlabsdev.com',
            'testpass2'
        )
        Tag.objects.create(user=user2, name='Fruity')
        tag = Tag.objects.create(user=self.user, name='Comfort Food')

        res = self.client.get(TAGS_URL)

        self.assertEqual(res.status_code, status.HTTP_200_OK)
        self.assertEqual(len(res.data),1)
        self.assertEqual(res.data[0]['name'], tag.name)
```

TAGS_URL = reverse('recipe:tag-list') lo que hace esta parte es ya sabiendo como se va a llamar la url la vamos a buscar a la app correspondiente. 

Primero creamos los tests para la parte publica:

Y lo unico que testamos es que cuando el ususario no esta registrado devuelva un error indicando que los tags solo los puede ver un usuario registrado. 

Para la parte Privada:

Lo primero que hacemos es crear un usuario y forzar su autorizacion en el setUP()

Primero testeamos que devuelva las tags cuando hacemos el llamado a la API. En este caso cuando llamamos al serializer tenemos que indicarle el parametro MAny ya que nosotros vamos a estar llamando a varios objectos. 

Verificamos que el status code sea el correcto y despues verificamos que la data del request este de acuerdo con la del serializer. 

Por ultimo verificamos que solo devuelva las tags solo de usuario registrado. 

### Vamos a implementar la feature para las tags:

> Primero creamos el serializer entonces creamos un nuevo archivo serializer.py dentro de recipe:

```python
from rest_framework import serializers

from core.models import Tag


class TagSerializer(serializers.ModelSerializer):
    """Serializer for Tag Objects"""

    class Meta:
        model = Tag
        fields = ('id', 'name')
        read_only_fields = ('id',)
```

> Cremos el views para nuestra nueva feature:

```python
from rest_framework import viewsets, mixins
from rest_framework.authentication import TokenAuthentication
from rest_framework.permissions import IsAuthenticated

from core.models import Tag
from recipe import serializers


class TagViewSet(viewsets.GenericViewSet, mixins.ListModelMixin):
    """Manage tags in database"""
    authentication_classes = (TokenAuthentication,)
    permission_classes = (IsAuthenticated,)
    queryset = Tag.objects.all()
    serializer_class = serializers.TagSerializer

    def get_queryset(self):
        """Return objects for the current authenticated user only"""
        return self.queryset.filter(user=self.request.user).order_by('-name')
```

Como en este caso no necesitamos todos los funciones create delete etc, entonces solo usamos el listmodelfunctions para eso usamos una combinacion del viewsets y el listmodelmixin
Nuevamente indicamos el authentication clases para indicar el tipo de autenticacion y luego el permission_classes para setear que es requerida la authenticacion para el request

Tenemos que pisar el get_queryset para que solo devuelva el queryset para el usuario registrado y que devuelva las tagas ordenadas en orden alfabetico. 

creamos la urls.py:

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from recipe import views

router = DefaultRouter()
router.register('tags', views.TagViewSet)

app_name = 'recipe'

urlpatterns = [
    path('', include(router.urls))
]
```

El defaultrouter es una funcional que nos crea directamnete la url para nuestro viewset, cuando tenes una viewset podes tener multiples urls asociadas a la viewset. El default router nos crea directamente las urls para todas las acciones de nuestra viewset. 

Luego lo unico uqe tenemos que hacer es registrar nuevas rutas. Luego incluimos todas las urls que coincidan con nuestra app recipe. 

Lo ultimo es relacionar las url agragadas con el urls principal:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/user/', include('user.urls')),
    path('api/recipe/', include('recipe.urls')),
]
```





















