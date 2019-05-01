Continous delivery for microservices
------------------------

This is a step by step guide on how to set up a GitOps workflow with Istio/flagger and Weave Flux. GitOps is a way to do Continuous Delivery, it works by using Git as a source of truth for declarative infrastructure and workloads. In practice this means using git push instead of kubectl create/apply or helm install/upgrade.
 
Requirements
------------

  * A Kubernetes cluster (You can use managed Kubernetes cluster on GCP, AWS, Azure, OVH) 
  * docker, helm, kubectl, and git binaries.
  * Fork this repo into your github account
  * Access to a public registry (Docker HUB, GCR, Quay)

Prepare the cluster
-----------------------

Let's start by installing the necessary tools to use our cluster as a continuous delivery infrastructure. To follow this tutorial you must have a working K8S cluster and have configured the connection to it with administrator rights via kubectl.

#### Helm install

Download the latest version of Helm for your operating system at the following URL: https://github.com/helm/helm/releases

```bash
# Linux install
 wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
 tar zxvf helm-v2.13.1-linux-amd64.tar.gz
 sudo mv linux-amd64/helm /usr/local/bin/
 rm -rf linux-amd64
```

We will now install Tiller on the cluster, we will have to create an account so that tiller can deploy our applications on a cluster with RBAC enabled.

```yaml
# cluster-admin-tiller.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

```bash
# Install tiller with cluster admin rights

kubectl apply -f cluster-admin-tiller.yaml
helm init --service-account tiller

# Check installation
helm version
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
```

#### Weave Flux

Now that Helm is installed, we will be able to deploy the different applications we need, starting with Flux for the GitOps part.

> Don't forget to fork this repo on your account you will need the URL of your fork during the installation of feeds.

```bash
helm repo add weaveworks https://weaveworks.github.io/flux
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flux/master/deploy-helm/flux-helm-release-crd.yaml

# Install Weave Flux and its Helm Operator by specifying your fork URL (Just change your user)
helm upgrade -i flux \
--set helmOperator.create=true \
--set helmOperator.createCRD=false \
--set git.url=git@github.com:<YOUR_USER>/cd-containers.git \
--set git.path=flux/ \
--namespace flux \
weaveworks/flux
```

Retrieve the public key and add it to your forked repo. If you don't know how to do it, you can follow this documentation https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys

> Do not forget to add write access when you add the public key in github

```bash
# Copy the result of the command in the deploy keys section of your forked repo
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDW+goQDzlTdo...
```

#### Istio

Now that we have installed Flux, we will be able to use it to deploy missing applications in GitOps mode. In this repo, there is already the necessary configuration to install Istio via Flux. 

You just have to remove the ignore annotation, so that Flux automatically deploys Istio. Let's start by installing the Istio-init chart in charge of Istio's CRDs.

```yaml
# Edit the ini-istio-1.1.x.yaml and change the weave.work/ignore annotation to false
---
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: init-istio
  namespace: istio-system
  annotations:
    flux.weave.works/ignore: 'false'
spec:
  releaseName: init-istio
  chart:
    repository: https://storage.googleapis.com/istio-release/releases/1.1.0/charts/
    name: istio-init
    version: 1.1.0
  values:
    global:
      hub: gcr.io/istio-release
      tag: master-latest-daily
      imagePullPolicy: IfNotPresent
    certmanager:
      enabled: false
```

> Do not forget to push your changes !

```bash
# Check the logs to see the deployment of the helm chart
kubectl logs -f deployment/flux -n flux
Event(v1.ObjectReference{Kind:"HelmRelease", Namespace:"istio-system", Name:"init-istio", UID:"b620d559-6998-11e9-8381-8e7b409193c5", APIVersion:"flux.weave.works/v1beta1", ResourceVersion:"6823", FieldPath:""}): type: 'Normal' reason: 'ChartSynced' Chart managed by HelmRelease processed successfully

#Istio-init chart should be installed now
helm ls | grep init-istio
init-istio      1       DEPLOYED        istio-init-1.1.0        1.1.0           istio-system
```

Now follow the same process for the "istio-1.1.x.yaml file", and finally for "gw-istio-system.yaml" file. You must respect this order because the resources are dependent on each other. Let's make sure everything is properly installed before we continue.

```bash
# You should see these 3 Helm charts
helm ls
NAME            REVISION             STATUS          CHART                   APP VERSION     NAMESPACE
flux            3                    DEPLOYED        flux-0.9.1              1.12.0          flux
init-istio      1                    DEPLOYED        istio-init-1.1.0        1.1.0           istio-system
istio           1                    DEPLOYED        istio-1.1.0             1.1.0           istio-system

