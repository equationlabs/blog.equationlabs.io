# How to build a CI/CD workflow with Skaffold for your application (Part III)

🔥 This is third part (and last) of the series ["Full CI/CD workflow with Skaffold for your application".](https://blog.equationlabs.io/series/workflow-with-skaffold)

## Let's recap: `The Workflow`

This is the `workflow` so far:

*📣 You can check how to get to this point in the firsts two delivery of the series.*

![2fl6qCIhG.png.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1667561874111/Qhz2KHP07.jpeg align="left")

## Gitlab `K8s agent` and Security

This main part in the integration of `k8s` and `Gitlab` with the `Gitlab K8s Agent`, is, in my experience, the best and easy way I find to integrate K8s with a DevOps platform like Gitlab.

Let's recap some steps

* You need to add an Agent and them run a helm chart into your cluster to allow the secure communication between both.
    
* the agent can be configured in 2 ways:
    
    * `CI_ACCESS`: Allow access from the project repository pipeline to the cluster and then you are in charge to manage how to deploy in the cluster.
        
    * `GITOPS_ACCESS`: This allow a full gitops flow like `ArgoCD` for example, updating your cluster in a pull based way in sync with the main branch of the repository.
        

In my case a use the first one `CI_ACCESS` since I want to manage in a more granular way, the whole process with `skaffold`, so mi configuration is way simpler

I have 2 repositories in a application group, one for the micro service itself and one for the agent (the agent also could be put in the micro service repository, but if you want more granular access or share the agent/cluster between applications of the same stack, this is the best way)

![Screenshot 2022-11-09 at 09.17.42.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667982028996/VdcNVOaUU.png align="left")

So in the K8s-agents repository, we only have the declarative config.yaml file for every agent that we want to create (for this example I have 2, one for lower-envs/runner and one for production since they are 2 different clusters)

![Screenshot 2022-11-09 at 09.23.17.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667982211419/lgoKLKteB.png align="left")

And in the config itself, I give access to all the projects that I want to use in the cluster.

![Screenshot 2022-11-09 at 09.24.37.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667982282906/c40e50lXX.png align="left")

And last, bit no least, the need to link the agent with the cluster, for that, you should go to the k8s project, Kubernetes Cluster menu, and there you will see an interface where you will receive the instructions on how to link agent/cluster via a helm chart to be installed in the cluster.

![Screenshot 2022-11-09 at 09.26.58.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667982484851/8zvk9zHf8.png align="left")

![Screenshot 2022-11-09 at 09.27.05.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667982568682/jDSedEjlDe.png align="left")

After that, you cluster and gitlab instance are now linked, and you applications can use the cluster as Kubernetes-Executor runners and also for dynamic review environments (aka dynamic QA instances to say so)

### Deployment and Safety Recommendations for `K8s Agents`

To restrict access to your cluster, you can use impersonation. To specify impersonations, use the `access_as` attribute in your Agent's configuration file and use K8s RBAC rules to manage impersonated account permissions.

You can impersonate:

* The Agent itself (default) = The CI job that accesses the cluster
    
* A specific user or system account defined within the cluster
    

Impersonation give some benefits in terms of security:

* Allows you to leverage your K8s authorisation capabilities to limit the permissions of what can be done with the CI/CD tunnel on your running cluster
    
* Lowers the risk of providing unlimited access to your K8s cluster with the CI/CD tunnel
    
* Segments fine-grained permissions with the CI/CD tunnel at the project or group level
    
* Controls permissions with the CI/CD tunnel at the username or service account
    

## Provisioning cluster with `terraform`

As I said, the main goal of this tutorial is try to get the same tooling for local development, pipeline and deployment, but (always got a but), we have 2 sets of the terraform configuration instructions, of example for local development I want, as developer, to get as much of the observability tools that I have on production, in case I need to test metrics, build dashboards on grafana, etc, but without the complexity of production infrastructure architecture.

So in this case, we can give the application diagram for the first part here, as recap of 'how our local development stack`looks like, and how to achieved with`terraform \`.

[![application-diagram.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667247028660/_7nejmil9.png align="left")](http://www.plantuml.com/plantuml/png/RPDFSzem4CNl_XGg9p9juaiFcPxY6gRj55eFX4DFZ90Nq4H_t9L4OJhvxbqXrqGaXqpqPt_xtZxX1-Sv-g1LyKuQeK8BREzzvpwL9V8_Tplfzs4J7A2mneFnTyBgibFSHERM-LR9JLb_l6tYqMe-ApLt7f2ErhNLdJMHwMB_ObRz-hbwNC-g7vDbNJNJyKrHB4zKhTVJen-JW0iQy0CRrKeIDg9xnlgAppQObkDf_7JlgEBxlMDLroafk9VMi5g5A3kwON-9OOFq6E40wA11UpmHzuZ0j_9fHCj5kc7dAzBAC07evzpmNV9psKKoNifjb0QcpySwsMMlxVABoQgJ15S7BXNVI2NzYLNDDxO4F4W1iR6M0YrpwU3rBF49q2e5-4QVo3TV6_QUBInl5y4Om7ugmhYaxMH3SRJInU7Z_uW8BlQGacOBK5TvPOfeWuV8n1y88ObOJ_AmhWBlq1va2sSjUZfMBoONT9MDr7lhBT72YoJpJ7z5FWUrrU3t42BG39j8pS6Z5EwSQumWPySxv5loIeLVqkhCM2EzHMbsPMMuEdbga48Xcvd9JDW9v1qmdHIpQD9uWrY61PVoQBddh0y8YNhElWSuUa0oq_G5aPpsP_712RWsznQ2y3k0y__DqLYiIEQ63ov_im5nBvZY0KmRjFe7)

So, for this application structure, i want to get in my local environment the cluster and it's 'pre-requisites' for my architecture, understood as pre-requisites all the others components inside the cluster that no belongs to the application itself (monitoring stack, traefik, cert-manager, etc).

So for that, I write simple modules to install that dependencies inside the local cluster, and get them available to use when a run my application locally with `skaffold`.

My structure for infrastructure folder looks like this:

![Screenshot 2022-11-09 at 09.49.38.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667983890051/d2r2cjn0I.png align="left")

Every module has inside their `main.tf` configuration file setting the desired state for my cluster after it's applied.

Lets take a look for one of this module (prometheus) main file:

```bash
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.13.1"
    }
    helm = {
      source  = "hashicorp/helm"
      version = ">= 2.7.0"
    }
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = ">= 1.14.0"
    }
  }
}

resource "kubernetes_namespace_v1" "monitoring_namespace" {
  metadata {
    name = var.monitoring_stack_namespace
  }
}

resource "helm_release" "prometheus_stack" {
  name  = var.monitoring_stack_prometheus_name
  repository = "https://prometheus-community.github.io/helm-charts"
  chart = "prometheus"
  version = var.monitoring_stack_prometheus_version_number
  namespace = var.monitoring_stack_namespace
  create_namespace = false

  values = [
    file("${path.module}/manifests/prometheus-override-values.yaml")
  ]

  depends_on = [
    kubernetes_namespace_v1.monitoring_namespace
  ]
}

resource "kubectl_manifest" "prometheus_stack_ingress" {
  yaml_body = file("${path.module}/manifests/prometheus-ingress.yaml")

  depends_on = [
    helm_release.prometheus_stack
  ]
}
```

And then, in the root `main.tf` configuration file, you can wrap as many modules as you want, for my case, with my 4 modules was enough (prometheus, traefik, cert-manager, grafana)

```bash
module "cert_manager_stack" {
  source = "./module/cert-manager"
}

module "traefik_stack" {
  source = "./module/traefik"

  depends_on = [
    module.cert_manager_stack
  ]
}

module "prometheus_stack" {
  source = "./module/prometheus"

  depends_on = [
    module.traefik_stack
  ]
}

module "grafana_stack" {
  source = "./module/grafana"

  depends_on = [
    module.prometheus_stack
  ]
}
```

This also, has a handy target in our Makefile, allowing developers and operators, easily setup and remove cluster pre-requisites

![Screenshot 2022-11-09 at 09.48.58.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667984415271/B6SI0SOw1.png align="left")

![Screenshot 2022-11-09 at 09.49.06.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667984399818/Yh-hM8j4L.png align="left")

Setup the pre-requisites took about 3 minutes, but if something that you need to do, time to time, you can setup your cluster today, work on your feature for some days, and then shutdown.

After all this, you will have a fully functional Local-To-Prod pipeline. (if you need to see how the Gitlab CI file looks like, is in the second part of this series)

## Next

This is the las delivery of the series, but now I'll write about the others tools that i use to address different challenges in my day to day work.

If you are interested the next topics I'll write on are:

* Managing database migrations at scale in `Kubernetes` for `PHP` application with `symfony/migrations` component
    
* `Istio`, `Cert-Manager` and `Let's Encrypt`: Secured your `k8` clusters communication with automated generation and provisioning of SSL Certificates
    
* Internal Developer Platform: A modern way to run engineering teams.
    
* The `Digital War Room` or how to get observability for Engineering Managers across applications and teams.
    

## Support me

If you find this content interesting, please consider buying me a coffee :')