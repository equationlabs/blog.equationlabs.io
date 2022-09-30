## Una mini API para hacer benchmark de Symfony con RoadRunner (Parte II)

> Esta es la continuación de la primera entrega que puedes leer aqui -> https://blog.equationlabs.io/una-mini-api-para-hacer-benchmark-de-symfony-con-roadrunner-parte-i 

TLDR; Hemos desarrollado una pequeña API  (```PHP``` 8.1  +  ```Symfony``` v6.1) utilizando ```RoadRunner``` como application server para realizar un pequeño benchmark de cuanta mejora tenemos al reemplazar ```Nginx``` por ```RoadRunner```

Si se preguntan porque elegí  ```RoadRunner``` en vez de otros SAPI que hacen lo mismo (```SwoolePHP```, ```ReachPHP```, etc) es porque utilizando este ultimo, la manera de desarrollar se mantiene variando un poco como manejamos las conexiones persistentes y su fácil implementación con ```Symfony``` y su componente ```Runtime```.

---

Bien, lo que tenemos ahora es una ```API``` con un solo endpoint que siempre retorna el mismo resultado, y queremos ver (a manera de benchmark) cuantos request puede manejar y en cuanto tiempo comparado con la misma API pero utilizando ```NGINX``` como convencionalmente es utilizado.


![Screenshot 2022-09-29 at 10.16.13.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664439498559/tyrfPkUw9.png align="center")

Para hacer el benchmark lo mas sencillo posible, vamos a utilizar ```Vegeta```  - para conocer mas de Vegeta puedes [mirar aquí](https://github.com/tsenart/vegeta) - 

El escenario es la misma aplicación con ```PHP+FPM+NGINX``` y por otro lado ```PHP+RoadRunner``` en ambas utilizaremos el siguiente comando de vegeta para hacer la prueba de carga:


```bash
cat src/tests/benchmark/target.txt | vegeta attack -rate 100 -duration=60s"
``` 
> Leyenda: 
> Rate: 100 Requests por Unidad (por default es 1s, con lo cual serian 20 req/s)
> Duration: 60segundos

El reporte nos generara la data para poder determinar los percentiles de respuesta de nuestros requests a lo largo de la prueba y así poder medir la latencia en ambos escenarios.


### PHP + FPM + NGINX
---
![php-fpm-vegeta.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664472152099/AC2neuAcQ.png align="left")

### PHP + ROADRUNNER
---
![Screenshot 2022-09-29 at 19.14.57.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664472165019/WF4Ll9lew.png align="left")

Si compramos la data de los histogramas resultantes de las pruebas (Vegeta permite exportar resultados que luego pueden ser graficados y cruzados con otros histogramas) nos da la siguiente escena:


![histirgram-comparison.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664517413332/CwL5xd60Y.png align="left")

EL percentil 95% parece estar muy cerca en ambos caos, pero en el caso de RoadRunner se mantiene constante a lo largo de la prueba de carga con lo cual (para el escenario de la prueba) su degradación en el tiempo es mínima (el tiempo de respuesta a los usuarios se mantiene estable) 

Hasta acá mi ensayo de prueba de carga utilizando PHP + RoadRunner, espero les haya gustado y si es así pues dejame tu comentario o like en el post.