# The Istio GW ("gw-istio-system.yaml" file)
kubectl get gateway -n istio-system
NAME       AGE
cloud-gw   7m

# It shouldn't have any pods in pending otherwise it is that your nodes don't have enough CPU to deploy Istio
kubectl get po -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
istio-citadel-645ffc4999-btqmf            1/1     Running     0          12m
istio-galley-978f9447f-q55f4              1/1     Running     0          12m
istio-ingressgateway-8ccdc79bc-gk2rd      0/1     Running     0          12m
istio-init-crd-10-zjwcp                   0/1     Completed   0          20m
istio-init-crd-11-lpx69                   0/1     Completed   0          20m
istio-pilot-5d5dc955dd-rdc6z              0/2     Running     0          12m
istio-policy-c9469c4df-g8x88              2/2     Running     1          12m
istio-sidecar-injector-6dcc9d5c64-wsnc6   1/1     Running     0          12m
istio-telemetry-55d5b7d4dc-z4hwc          0/2     Running     0          12m
prometheus-66c9f5694-tvstz                1/1     Running     0          12m
```

#### Flagger

Last application for the automation of canary deployment with Istio. We will use flux to deploy it as we did with Istio.

Same procedure simply edit the flagger-0.9.0.yaml file and change the annotation flux.weave.works/ignore to false.

```yaml
---
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: flagger
  namespace: "istio-system"
  annotations:
    flux.weave.works/ignore: 'false'
spec:
  releaseName: flagger
  chart:
    repository: https://flagger.app/
    name: flagger
    version: 0.9.0
  values:
    namespace: "myapp"
```

```bash
# Verify the installation with helm
helm ls | grep flagger
flagger         1                 DEPLOYED        flagger-0.9.0           0.9.0           istio-system
```

Build and deploy our app
-------------------------

Now we will be able to deploy our application on the cluster, and take advantage of the canary deployment features, as well as the deployment via Gitops.

First we will build our application via a container, and publish its first version on the dockerhub registry.  For the simplicity of the example we will simply build a nginx image with a modified html index.

```bash
cd demo-gitops/app
# Change REGISTRY/REPOSITORY with the correct values for your registry.
docker build -t REGISTRY/REPOSITORY/myapp:1.0.0 .
docker push REGISTRY/REPOSITORY/myapp:1.0.0
```

Then edit the flux deployment file for your application.

```yaml
---
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: myapp
  namespace: myapp
  annotations:
    # change flux.weave.works/ignore to false
    flux.weave.works/ignore: 'false'
    flux.weave.works/automated: "true"
    flux.weave.works/tag.chart-image: semver:~1.0
spec:
  releaseName: myapp
  chart:
    git: git@github.com:skhedim/demo-gitops.git
    path: charts/myapp
    ref: master
  values:
    image:
      # change with your correct image path
      repository: REGISTRY/REPOSITORY/myapp
      tag: 1.0.0
    canary:
      enabled: true
      istioIngress:
        enabled: true
        gateway: cloud-gw.istio-system.svc.cluster.local
        # Change the MY_SVC_IPwith the result of following command:
        # kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
        host: MY_SVC_IP.xip.io
```

Push your changes, and you should be able to access your application using the url http://MY_SVC_IP.xip.io.

Canary deployment with Istio/Flagger
------------------------------------

Flagger takes a Kubernetes deployment and optionally a horizontal pod autoscaler (HPA) and creates a series of objects (Kubernetes deployments, ClusterIP services and Istio virtual services) to drive the canary analysis and promotion.

### Canary Custom Resource

For a deployment named _myapp_, a canary promotion can be defined using Flagger's custom resource:

```yaml
apiVersion: flagger.app/v1alpha3
kind: Canary
metadata:
  name: myapp
  namespace: test
spec:
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  # the maximum time in seconds for the canary deployment
  # to make progress before it is rollback (default 600s)
  progressDeadlineSeconds: 60
  # HPA reference (optional)
  autoscalerRef:
    apiVersion: autoscaling/v2beta1
    kind: HorizontalPodAutoscaler
    name: myapp
  service:
    # container port
    port: 9898
    # service port name (optional, will default to "http")
    portName: http-myapp
    # Istio gateways (optional)
    gateways:
    - cloud-gw.istio-system.svc.cluster.local
    - mesh
    # Istio virtual service host names (optional)
    hosts:
    - myapp.example.com
  # promote the canary without analysing it (default false)
  skipAnalysis: false
  # define the canary analysis timing and KPIs
  canaryAnalysis:
    # schedule interval (default 60s)
    interval: 1m
    # max number of failed metric checks before rollback
    threshold: 10
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 50
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    # Istio Prometheus checks
    metrics:
    - name: istio_requests_total
      # minimum req success rate (non 5xx responses)
      # percentage (0-100)
      threshold: 99
      interval: 1m
    - name: istio_request_duration_seconds_bucket
      # maximum req duration P99
      # milliseconds
      threshold: 500
      interval: 30s
    # external checks (optional)
    webhooks:
      - name: integration-tests
        url: http://myapp.test:9898/echo
        timeout: 1m
        # key-value pairs (optional)
        metadata:
          test: "all"
          token: "16688eb5e9f289f1991c"
