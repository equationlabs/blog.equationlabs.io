## Cómo mantener tus secrets de k8s de manera segura en  Git

Hace poco escuche una excelente charla virtual de [JuanJo Ciarlante](https://github.com/jjo) sobre este tema y me pareció excelente plasmarlo en un tutorial corto y disponible para quien quiera implementarlo en sus proyectos de manera de poder mantener estos datos sensibles dentro de tu repo sin necesidad de software o aplicaciones adicionales para manejar estos datos.

## ¿Cual es el problema con Secret de K8s?
El problema principal de los `Secrets` genéricos creados a través de `kubectl`, es la formula que utliiza para "asegurar" dicha secret, y es un simple ofuscado con `base64` para poder ser guardado de manera eficiente en su base de datos `etcd` donde `kubernetes` guarda todos estos valores.

Un ejemplo de objeto `Secret` de `kubernetes` seria el siguiente

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
``` 

Ahora imagina que al ser ofuscados únicamente con un `base64` cualquier developer con acceso al repositorio podría fácilmente conocer el valor  **(y obviamente nadie quiere eso por razones obvias)**:

```bash
echo MWYyZDFlMmU2N2Rm | base64 --decode
1f2d1e2e67df
``` 

## Sealed Secrets como solución
Sealed Secret (menos potente que Helm Secrets pero bastante robusta) se encarga de `encriptar` tu `Secret` genérica actual de manera asimétrica que solo el controlador dentro del cluster sabe `desencriptar` (mediante llave publica/privada) con lo cual es bastante seguro guardar los secretos encriptados en tu repositorio ya que nadie mas tiene acceso a la llave privada dentro del cluster (mas que el equipo de seguridad posiblemente y esto esta bien).


La utilidad de `Kubeseal` se compone de dos partes:

- Un agente que se instala en el cluster
- Una utilidad cli `kubeseal` que te permitira encriptar la `Secret` genérica como tal


Instalamos el controlador/agente en el cluster

```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/controller.yaml
``` 

Luego instalamos la utilidad del lado del cliente

```
brew install kubeseal
``` 

Luego debemos extraer la calve publica desde el cluster de la siguiente manera (esta se puede guardar de manera segura en el cluster y que solo se utiliza para encriptar/firmar) y luego se puede proceder a encriptar la secret genérica de `k8s`


```
kubeseal --fetch-cert > mycert.pem
kubeseal < mysql.secret.yaml > mysql.sealedsecret.json
``` 

El nuevo archivo contiene un `Custom Resourced Definition (CRD)` de kubernetes como el siguiente:

```
{
  "kind": "SealedSecret",
  "apiVersion": "bitnami.com/v1alpha1",
  "metadata": {
    "name": "mysql-credentials",
    "namespace": "default",
    "creationTimestamp": null
  },
  "spec": {
    "template": {
      "metadata": {
        "name": "mysql-credentials",
        "namespace": "default",
        "creationTimestamp": null
      },
      "type": "Opaque",
      "data": null
    },
    "encryptedData": {
      "password": "AgCcnJundAHMO2gl0LGd965+8W2CaJAW6DkoJR....."
    }
  }
}
``` 

Este archivo puede ser utilizado para crear una `SealedSecret` en tu cluster, y ya que el controlador esta permanente escuhando por recursos `SealedSecret`, cuando encuentra, uno procede a desencriptarlo con la llave privada que solo reside dentro del cluster y convertirlo en un objecto `Secret` genérico de `k8s` y aplicarlo automáticamente al cluster.


```
kubectl apply -f  mysql.sealedsecret.json
``` 

Como la clave privada solo reside dentro del cluster, todas tus secrets generadas con `kubesea`l pueden residir sin problemas dentro de tu repo ya que serán casi imposiblees de desencriptar, puedes dormir un poco más tranquilo ahora.

### Hay algunas desventajas de `Kubesea`l:
- La clave privada debe estar bien resguardad y respaldada varias veces (con algún proceso automatizado y guardado en otro vault) si pierdes esta llave no podrás desencriptar de nuevo tus secrets
- Como no puede leer lo que esta dentro de la `SealedSecret` si quisieras cambiar el valor tendrás que reencriptar de nuevo y volver aplicarla al cluster
- La clave privada reside dentro del cluster si mayores seguridades que las proporcionadas por el entorno, proveedor, etc, que aunque son buenas capaz no es lo más seguro posible.

-----
Enlaces útiles
- EL repositorio de Sealed Secrets  [https://github.com/bitnami-labs/sealed-secrets](https://www.youtube.com/watch?v=iGOO3pIxWNc) 
- La charla de Juanjo Ciarlante ->  [https://www.youtube.com/watch?v=iGOO3pIxWNc](https://www.youtube.com/watch?v=iGOO3pIxWNc) 



