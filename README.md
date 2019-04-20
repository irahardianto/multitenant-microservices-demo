# Full Isolation in Multi-Tenant SaaS with Kubernetes + Istio

Hey! Welcome, this repositories contains helmcharts of ["Hipster Shop"](https://github.com/GoogleCloudPlatform/microservices-demo) a 10-tier microservices application deployment. The deployment is intended to demonstrate of how we can deploy these microservices in Kubernetes with seperate namespaces that directly map to each tenant on-boarded, then in-turn the frontend application which exposes the website where users can browse items, add them to the cart, and purchase them, and accessible independently through different subdomain for each tenant.

     http://<tenant-name>.<saas-domain> (e.g. http://tasty.spicynoodle.xyz)

The repository of the deployement I used for my DevOpsDays Jakarta 2019 talk about Full Isolation in Multi-Tenant SaaS with Kubernetes + Istio. You can find the slide [here](https://www.slideshare.net/MIchsanRahardianto/full-isolation-in-multitenant-saas-with-kubernetes-and-istio). 

I was talking about different architecture approach in multi tenancy SaaS, trade offs between each architecture. How Kubernetes and Istio can be leveraged to lower the barrier to setup full isolation deployments which offers the highest security and data privacy between tenants, the setup would also be the best approach both when the business scales or disaster happens.

Get Started:
- [Get It Up And Running](https://github.com/irahardianto/multitenant-microservices-demo/#get-it-up-and-running)
- [Introduction](https://github.com/irahardianto/multitenant-microservices-demo/#introduction)
- [Helm](https://github.com/irahardianto/multitenant-microservices-demo/#helm)
- [Routing](https://github.com/irahardianto/multitenant-microservices-demo/#routing)

----------
[Get It Up And Running](https://github.com/irahardianto/multitenant-microservices-demo/#get-it-up-and-running)
-------
To be able to get the deployment up and running, you need to have Kubernetes cluster up and running with Istio installed. The easiest way to do it from scratch is by using GKE and tick the "Enable Istio" box located in additional features section during the cluster creation, or through the following command below, as explained in more detail in the [Istio on GKE documentation](https://cloud.google.com/istio/docs/istio-on-gke/installing).

    $ gcloud beta container clusters create istio-demo  \  
        --addons=Istio  --istio-config=auth=MTLS_PERMISSIVE \  
        --cluster-version=[cluster-version]  \  
        --machine-type=n1-standard-2  \  
        --num-nodes=4

If you already have Kubernetes up and you are not using GKE, you can install Istio manually by referring to the [Quickstart Evaluation Install documentation](https://istio.io/docs/setup/kubernetes/install/kubernetes/).

Once you already have the cluster up and running, we then need to deploy the Istio ingress gateway, with the following command.

    $ kubectl apply -f istio-manifests/spicynoodle-gateway.yaml

Once you have the Istio ingress gateway up and running, go to your DNS server and configure the A record of your domain to point to the Istio ingress gateway IP address, and then create a CNAME record for all tenat you want to deploy.

![enter image description here](http://i64.tinypic.com/x2kk83.png)

The preceding picture shows how you can set your DNS record if you are using Cloud DNS, you can use Route 53 if you are using AWS as your platform. In the example above, you can see that we have anticipated that there will be three tenant using our system, let's deploy the first with helm.

    $ helm install --name tasty ./helmchart

If everything configured properly, then you will have tenant super be able to access their isolated deployment through their registered subdomain 

    http://tasty.spicynoodle.xyz

But probably at this point, if you haven't setup helm in the first place you wont be able to deploy the tenant microservices, so lets have it [installed](https://github.com/irahardianto/multitenant-microservices-demo/#helm)!

----------
[Introduction](https://github.com/irahardianto/multitenant-microservices-demo/#introduction)
-------
Just to make you not confused, the reason I have spicynoodle as the domain name and other related deployment parameters, while using Hipster Shop which doesn't any correlation whatsoever, is because originally I was going to write a simple noodle ecommerce site, but then life happened and DevOpsDays Jakarta was couple weeks away, then the rest is history.

Anyway, we will be deploying lots of microservices, for each our tenant we will be deploying 10 microservices. As the shown in the diagram below, these microservices will be deployed in the same Kubernetes cluster making our cluster to be multi-tenant while each of our tenant will enjoy single tenancy design that provide them with full isolation, both data wise communication wise.

Why is this important? Simply because by using single tenant approach, we can scale our business into whatever scale we want, we don't have any limitation of customisation be it to serve big enterprises customers or to be compliance with regulations from different region when we decided to after the market. The full isolation approach will also yield better data security and privacy, while also benefited from resource sharing efficiency from using multi tenant Kubernetes cluster.'

The traffic will be routed to the Istio ingress gateway then to virtual services which in turn will for ward the traffic to the frontend application of each tenant that is located in their respective namespaces.

![enter image description here](http://i67.tinypic.com/1zbf3ex.png)


----------
[Helm](https://github.com/irahardianto/multitenant-microservices-demo/#helm)
-------
Helm is a tool to help you manage Kubernetes application, in our use case we will be using helm to deploy our 10 hipstershop microservices, a redis, services, virtual service, service entry and a namespace that is istio-injection enabled to host all of those application.

Helm comes in two parts, the client and the tiller. The client is one you are interacting with, and the tiller is the one translating and deploying your instruction inside your Kubernetes cluster. There are several ways to install it as described in the [Quickstart Guide](https://helm.sh/docs/using_helm/#installing-helm). We will go with one of them as follows.

    $ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
    $ chmod 700 get_helm.sh
    $ ./get_helm.sh
    
Once you have installed it, you need to configure it using

    $ helm init

By default it will be using the current context that is currently being used by kubectl. Once we are done with that, we will be needing to give permission to the tiller via the following command.

    $ kubectl create serviceaccount --namespace kube-system tiller
    $ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    $ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

Once that done, you can start deploying your Kubernetes application through helm, happy helming!

### Namespace

Coming back to the deployment command we run earlier. Lets do this with dry run this time, so that there will be no resource be allocated, this is super helpful if we are validating whether our scripts will work as expected and see the result of the instruction inside the Kubernetes cluster.
      
    $ helm install --dry-run --debug --name tasty ./helmchart
  
There are lots of thing going on once you run the command, particularly you can observe the namespace. What it is, is basically a way for you as the cluster administrator allocate an environment for your certain group of users, in our case those groups are our tenant, further readings [here](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). 

    apiVersion: v1
    kind: Namespace
    metadata:
      name: {{ .Release.Name }}
      labels: 
        istio-injection: enabled

The preceding snippets are taken from helmchart/templates/namespace.yaml. Here we are using `{{ .Release.Name }}` as the variable to name the namespace, this variable is filled with the value you passed after the `--name` handle, which in this case it is `tasty`.

And if you open other files, you will find that those deployments and services are to be deployed in the namespace you created by declaring this line `namespace: {{ .Release.Name }}`

### Values
Values is being passed to the templates that we are going to deploy our Kubernetes application from values.yaml. For example, as you see where we are assigning container port `containerPort: {{ .Values.adsService.ports.containerPort }}`, you can find the value of the containerport variable if you examine the values.yaml file, so does other parameters defined for adservice use as follows.

    adsService:
      terminationGracePeriodSeconds: 5
      image:
        repository: gcr.io/google-samples/microservices-demo/adservice
        tag: v0.1.0
        pullPolicy: IfNotPresent
      ports:
        containerPort: 9555
      healthChecks:
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            probeBinaryPath: "/bin/grpc_health_probe"
            probePort: 9555
        livenessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            probeBinaryPath: "/bin/grpc_health_probe"
            probePort: 9555
      resources:
        requests:
          cpu: 200m
          memory: 180Mi
        limits:
          cpu: 300m
          memory: 300Mi
      service:
        type: ClusterIP
        name: grpc
        port: 9555

This also applies to other deployment and services too. We do it this way so that the configuration is easier to maintain, and reproducible in the future.

### Go Templating

Helm use go templating, thus we are able to use the features in building our charts.

    image: {{ printf "%s:%s"  .Values.cartService.image.repository .Values.cartService.image.tag }}
    command: {{ printf "[%q, \"-addr=:%v\"]"  .Values.cartService.healthChecks.readinessProbe.exec.probeBinaryPath .Values.cartService.healthChecks.readinessProbe.exec.probePort }}

The preceding lines are couple examples of how we can use the go templating in evaluating the values of our variables, one help full article [Top fmt formatting tips \[cheat sheet\]](https://yourbasic.org/golang/fmt-printf-reference-cheat-sheet/).

### Setting Up New Tenant

Earlier we have setup tenant "tasty" in our cluster, with a very easy command. This is important, because in order to maintain massive amount of client that we always wanted for our business, we need automation to play part in our deployment.

![enter image description here](http://i64.tinypic.com/f06mhf.png)

The preceding diagram shows the overview of if hook up our CI/CD pipeline to automate our application deployment. The key is, what every is reproducible should be automated, then we can onboard hundreds, thousands or any number of clients our business wish.

Let deploy new tenant environment and call it super.

    $ helm install --name super ./helmchart

### Rolling Out New Update Independently

One of the point of having single tenancy design as mention earlier, is to have the ability to scale independently and isolated, so that if our business decided to tap in new market that is not originally planned (e.g. new region with different policies and regulations or supporting customisation) that our application is not designed to accommodate, we are free to adjust without impacting the other tenant.

Having said that, maintaining different deployment that has different configurations and applications can greatly add complexity to our infrastructure. Again Kubernetes to the rescue, with the following command, we can start deploying the new customisation.

    $ helm upgrade super ./helmchart --set=frontend.image.repository=irahardianto/super-shop --set=frontend.image.tag=v1.1.0

After executing the command above we will update the base image of the frontend application that is being used by tenant "super" to irahardianto/super-shop:v1.1.0 without impacting tenant "tasty" that still use the default image.  

Now we know, that we can deploy separately for each of our tenants, these tenants could use different custom image that is coming from different development line and could be maintained by different development team. The diagram below shows how we could alter the pipeline if we decided to deploy those custom images.

![enter image description here](http://i64.tinypic.com/2jenrer.png)

----------
[Routing](https://github.com/irahardianto/multitenant-microservices-demo/#routing)
------- 
DNS
Istio Ingress Gateway 
Virtual Service
Service Entry