```

### Istio Gateways and Virtual services

#### Gateways

A Gateway configures a load balancer for HTTP/TCP traffic operating at the edge of the mesh, most commonly to enable ingress traffic for an application.

Unlike Kubernetes Ingress, Istio Gateway only configures the L4-L6 functions (for example, ports to expose, TLS configuration). Users can then use standard Istio rules to control HTTP requests as well as TCP traffic entering a Gateway by binding a VirtualService to it.

#### Virtual services

A VirtualService defines the rules that control how requests for a service are routed within an Istio service mesh. For example, a virtual service could route requests to different versions of a service or to a completely different service than was requested. Requests can be routed based on the request source and destination, HTTP paths and header fields, and weights associated with individual service versions.

### Check the configuration

The istio gateway was deployed by flux via the istio-gw.yaml file of the repo. The Virtual service part will be automatically created by Flagger. We will now have to deploy our application and create the Canary file to Flagger

```bash
# Check the Istio GW configuration
kubectl get gateway -n istio-system  -o yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cloud-gw
  namespace: istio-system
  annotations:
    flux.weave.works/ignore: 'false'
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

# Check if you can see the canary configuration
kubectl get canary -n myapp
NAME    STATUS        WEIGHT   LASTTRANSITIONTIME
myapp   Initialized   0        2019-04-01T08:33:42Z
```

Deploy a new version
--------------------

Now everything is in place to test a canary deployment. Flagger monitors the myapp deployment,and waits for the image to be changed to create a second deployment and a virtual service.

In parallel Flux monitors the registry and waits for a new tag to be pushed into the registry to update the deployment. Modify the index.html file and push the new image to the registry

```html
<!--Change the version H2 title-->
<!DOCTYPE html>
<html>
<body>

<h2>Welcome to myapp v1.0.1</h2>

</body>
</html>
```

```bash
# Change REGISTRY/REPOSITORY with the correct values for your registry.
docker build -t REGISTRY/REPOSITORY/myapp:1.0.1 .
docker push REGISTRY/REPOSITORY/myapp:1.0.1

# Wait for flux to detect the new image in the registry
kubectl -n flux logs -f $(kubectl -n flux get pods -l app=flux -o jsonpath='{.items[0].metadata.name}')
```

Once Flux has detected that a new image has been pushed into the registry, that its tag matches the filter "tag.chart-image: semver:~1.0". It deploys the image and increments the helm chart revision.

After that, Flagger who monitors the myapp namespace and waits for an image to be changed on a deployment. When it detects the image change made by Flux, it will create a second deployment with the old image, another with the new one named myapp-canary and create a Virtual service to load and balance traffic between the two deployments.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
  namespace: myapp
  ownerReferences:
  - apiVersion: flagger.app/v1alpha3
    blockOwnerDeletion: true
    controller: true
    kind: Canary
    name: myapp
spec:
  gateways:
  - cloud-gw.istio-system.svc.cluster.local
  - mesh
  hosts:
  - <MY_SVc_IP>.xip.io
  - myapp
  http:
  - route:
  # We see the two deployments and the weight of each corresponding to the percentage of traffic oriented towards the two deployments
    - destination:
        host: myapp-primary
        port:
          number: 80
      weight: 60
    - destination:
        host: myapp-canary
        port:
          number: 80
      weight: 40
```

```bash
# The two deployments created by Flagger during the image change by Flux
kubectl get deploy -n myapp
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
myapp-canary    1/1     1            1           1m
myapp-primary   1/1     1            1           1m

kubectl get canary -n myapp
# We can see the progression and redirection of traffic to the canary deployment
NAME    STATUS      WEIGHT   
myapp   Progressing   10      
```

Now let's see how Flagger determines if it should continue to direct traffic to the new version of the application or if it should rollback to the old version.

To analyze the traffic we will use the kiali dashboard provided with Istio, to detect HTTP codes and traffic on our application

```bash
# Command used to access the kiali dashbord
kubectl port-forward svc/kiali 8080:20001 -n istio-system &
```

Now access the following URL http://localhost:8080/kiali, (if this doesn't work try it in private browsing). Use the admin login and the admin password, and go to the graph part, finally choose the myapp namespace to see the graph of our application appear.

