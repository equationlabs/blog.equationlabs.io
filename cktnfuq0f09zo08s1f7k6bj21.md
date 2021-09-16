## Jugando con Jenkins y sus pipelines declarativos (y aprovechamos para conocer Blue Ocean)

> Puedes ver la construcción de un  [pipeline completo de CI/CD con GitLab, Kaniko, Skaffold, Terraform y Kubernetes en mi otro post](https://blog.equationlabs.io/trunk-based-development-kubernetes-y-gcp-un-pipeline-con-la-ayuda-de-gitlab-kaniko-skaffold-y-terraform) y como hacer un circuito completo de GitOps donde tu repositorio es la fuente de la verdad para toda tu operación

Me encanta utilizar **GitLab** y su motor de **CI/CD**, es muy sencillo y fácil de utilizar para cualquier desarrollador, su curva de aprendizaje es bastante baja y viene "out of the box" con todas las licencias (gratis o pagas) de GitLab, pero aparta esta declaración de amor incondicional por GitLab, siempre hay espacio para aprender de otras herramientas, hoy le toca el turno a **Jenkins**.

Hace poco (mas bien hace una año y monedas) Jenkins introdujo un plugin llamado **Blue Ocean**, una especie de *"revamping"* de UI/UX para darle a **Jenkins** la posibilidad de armar pipelines a través de una interfaz gráfica 

Requisitos:
- Un repositorio n GitLab con un proyecto sencillo en NodeJS
- Una instancia de Jenkins con el plugin de Blue Ocean activado

## Lenguaje Declarativo
Lo primero es en nuestro repositorio de GitLab, crear un archivo con el nombre *jenkinsfile* (a manera del .gitlab-ci.yaml) donde estará la especificación de los pasos y tareas de nuestro pipeline en **jenkins**


```
pipeline {
  agent any
  stages {
    stage('Git') {
      steps {
        echo 'clone your repo here'
      }
    }
    stage('Build') {
      steps {
        echo 'build your applications and save your artefacts in this step'
      }
    }
    stage('Test') {
      steps {
        echo 'Run your app in preparation for tests'
        echo 'Run your test in this step'
      }
    }
    stage('Deploy') {
      steps {
        echo 'Deploy your stuffs'
      }
    }
  }
}

``` 

Cómo verás, es totalmente "straightforward" el lenguaje utilizado para declarar el pipeline, lo que hace de su entendimiento para cualquier desarrollador de algo sencillo. Push & Commit antes del siguiente paso 💪 .

Ahora configuremos Jenkins, enlazándolo con nuestro repositorio, para que sepa dónde encontrar la directriz del pipeline a través de la UI ofrecida por el plugin de Blue Ocean de la gente de Jenkins. ( [Yo monte una instancia local de Jenkins con la imagen oficial de DockerHub](https://hub.docker.com/r/jenkins/jenkins) )

En este video les muestro los pasos en la configuración del pipeline en Jenkins tomando como base el archivo **Jenkinsfile** en el root del repositorio.


%[https://www.loom.com/share/acc102a1aef04927b7e43e22307c5d57]


![Screenshot 2021-09-16 at 20.44.52.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1631817924966/15wr1n4A6.png)

Aquí ya tienes un pipeline funcional en Jenkins directamente desde tu repositorio, totalmente declarativo, ahora agreguémosle un par de twist mas para correr realmente los test de la aplicación **NodeJS** que esta dentro del repositorio que acá les comparto.

Ahora nuestro **Jenkinsfile** quedaría algo como esto:

```
pipeline {
  agent any

  tools {
    nodejs 'node'
  }

  environment {
    CI = 'true'
  }

  stages {
    stage('Git') {
      steps {
        echo 'Repository already cloned'
      }
    }

    stage('Install Dependencies') {
      steps {
        sh 'npm install'
      }
    }

    stage('Test') {
      steps {
        sh 'npm start'
        sh 'npm test'
      }
    }

    stage('Deploy') {
      steps {
        echo 'Deploy your stuff on any cloud'
        echo 'Here is not covered yet in this tutorial so we need another time'
      }
    }
  }
}
``` 

Una vez comiteado a tu repo deberias poder ver algo asi en tu dahsboard de Jenkins / Blue Ocean


![Screenshot 2021-09-16 at 23.17.48.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1631827097424/XrJ8CePf8.png)


![Screenshot 2021-09-16 at 23.17.28.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1631827112453/x6biOeUqb.png)



----
%[https://gitlab.com/rcastellanosm/nodejs-and-jenkins-declarative-pipelines]
