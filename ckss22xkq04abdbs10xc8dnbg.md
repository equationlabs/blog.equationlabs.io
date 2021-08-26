## Tutorial: k8s desde dev hasta prod con Gitlab, Skaffold, Kustomize y Kaniko


### La prueba de concepto

![ci-ce-release-cycle.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1629916807662/y4QaYpwPd.jpeg)

Vamos a trabajar con lo siguiente:
- Repositorio en `GitLab` con una aplicación de `Symfony` default
- Cluster activo de `Kubernetes` puedes tenerlo en `AWS` o `GCP` y localmente con utilizar en Mac `Docker-Desktop` o `minikube`.
- `kubectl` CLI oficial de kubernetes para interactuar con tu cluster
- `Skaffold` para manejar el ciclo de push, deploy de tu pipeline
- `Kaniko` para manejar el ciclo de build de tus imágenes

### Instalando todos los requisitos
Basado en MacOS sin embargo en los sitios de cada herramienta esta bien detallado cómo instalar para los demás sistemas operativos.

* **kubectl**: En tu terminal ejecuta `brew install kubectl` y luego para verirficar que esta correctamente instalado ejecuta `kubectil version --client=true`
* **Skaffold**: de igual manera en tu terminal ejecuta `brew install skaffold` 

### Ahora: ¡Manos a la obra!
Manejaremos la siguiente **estructura de archivos** para este tutorial

````
project_root:
api: # Archivos de Symfony y Dockerfile
deployment: # Archivos de CI/CD
    base:
        configs:
            nginx:
                default.conf
        _registry-secret.yaml
        api-app.yaml
        api-load-balancer-service.yaml
        kustomization.yaml
    environments:
        dev:
            api-app.patch.yaml
            kustomization.yaml
   skaffold.yaml
   .build-template.json
.gitlab-ci.yaml
````

`Skaffold`, es una herramienta de lineas de comandos desarrollada por `Google`, que hace el desarrollo local con `k8s` un soplo, teniendo en cuenta lo complicado que es manejar todos los manifiestos yaml de `k8s`,  buildear, pushear, etc, esta herramienta funciona a modo de `hot reloading` para desarrollo local, con lo cual ante cualquier cambio de tu código, **buildea, rearma los yamls y deploya** para que puedas seguir desarrollando sin necesidad de trabajos manuales sobre los yamls.

`Kustomize` es una herramienta de comandos que ahora viene incluida por default con `k8s` y que permite poder **"templatizar"** tus manifiestos de  `k8s ` y permitirte así poder a partir de un manifiesto base **"patchear"** manifiestos  `k8s` para cada uno de tus ambientes.

Veamos a ahora como es nuestro archivo de configuración de `Skaffold`:

```
apiVersion: skaffold/v2beta21
kind: Config
metadata:
  name: api-app
build:
  artifacts:
    - image: api
      context: api
      docker:
        dockerfile: Dockerfile
      sync:
        infer:
          - '**/*.php'
          - '**/*.js'
profiles:
  - name: development
    build:
      local:
        push: false
    deploy:
      kubeContext: docker-desktop # or your local k8s cluster context like minikube
      kustomize:
        paths:
          - deployment/k8s/environments/dev
```

En este paso, ya lo que nos queda es armar nuestros manifiestos basados en los servicios y contenedores que nuestro aplicativo necesite, y junto con `kustomize` ir modificando las variaciones para cada `stage`, en el repositorio podrán ver un poco mas en detalle como están construidos estos manifiestos.

Luego de esto podríamos hacer el primer intento de correr localmente skaffold y hacer cambios en nuestro código y con esto ya tendremos nuestro ciclo de desarrollo local completo.

```bash
{▸} ~ skaffold dev -p development -f deployment/skaffold.yaml
``` 

### Ahora: Pipelines en GitLab y cluster en Google Cloud

El paso más importante en esta etapa es construir nuestro archivo `.gitlab-ci.yml` que es el encargado de orquestar todos los pasos a ejecutarse en nuestro pipeline, en este caso serán 4 pasos (jobs) para este tutorial:

- `Test` test suites ejecutadas por `PHPUnit` 
- `Build` con Kaniko construimos las imágenes, las tagueamos y las enviamos a nuestro registro de imágenes (utilizaremos el mismo de gitlab en este caso), `Kaniko` se encarga de guardar una capa de cache en el mismo registro.
- `Deploy` con `Skaffold` se construyen dinámicamente los manifiestos junto con las imágenes construidas en el paso anterior generando un manifiesto final que es aplicado en el cluster
- `Destroy` con Skaffold se puede también eliminar del manifiesto aplicado actualmente.

Luego de esto si debemos estar seguros que nuestro cluster de K8s esta integrado al proyecto en GitLab, en realidad es un paso bastante sencillo y se puede ver detallado  [aquí](https://docs.gitlab.com/ee/user/project/clusters/add_existing_cluster.html) 

Una vez integrado tu cluster en tu proyecto de GitLab, el paso `deploy` con `skaffold` del pipeline será capaz de promover el manifiesto k8s directo en tu cluster de Google Cloud o de Amazon Web Services.


----
Enlaces a recursos

- Kaniko ->  [https://github.com/GoogleContainerTools/kaniko](https://github.com/GoogleContainerTools/kaniko) 
- Skaffold ->  [https://skaffold.dev/](https://skaffold.dev/) 
- K8s CLI aka kubectl ->  [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/) 

----
%[https://gitlab.com/rcastellanosm/gitops-example-with-kaniko-and-skaffold]