Failed Canary analysis
----------------------

We will build a new image and increment the tag to launch a new canary analysis and see the flagger behavior.

```html
<!--Change the version H2 title-->
<!DOCTYPE html>
<html>
<body>

<h2>Welcome to myapp v1.0.2</h2>

</body>
</html>
```

```bash
# Change REGISTRY/REPOSITORY with the correct values for your registry.
docker build -t REGISTRY/REPOSITORY/myapp:1.0.2 .
docker push REGISTRY/REPOSITORY/myapp:1.0.2

# Check the images tags of each deployment
kubectl get deploy -n myapp -o yaml | grep image        
        - image: repository/myapp:1.0.2
          imagePullPolicy: Always
        - image: repository/myapp:1.0.1
          imagePullPolicy: Always

# Once the image update is complete, the canary analysis starts
kubectl get canary -n myapp
NAME    STATUS   WEIGHT   
myapp   Progressing   0 
```

We will generate traffic. You can then look at the graph in kiali and you can see the virtual service directing traffic on both deployments, and error codes, for the moment there should only be code 200 indicating that everything works well on our application. We can see the canary analysis progressing to the new version.

```bash
# generate traffic on your deployment
curl -L https://goo.gl/S1Dc3R | bash -s 20 "http://<MY_SVC_IP>.xip.io"
# Check the graph in the kiali console and after and after 1 or 2 minutes watch the progress of the analysis
CTRL+C
kubectl get canary -n myapp
NAME    STATUS   WEIGHT   
myapp   Progressing   20 
```

According to the configuration of the canary analysis as long as there is no error, the progression to the new application is done in steps of 5% of the traffic. We will now generate 404 errors and look at the deployment behavior

```bash
# generate traffic on your deployment on a non existent page to generate 404 errors
curl -L https://goo.gl/S1Dc3R | bash -s 20 "http://<MY_SVC_IP>.xip.io/notfound"
# Check the graph in the kiali console and after and after few minutes watch the progress of the analysis
CTRL+C
kubectl get canary -n myapp
NAME    STATUS   WEIGHT   
myapp   failed   0 
```

The analysis failed because more than 5% of the traffic contained 404 errors. By default Flagger only takes into account 5XX errors in its analysis, but The canary analysis can be extended with custom Prometheus queries. For this demo I added a control on 404 errors to easily make the analysis fail.

```json
    metrics:
    - name: "404s percentage"
      threshold: 5
      query: |
        100 - sum(
            rate(
                istio_requests_total{
                  reporter="destination",
                  destination_workload_namespace="test",
                  destination_workload="podinfo",
                  response_code!="404"
                }[1m]
            )
        )
        /
        sum(
            rate(
                istio_requests_total{
                  reporter="destination",
                  destination_workload_namespace="test",
                  destination_workload="podinfo"
                }[1m]
            )
        ) * 100
```

The analysis having failed flagger a rollback on the previous version of the application, if you go to the URL of the application. you will see that you always in version 1.0.1.

Successful Canary analysis
--------------------------

We will build a new image and increment the tag to launch a new canary analysis and see the flagger behavior.

```html
<!--Change the version H2 title-->
<!DOCTYPE html>
<html>
<body>

<h2>Welcome to myapp v1.0.3</h2>

</body>
</html>
```

```bash
# Change REGISTRY/REPOSITORY with the correct values for your registry.
docker build -t REGISTRY/REPOSITORY/myapp:1.0.3 .
docker push REGISTRY/REPOSITORY/myapp:1.0.2

# Check the images tags of each deployment
kubectl get deploy -n myapp -o yaml | grep image        
        - image: repository/myapp:1.0.2
          imagePullPolicy: Always
        - image: repository/myapp:1.0.3
          imagePullPolicy: Always

# Once the image update is complete, the canary analysis starts
kubectl get canary -n myapp
NAME    STATUS   WEIGHT   
myapp   Progressing   0 
```

We will generate traffic. You can then look at the graph in kiali and you can see the virtual service directing traffic on both deployments, and error codes, for the moment there should only be code 200 indicating that everything works well on our application. We can see the canary analysis progressing to the new version.

```bash
# generate traffic on your deployment
curl -L https://goo.gl/S1Dc3R | bash -s 20 "http://<MY_SVC_IP>.xip.io"
# Check the graph in the kiali console and after and after a few minutes watch the progress of the analysis
CTRL+C
kubectl get canary -n myapp
NAME    STATUS   WEIGHT   
myapp   Succeeded   0 
```

This time as we did not generate any 404 errors, the analysis is finished and the new version is deployed, now if you go to the application's URL, you will see that it is version 1.0.3 that is deployed.