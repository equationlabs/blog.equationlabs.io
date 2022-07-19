## Una mini API para hacer benchmark de Symfony con RoadRunner (Parte I)

PHP ha avanzado mucho desde sus inicio, y eso incluye un ecosistema que ahora incluye poderosos application servers cómo lo es RoadRunner. Este último es un application server, load balancer y process manager hecho en GoLang, que utilizando **GoRoutines** y **multithreading**, mantiene la aplicación PHP en memoria entre requests eliminando la necesidad de boot loading y code loading process reduciendo así, la latencia de tu aplicación pudiendo servir mas requests en menos tiempo. *(performance at it's best)*

Algunas "features" interesantes de RoadRunner:

   - Production-ready PSR-7 compatible HTTP, HTTP2, FastCGI server
   - Agnóstico el framework de trabajo
   - Servidor de métricas integrado (prometheus)
   - Integraciones para Symfony, Laravel, Slim, CakePHP, Zend Expressive, Spiral
   - y mas...

> Es importante recalcar que existen variedad de Runtimes que pueden ser utilizados con PHP (Swoole, ReactPHP, Bref y hasta el siempre conocido PHP-FPM) en nuestro caso vamos a utilizar RoadRunner por su simplicidad de instalación y conociendo de antemano que Swoole es capaz de rendir mas performance hoy en día que RoadRunner

Para probar un poco como funciona, vamos a implementar una pequeña API (utilizando un poco de CQRS) con un solo endpoint, que nos permita, recibir un request, ir a otra api externa (tipo clima), recibir la respuesta, transformarla y retornarla al cliente final.


Acá un pequeño diagrama de secuencia para entender el flujo de la misma
![sequence-diagram-api.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658159844888/jAGpW1FPk.png align="left")

> NOTA: Todo esto está disponible en el README del repositorio que esta al final de este post.

Utilizar RoadRunner con Symfony es bastante sencillo, recuerda que no necesitaras ni Nginx ni FPM para esto, solo el binario de RoadRunner, tu código fuente y sus dependencias declaradas en el archivo de composer por lo tanto el setup es muy simple, en mi caso el Dockerfile es el siguiente:


```bash
FROM spiralscout/roadrunner:2.10.5 as rr

FROM php:8.1 as php
RUN apt-get update && apt-get install -y libzip-dev unzip bash

# Copy Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Source Code
ADD . .
RUN composer install -o

# Copy RoadRunner
COPY --from=rr /usr/bin/rr /usr/bin/rr
``` 

RoadRunner necesita de su archivo de configuración ***.rr.yaml*** para funcionar, este archivo en mi caso, incluye la clase del lado de symfony que servirá de entrypoint (esta delcarado como una variable de ambiente llamada APP_RUNTIME).

Para poder hacer funcionar RoadRunner con Symfony es necesario hacer uso del componente symfony/runtime e instalar el runtime adecuado, que en nuestro caso será [https://github.com/php-runtime/roadrunner-symfony-nyholm ](php-runtime/roadrunner-symfony-nyholm). 

> El componente symfony/runtime desacopla la lógica de boostraping de cualquier estado global para asegurarse que la aplicación pueda ejecutarse con variedad de runtimes como PHP-PM, ReactPHP, Swoole, RoadRunner, etc. sin ningún cambio en tu aplicación. para conocer mas puedes ver la documentación oficial aqui: https://symfony.com/doc/current/components/runtime.html

Nuestro archivo de configuración de RoadRunner (para este caso practico) es el siguiente:

```yaml
version: "2.7"

server:
  command: "php public/index.php"
  env:
    - APP_RUNTIME: Runtime\RoadRunnerSymfonyNyholm\Runtime

http:
  address: 0.0.0.0:8080
  middleware: [ "gzip" ]
  pool:
    num_workers: ${RR_NUM_WORKERS}
    max_jobs: ${RR_MAX_JOBS}
    supervisor:
      max_worker_memory: ${RR_MAX_WORKER_MEMORY}

metrics:
  address: 0.0.0.0:2112

logs:
  mode: production
  channels:
    http:
      level: error 
    server:
      level: error
      mode: raw
    metrics:
      level: error

``` 
> Puedes encontrar mas detalles sobre como configurar RoadRunner para cada ambiente (dev, debug, production, etc) en el siguiente enlace https://roadrunner.dev/docs/intro-config/2.x/en

Adicionalmente en tu docker-compose debes indicar en el command de inicialización donde esta el binario de RoadRunner y que archivo de configuración quieres utilizar, por ejemplo debería quedar algo como esto:

```yaml
php:
    build:
      dockerfile: Dockerfile
      context: .
    ports:
      - "8080:8080"
      - "2112:2112" # para exponer las métricas del servidor embebido de prometheus que traer RR
    env_file:
      - .env
    working_dir: /opt
    volumes:
      - ./:/opt
    command: [ '/usr/bin/rr', 'serve', '-c', '.rr.yaml' ]
``` 

Una de las cosas que me gusta de RoadRunner es que trae por default un servidor de Prometheus embebido, con lo cual automáticamente expone métricas para ser consumidas por un prometheus collector de manara muy sencilla en **http://{host}:2112/metrics** y ademas te permite a través de una cómoda interfaz agregar tus métricas custom de aplicación utilizando el mismo servidor sin necesidad tener que instalar librerías adicionales en tu aplicación o un servidor adicional de prometheus para exponer métricas.

> Como plus handy, en la documentación oficial tienes un dashboard de grafana para poder monitorear toda tu aplicación que se este ejecutando sobre RoadRunner (workers, consumo de CPU, etc)


![Screenshot 2022-07-19 at 09.00.20.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658214028357/9S7kT0v4z.png align="left")


Con esto ya tenemos la primera parte para poder comenzar nuestro pequeño benchmarking.

En la segunda entrega de este post, vamos a ir directo a las diferentes ejecuciones del benchmark comparando PHP-FPM contra RoadRunner utilizando una herramienta de HTTP Load llamada Vegueta. (puedes también utilizar Apache Bench o WRK tool) 

> Puedes conseguir mas información de Vegeta en este enlace https://github.com/tsenart/vegeta


¡Hasta el próximo post!
