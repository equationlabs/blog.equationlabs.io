# How to build a CI/CD workflow with Skaffold for your application (Part I)

`Skaffold` (part of the `Google Container Tools` ) was on the market since 2018, but was in 2020 when (at least for me) reach a prod-grade maturity level on the tool.
 
And I was more than fascinated on how this tool not only can facilitate the developer work on local environments but also as complete pipeline from development to production environment if is used with a couple of other tools.

`Easy and Repeatable Kubernetes Development`, no matter if you are a developer, lead, platform engineer, SRE or head of Engineering, all we're agree on that 🙋🏽‍♂️. 

We want an easy, repeatable, reproducible development workflow, that brings more autonomy in the teams to bring more product value to the final user in a secure way.

I want to show you, how I use `Skaffold` as building-block for my micro service CI/CD pipeline from local to production.

## The Toolset

* `skaffold` cli  (you can use the provided docker image or install in you machine)
* ` kubectl` and ` Kustomize` (kustomize is part of kubectl cli already)
*  `K8s` cluster (local and remote) - if you use docker-desktop you've already one cluster installed by default, to use for local development.

> Isn't intended in this article shows you how to install a k8s cluster for local development, you have various alternatives like Minikube out there. Like I said, I my case since I use docker-desktop, they come with a k8s inside, so I can use that for local development, make it easier the development experience, however if you don't use docker desktop you could use any of the others alternatives available in the market.

## The Workflow 
The main idea is, to use 'skaffold' as a building block from local to production environment, simplifying the tooling used by the developer and facilitating the integration with the actual gitlab repository service.

The more complex part would be the `integration` and `functional` tests, the `unit` tests can be running through `skaffold test stage` without problems, but since integration and functional tests needs the complete application and dependencies running, a little more work needs to be done to accomplish that, however isn't as complex as sounds since I used a `Kubernetes gitlab runner`. to run the pipelines, we can use the same runner to deploy the application In a special namespace, run our tests and then remove the application from the runner.

![Screenshot 2022-10-31 at 20.17.18.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667243854951/2fl6qCIhG.png align="left")

> Be aware, that you need to do a cleanup process after each pipeline or stage runs, to avoid left running process consuming capacity and space in your Kubernetes runner and to avoid recurring in unexpected operational costs.

## The Application
It's the simplest micro service you may know, but it's intended for demonstration purposes. Was made in `Symfony` with `RoadRunner` as application server, expose metrics to a `prometheus` metrics server then this metrics are fetched from the `monitoring stack` to  be used in a `grafana` dashboard for monitoring and observability purposes.

