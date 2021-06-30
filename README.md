# recipe-app-api

## Instalando programas necesarios:

1) Instalar docker desktop.

    a) Para windows (Se debe contar con windows pro y hyper-V habilitado):

        - Seguir proceso de instalación:
            https://docs.docker.com/docker-for-windows/install/

2) Configurar la imagen docker y la aplicacón:
    a) En Terminal ejecutar el siguiente comando en el directorio principal del repositorio:
    ```
        ~/recipe-app-api > docker build
    ```
    b) Realizar la decoraciones necesarias para el proyecto, por Terminal ejecutar el siguiente comando en el directorio principal del repositorio:
    ```
    ~/recipe-app-api > docker-compose build
    ```

3) Con esto ya podemos utilizar los commandos dentro del directorio del proyecto con el siguiente comando:
    Por ejemplo para realiar los tests:

    ```
    docker-compose run app sh -c "python manage.py test"
    ```

    - De la misma forma podemos correr cualquier comando de django como si nos encontraramos trabajando en un directorio no virtualizado.

4) Para iniciar el servidor se debe correr el siguiente comando en la terminal:

    ```
    docker-compose up
    ```

    Para utilizar la app en el navegador: 127.0.0.1:8000/




