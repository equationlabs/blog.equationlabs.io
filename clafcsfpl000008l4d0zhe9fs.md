# Managing database migrations safely in high replicated k8s deployment.

So, you want to run migrations in a cloud native application running on a `Kubernetes` cluster, and don't die trying huh!

\*\*\*Well you're in the right place!! \*\*\*

After I break some applications in terms of database migrations in a multi replica and concurrent deployment process, I want to give you some advices based on my faults, on how you can run you migrations in a safely way, with native `k8s` specs and without hacks of any kind. (no `helm`, no `external deployers`, pure and plain `k8s` process well orchestrated)

## The Problem

Is very common, modern application evolves faster, new features arise from product to satisfy the final user, and with every new deploy is too common the need to alter your database in some form and you have, many tools to allow you to manage the execution of the migrations against your database, BUT, not when they occur.

If you have an application pod, let say with 4 replicas, and you deploy it, the all 4 will try to run the migrations at the same time potentially causing data corruption and data loss, and nobody wants that.

## When to run migrations `(the workflow)`

In the `old-way` of run migrations we used to have something like this:

* Put the application in `maintenance mode` (divert traffic to a special page)
    
* Run database migrations
    
* Deploy new base code
    
* Disable maintenance mode on application
    

Obviously, this isn't acceptable approach if you want to achieve `zero-downtime` deployments in the actual always-on world, we need to achieve *(at leats)* the following steps to assure that migrations and application run in a safety way.

* Run migrations while the old version of the application is still running, and do the rolling update "only" when migrations are successfully run.
    

Something like this:

![Pipeline + cluster proposals-2.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1668344068616/A3UYB5av5.jpg align="left")

## `Job`, `InitContainer` and `RollingUpdates`

So after choose our strategy to run `migrations` on `k8s`, we need to write our manifests in order to accomplish the defined workflow.

First, the migrations job itself:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: availability-api-migrations
spec:
  ttlSecondsAfterFinished: 60
  backoffLimit: 1
  template:
    spec:
      containers:
        - name: migrations
          image: availability-api-migrations
          command:
            - '/bin/sh'
            - '-c'
            - ' bin/console doctrine:migrations:migrate --no-interaction -v'
          envFrom:
            - secretRef:
                name: protected-credentials-from-vault
      restartPolicy: Never
```

Now in the deployment manifest of the application we need to defined 2 things very important to allow our workflow work as expected.

* The rolling update strategy
    
* the init container and command to forbid deployment init until migrations are done.
    

```yaml
apiVersion: apps/v1
kind: Deployment
...
spec:
  ...
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
   template:
      spec:
        initContainers:
          - name: wait-for-migrations-job
            image: bitnami/kubectl:1.25
            command:
              - 'kubectl'
            args:
              - 'wait'
              - '--for=condition=complete'
              - '--timeout=600s'
              - 'job/availability-api-migrations'
```

What is the meaning of that snippet above:

* `RollingUpdate` strategy:
    
    * The rolling update allow us, to define the strategy on how many pod replicas we want to update with the new code at a time (you can also choose between other `k8s` strategies like `recreate`, `blue/green` or `canary`)
        
* `InitContainer` migration job watcher
    
    * Now we want, to `forbid` in some way that the rollout deployment begins until the migrations job was finished in completed status. Fortunately the `kubectl cli` tools allow us to ask and wait the status of `k8s` component, we use that advantage of the kubelete api to use here, and "block" in some way the deployment
        

```bash
$kubectl wait --for-condition=complete --timeout=600 job/availability-api-migrations
```

We wait for the job `availability-api-migrations` status complete with a `timeout` o five minutes, the kubectl will ask permanently until timeout was reach. Obviously if the migration job finish in there milliseconds, automatically the wait loop ends and allow the deploy to begin otherwise, the deploy will fail. The timeout is the MAX allowed time to wait for ask for complete status on a job.

At least with this we can assure that data consistency is preserve, but it's importan follow some guidelines in terms on how to write and deploy migrations in you applications, below we'll cover that.

## Safety recommendations

* Write your migrations always thinking in a fast execution, if you expect to run migrations that took more than 5 minutes, considering ask for a maintenance window to restrict traffic access to the application.
    
* Do your migrations always thinking in retro compatibility of your current code running on production.
    
* For example, do not alter a table adding a column NOT NULLABLE or without DEFAULT value.
    
* If you need to add a column and delete other, the best strategy to follow is to do 2 separates deployments.
    
    * Run the first deploy adding the column, and validate that all is running smoothly in production.
        
    * Run a new deployment only deleting the old column from the database.
        

## Support Me

If you like what you just read and you find it valuable, then you can buy me a coffee by clicking the link in the image below, I would be appreciated or you can become an sponsor :).