[![application-diagram.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667247028660/_7nejmil9.png align="left")](http://www.plantuml.com/plantuml/png/RPDFSzem4CNl_XGg9p9juaiFcPxY6gRj55eFX4DFZ90Nq4H_t9L4OJhvxbqXrqGaXqpqPt_xtZxX1-Sv-g1LyKuQeK8BREzzvpwL9V8_Tplfzs4J7A2mneFnTyBgibFSHERM-LR9JLb_l6tYqMe-ApLt7f2ErhNLdJMHwMB_ObRz-hbwNC-g7vDbNJNJyKrHB4zKhTVJen-JW0iQy0CRrKeIDg9xnlgAppQObkDf_7JlgEBxlMDLroafk9VMi5g5A3kwON-9OOFq6E40wA11UpmHzuZ0j_9fHCj5kc7dAzBAC07evzpmNV9psKKoNifjb0QcpySwsMMlxVABoQgJ15S7BXNVI2NzYLNDDxO4F4W1iR6M0YrpwU3rBF49q2e5-4QVo3TV6_QUBInl5y4Om7ugmhYaxMH3SRJInU7Z_uW8BlQGacOBK5TvPOfeWuV8n1y88ObOJ_AmhWBlq1va2sSjUZfMBoONT9MDr7lhBT72YoJpJ7z5FWUrrU3t42BG39j8pS6Z5EwSQumWPySxv5loIeLVqkhCM2EzHMbsPMMuEdbga48Xcvd9JDW9v1qmdHIpQD9uWrY61PVoQBddh0y8YNhElWSuUa0oq_G5aPpsP_712RWsznQ2y3k0y__DqLYiIEQ63ov_im5nBvZY0KmRjFe7)

Now that we have all the context clear, lets begin with the next steps, first at all, i need to setup our `repository skeleton`, and `directory structure`  in order to be as functional as possible to my intended workflow and development process.

> Take into account, that this is how I setup my repositories and you should fit this to your expectations and operational workflow.

Additionally, my principal goal when I began to work with this `GitOps` approach was also, reduce the cognitive load and operational complexity for new an current colleagues, reducing also the onboarding time and the number of tools that we need to do our work.

If you need a more complex for metrics scenario, I normally try to use `thanos` for that, since allows me to easily scale `prometheus` and get `long term storage` in commonly knows cloud object storage services like `Amazon S3` or `GCP Cloud Storage`. Below you can find a diagram of the same application but implementing thanos. 

[![application-diagram-extent.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667247112713/y0Hhlyreq.png align="left")](http://www.plantuml.com/plantuml/png/ZLHDSzCm4BtxLuYSqe7M1pXqEDKudSAGW4cQ0oUF8cyIKLao-WWDJFyxkvQdYUqomodoQj--js-rkN6UMnzgbRoIMgXG0TjxtxZtQMhvhwkTzFkm2GwiCDg3zbV2r6cZk2RCfVELafiqVtTPK6YzcASrTnuiXihSr8tHX6ceVZBFldzTtvVpxCjibMV5xVGYILP7pAxBsqS_HG8NQh1ls2HN4c6Jq_q74tJ5xN7wSEtm_lErOrdJA2cubqQpN0KYdLomFmbZpxnJ2mUm3Wfh7ey8kxV0j_9XWiTbl67j5HATemHONzPSyrqKWv-B-4L8vnIZ3BabTd0jU2YJdyHbZKHKTk1IyOrKqXzPLdnY2ociSM0FKa3KtPE0PbkZ5DWNiAIY-5YmrsnfUBKCMbFhiO3sNEBdR8EzLz9Hf_HB4C757Z2F4fUW1kRq6Aq97beGlGN4H4Wv6tWpyBUnvY0h0iOHvSlP2RlkDTMfwqJXmOl8yvJqPk7tN1jNEYmhkAKPjW6sYW528diDVW_1iGLuAqKSoQY6m00NtfnLoRlG_zLfdXDwsOIj8u3HGDjX99rXemPwHRRWnPxmksMH8og2vZsctc2SiBm15kdw0ueUZtiT2XYJ9aylxbdPiNJ3N1WjiQ3Kk-6w3Ot-6S1AEBFvMmpyosJMAxApVCirnzoxU2BOYJpDD5T77wVpRDYGUTGqtHn7JgzFR2FjmSM7N77FMNpPr3AQrVlNWaSF5YKLNGRPTTl5scMzECyscnyWVEcyiNm7c3etwESzs9gjOemeLs_Jkxn8iz_1jWiRNzBvGtY9TUrEAzkw6Zli_bR7sse1ctN-7DCnZH_HI3UTm9qCxIEZYsFSU0uteAjGgxy0)

Let's continue this guide  with the simplified version of the application (only `prometheus` and `Grafana`)


##  Folder Structure and Orchestration process
Lets recap on some concepts about the tooling that we're implementing here to fulfil the promise of a full workflow with skaffold from local to production.

* I deploy on `K8s` cluster with `kubectl` and `kustomize` (kustomize is part of k8s bundle).
* I use `skaffold` as workflow building block through their cli command steps (build, test, render, deploy, verify)
* I build images locally with `docker` cli (is a pre requisite for my workflow) and in `gitlab` i have a couple of options alongside docker (`Kaniko` or `Docker in Docker` variations), but i'll cover that in the next steps. 
* I use a `Makefile` as command "collector" entrypoint no only for local development but for gitlab pipeline to group commands in single word ones (make run, build, unit, etc).
* I use `terraform` declarative configuration files, to set the desired state of my working cluster (in that case the local one), in this desired state we have some pre-requisites needed in my architecture definition, like `cert-manager`, `traefik`, `prometheus` and `grafana`, like my machines on staging and production. 

```bash
* deploy/
    manifests/  # the place where k8s yaml resides
         - *k8s.yaml
         - kuztomization.yaml
    overlays/ # for every environment that you want, you should have an overlay
        development/
              - *.k8s.patch.yaml
              - kuztomization.yaml
        production/
             - *.k8s.patch.yaml
             - kuztomization.yaml
   - skaffold.yaml
* infrastructure/ #terraform scripts to install cluster pre requisites vault, cert-manager, treafik
* src/ # all the source code of your application
    - Dockerfile 
- Makefile
- .gitlab-ci.yaml
``` 

Another important component of this setup is the `Dockerfile`, to be able to use the same dockerfile to build images for development and production environments (with the dependencies of each of them), we build a `multi-stage` dockerfile that allows us to get a target for development and a target for production that we can point to in the `skaffold` build phase. 

![Screenshot 2022-10-31 at 20.31.39.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667244719111/LWX13klyl.png align="left")

`Kustomize` files (`kustomization.yaml` ones) allow me to declare a `patch or merge` of the a part of the main `k8s` manifest, to apply the changes that I need to change in some environment without the necessity of duplicate the entire YAML, so for example, if I have the following `k8s` manifest declaring a api with 1 replica, I can declare a patch to set that number to 4 replicas if the environment is prod.

![Screenshot 2022-10-31 at 10.54.48.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667210150883/uVgpZtVjf.png align="left")

And the following image shows you how is a main `k8s` manifest and their corresponding patch for production. You can patch anything you want, adding all the data, metadata and others to every environment overlay.

![Screenshot 2022-10-30 at 17.54.36.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667148986552/Zj2sXJ9Oc.png align="left")

And then, we have our skaffold file in charge of the orchestration process of the workflow itself:

![Screenshot 2022-10-31 at 20.30.24.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667244800783/oBMDbsByO.png align="left")

Since `skaffold` allows us ti use `kustomization` as deploy strategy, with organise our profiles to do so, and also, with this, the development team has a lot of maneoveur to modify and deploy changes with zero effort.

Now we can run everything in one shot to see how this work. In our case we deploy the application (Availability API) alongside the monitoring stack (Grafana, Prometheus), so, it's everything is ok we'll capable to access all the tools (via browser) and the API requesting them.

To run skaffold you need to run the following command:
`skaffold dev -p development` but since we use a `Makefile` as command entrypoint, you can see above that make run do the same job as we need to run skaffold in development mode.

![ezgif-4-f2a066ca18.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1667151507712/gsBwGFfET.gif align="left")

I use my domain and a self signed certificate to access all applications via a `FQDN` over https (using `cert-manager` and `traefik` for that), now I'll be able to access all of them via that URLS (on the local machine this urls points to the loopback `127.0.0.1` in the `/etc/hosts` file)

We should have at lest this applications:
* Grafana 
* Traefik Dashboard 
* API /Application

Lets see in this animated GIF, that applications running:
![ezgif-4-162073e2b1.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1667153104081/m5Rk9v6Nx.gif align="left")

😅 with this , i already have a full local development cycle for my local environment, now the next milestone is, make my gitlab pipelines compliance with this tools and make the way to Low and Prod environments .

 Let's stop here for now. I'll prepare the material for the next blog entry. 

## Next Chapter
In the next chapter of this tutorial, I'll try to implement this local workflow in `gitlab` pipeline, allowing me to  use the `tests, build, render, deploy and verify` skaffold stages  in my entire pipelines and deploy the application to a `k8s` cluster in `GCP` in a full GitOps manner.

Thanks for reading and see you the next week for more!  😃

A Big KUDOS to the team #skaffold for the great job, if you wan to know more about you can reach them at [slack](https://kubernetes.slack.com/archives/CABQMSZA6) or in their [repo](https://github.com/GoogleContainerTools/skaffold) 

## Support me

If you like what you just read and you find it valuable, then you can buy me a coffee by clicking the link in the image below, I would be appreciated:

%%[buy-me-a-coffee]


