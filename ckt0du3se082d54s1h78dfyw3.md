## "Walking Skeleton" como estrategia segura para refactorizar aplicaciones legacy


> Este articulo contiene referencias a un repositorio donde estara armado el primero Walking Skeleton. Para saber como construí paso a paso dicho skeleton puedes leer mis entradas  [k8s desde dev hasta prod con Gitlab, Skaffold, Kustomize y Kaniko](https://blog.equationlabs.io/k8s-desde-dev-hasta-prod-con-gitlab-skaffold-kustomize-y-kaniko) y  [Trunk Based Development, Kubernetes y GCP: Un pipeline con la ayuda de Gitlab, Kaniko, Skaffold y Terraform](https://blog.equationlabs.io/trunk-based-development-kubernetes-y-gcp-un-pipeline-con-la-ayuda-de-gitlab-kaniko-skaffold-y-terraform) 


Creo que uno de los problemas más clásicos en empresas muy exitosas, es ¿Qué hacemos para transformar nuestro código legacy, en algo más evolucionable, que sea de rápido delivery sin que tengamos miedo de romper nada?

Ya sea que tengas un ecosistema de micro servicios o un monolito, creo que el gran reto para la mayoría de los equipos de desarrollo, arquitectura y de cloud es como poder evolucionar nuestra plataforma con total confianza, minimizando todos los riesgos posibles.

Es aquí donde llega el termino acuñado por **Nat Pryce y  Steve Freeman **en su libro **Growing Object-Oriented Software, Guided by Tests** donde acuña el término **"Walking Skeleton"** para definir una estrategia que permita evolucionar tu software minimizando el riesgo y maximizando el delivery siempre apalancándote en la construcción de tests fiables y repetibles para conducir tus desarrollos **(AKA Test Driven Development**) .



----

%[https://gitlab.com/rcastellanosm/gitops-example-with-kaniko-and-skaffold]
