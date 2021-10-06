## Trunk Based Development, Kubernetes y GCP: Un pipeline con la ayuda de Gitlab, Kaniko, Skaffold y Terraform


> Esta es la segunda entrega del articulo  [k8s desde dev hasta prod con Gitlab, Skaffold, Kustomize y Kaniko](https://blog.equationlabs.io/k8s-desde-dev-hasta-prod-con-gitlab-skaffold-kustomiza-y-kaniko) donde arme el skeleton base para esta segunda entrega, al igual que el anterior al final del artículo estará el enlace al repositorio con todo el código 

### Premisa
Ya tenemos un pipeline que cumple con las siguientes pasos: 
- `Test` test suites ejecutadas por `PHPUnit` 
- `Build` con `Kaniko` construimos las imágenes, las tagueamos y las enviamos a nuestro registro de imágenes (utilizaremos el mismo de gitlab en este caso), `Kaniko` se encarga de guardar una capa de cache en el mismo registro.
- `Deploy` con `Skaffold` se construyen dinámicamente los manifiestos junto con las imágenes construidas en el paso anterior generando un manifiesto final que es aplicado en el cluster
- `Destroy` con Skaffold se puede también eliminar del manifiesto aplicado actualmente.

Lo que queremos ahora es, eliminar un stage `Destroy`, y agregar un par de pasos más para aprovisionamiento de infraestructura con `Terraform`, el ciclo quedaría más o menos así:


![emr-base.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1630105679957/PyC35fH_1.jpeg)

## Vamos a ello
Como esta es la  [continuación de la primera parte](https://blog.equationlabs.io/k8s-desde-dev-hasta-prod-con-gitlab-skaffold-kustomize-y-kaniko) , no los voy a aburrir en la creación del pipeline, sino que nos enfocaremos en la mejora del del mismo y de los archivos de instrucciones de `Terraform`.

### Estructura de Archivos

Vamos a crear una nueva carpeta `infrastructure` y una sub carpeta `terraform` a nivel del root dir, donde irán colocados todos los archivos de `terraform`

Luego de esto vamos a crear los archivos declarativos para aprovisionar un cluster en nuestra infraestructura (en mi caso estoy utilizando `Google Cloud`). La idea principal es que adicionalmente a los `clusters permanentes (staging y producción)` estos archivos de terraform se encarguen de crear `cluster temporales` que permitan que los equipos cuenten dinámicamente con un cluster para probar sus features y que este mismo cluster temporal pueda eliminarse una vez se haga el merge de esa branch a la rama main del repositorio.

Para esto vamos a crear los archivos de `terraform` y aparte vamos a colocar un `webhook` en `GitLab`  al momento del merge request para eliminar dicho cluster temporal y el environment temporal de gitlab.


### Archivos de Skaffold y Kustomize para ambientes `review`
En un ínterin, como estamos utilizando `skaffold` para los ciclos de CI/CD, vamos a crear rápidamente un nuevo profile llamado `review` para nuestras aplicaciones de ambientes temporales/dinámicos para el manejo de los manifiestos de `kubernetes` para esto podemos hacer una copiar del folder `staging` que esta dentro de `deployments/k8s/environments` teniendo en cuenta cambiar los keys de `staging` a `review`

![Screen Shot 2021-08-30 at 14.27.04.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1630344435498/k94KHqj1f.png)

### Vamos con Terraform
En lugar de hacer el típico archivo `main.tf` en este caso y para mejor visualización vamos a separarlos por su función. 

Empecemos con el `0-provider.tf` donde, como su nombre lo indica, vamos a generar las instrucciones para que `terraform` sepa con que proveedor de servicios en la nube estamos trabajando (en nuestro caso `Google Cloud`), previamente necesitarás haber creado previamente tus credenciales de service account para que puedas comunicarte con tu proveedor (**GCP -> IAM & Admin > Service Accounts**, y luego hacer click en **Create Service Account**.)

Los valores `var.*` serán definidos mas adelante en un archivo de `terraform` conocido como `tfvars` esto nos permitirá en ese archivo escribir las variables dinámicas antes de inicializar el plan y aplicarlo.

```
# Google Cloud Platform Provider
# https://registry.terraform.io/providers/hashicorp/google/latest/docs
provider "google" {
  credentials = file("../configs/service-account.json")

  project     = var.project_id
  region      = var.region
}

terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
      version = "3.5.0"
    }
  }
}
```

En tu consola para ir probando localmente puedes ejecutar el siguiente comando para inicializar  `terraform `, esto descargara las primeras dependencias y las instalará en un folder  `.terraform ` del mismo directorio de trabajo y creará un archivo  `.lock.hcl ` que, como buena practica, deberías guardarlo en tu repo.

 ```bash
$: terraform init
 ```

Adicionalmente a esto vamos a crear un archivo `1-cluster.tf` para definir las instrucciones de `terraform` para crear un cluster en nuestra infraestructura de `Google Cloud Platform` junto con la instrucción para crear una VPC para este cluster.

```
resource "google_compute_network" "vpc_network" {
  name = var.cluster_network
}

resource "google_container_cluster" "gke-cluster" {
  name               = var.cluster_name
  network            = var.cluster_network
  location           = var.region
  initial_node_count = 1
}
```

Luego estarán los dos últimos archivos, `vars.tf` y `terraform.tfvars` dónde estarán las variables globales y sus valores y placeholders para que sean accesibles para `terraform`.

```
variable "project_id" {
  type  = string
}

variable "region" {
  type  = string
}

variable "cluster_name" {
  type  = string
}

variable "network_name" {
  type  = string
}
```

```
project_id = "YOUR_GCP_PROJECT_ID"
region     = "us-central-1c"
cluster_name = "PLACEHOLDER_CLUSTER"
network_name = "PLACEHOLDER_NETWORK"
```

Luego de esto nos queda completar nuestro pipeline de `GitLab` con los `jobs` que debemos sumar para `terraform` (incluye 3 validate, init, plan y apply) y para el `deploy` del ambiente dinámico tomando en cuenta que solo debe crear dichos `jobs` si el branch es un `feature branch` y que no sea la rama `main`. 

Si ves en detalle, hemos creado una variable de CI/CD de `GitLab` que contiene las credenciales de la `service-account` (** SERVICE_ACCOUNT_GCP**) de Google Cloud para que pueda estar accesible para el pipeline y pasarla a los jobs de terraform en forma de archivo json.

```
.terraform:
  image:
    name: hashicorp/terraform:1.0.5
    entrypoint:
      - '/usr/bin/env'
      - 'PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
  before_script:
    - cd infrastructure && mkdir -p configs && cd configs
    - echo $SERVICE_ACCOUNT_GCP | base64 -d > service-account.json
    - cd ../terraform
    - sed -i "s/\PLACEHOLDER_CLUSTER/$CI_COMMIT_REF_SLUG" terraform.tfvars
    - sed -i "s/\PLACEHOLDER_NETWORK/$CI_COMMIT_REF_SLUG" terraform.tfvars
    - terraform init
  cache:
    key: terraform-cache
    paths:
      - .terraform
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /feature/'
    - if: '$CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'

validate:
  extends: .terraform
  stage: provision
  script:
    - terraform validate

plan:
  extends: .terraform
  stage: provision
  script:
    - terraform plan -out plan.tfplan
  needs:
    - validate
  artifacts:
    paths:
      - plan.tfplan

apply:
  extends: .terraform
  stage: provision
  script:
    - terraform apply -input=false plan.tfplan
  needs:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'
      when: manual

apply:
  extends: .terraform
  stage: provision
  script:
    - cp $CI_PROJECT_DIR/planfile ./
    - terraform apply -input=false "planfile"
  needs:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'
      when: manual

deploy:feature:
  extends: .skaffold
  stage: deploy
  environment:
    name: "$CI_COMMIT_REF_NAME"
    url: $CI_COMMIT_SHORT_SHA.features.equatonlabs.io
  needs:
    - build:api
  script:
    - skaffold deploy -f deployment/skaffold.yaml -p feature -a latest-build.json --status-check=true
  rules:
    - if: '$CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'
      when: manual

destroy:feature:
  extends: .terraform
  stage: destroy
  environment:
    name: $CI_COMMIT_REF_NAME
    url: $CI_COMMIT_SHORT_SHA.features.equatonlabs.io
    action: stop
  script:
    - terraform destroy
  rules:
    - if: $CI_MERGE_REQUEST_APPROVED
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

```

### Ahora juntemos todo
Una vez tengamos todos nuestros archivos en lugar es momento de hacer nuestro `commit & push` al repositorio y seguir la ejecución del pipeline de `GitLab` y sus salidas para asegurarnos que esta todo en orden.

Tendríamos este escenario:

- Tener 2 nuevos stages `provision` y `deploy` cuando la rama comienza con `feature/` 
- Dinámicamente nuestro `cluster` y la `vpc` tendrán el nombre de la variable runtime `CI_COMMIT_REF_SLUG`  (ver mas  [acá](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) )
- En `GitLab` se creara un `environment` dinámico basado en la variable `CI_COMMIT_REF_NAME`
- Una vez aprobado el `MR` a la rama main, se dispara un `job` que elimina el cluster temporal que se utilizo para desarrollar y probar la `feature` y así **no incurrir en costos innecesarios**.
- Una vez aprobado el MR a la rama `main` se eliminará también el environment creado en `GitLab`
- En la rama `main` no debe aparecer el stage de `provision` ya que esos cluster no son permanentes en el tiempo y se manejan fuera de este `pipeline`.

**Pipeline en Features**

 
![Screenshot 2021-09-30 at 20.27.40.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633026544204/M7bt9gBHF.png)


** Environment dinámico y deploy dashboard**

![Screen Shot 2021-08-30 at 21.01.53.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1630368132018/V6MhRgpwD.png)

**Google Cloud con el Cluster Creado**

![Screen Shot 2021-08-30 at 19.45.14.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1630363525738/JRS2hQ-7X.png)

Ahora nos queda probar, para completar el circuito, que al aprobar el `MR`, se ejecute el paso de `destroy:feature` que elimine el cluster y el environment de el deploy dashboard de `GitLab`

**Pipeline con el MR aprobado**


![Screenshot 2021-10-06 at 17.35.05.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633534583973/9_dgH4g76.png)

Y esto es todo por ahora amigos, como siempre compartan, que compartir hace bien. Les dejo más abajo el repo con todos los files que vimos hoy para que los puedan ojear mas a detalles.

----

%[https://gitlab.com/equationlabs/stacks/emr/endovelicus]








