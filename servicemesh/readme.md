# Introduction to Istio

## Getting Started with Istio

Firstly, I like to do most of my work in containers so everything is reproducable
and my machine remains clean.

## Get a container to work in

Run a small `alpine linux` container where we can install and play with `istio`:

```bash
docker run -it --rm -v ${HOME}:/root/ -v ${PWD}:/work -w /work --net host alpine sh

# install curl & kubectl
apk add --no-cache curl nano
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
export KUBE_EDITOR="nano"

#test cluster access:
/work # kubectl get nodes
NAME                    STATUS   ROLES    AGE   VERSION
istio-control-plane   Ready    master   26m   v1.18.4

```

## Install Istio CLI

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.1 TARGET_ARCH=x86_64 sh -

cp istio-1.13.1/bin/istioctl /usr/local/bin/
chmod +x /usr/local/bin/istioctl
# mv istio-1.13.1 /tmp/

```

## Pre flight checks

Istio has a great capability to check compatibility with the target cluster

```bash
istioctl x precheck

```

## Istio Profiles

<https://istio.io/latest/docs/setup/additional-setup/config-profiles/>

```bash
istioctl profile list

istioctl install --set profile=default

kubectl -n istio-system get pods

istioctl proxy-status

istioctl x uninstall --purge
```

## Mesh our video catalog services

There are 2 ways to mesh:

- Automated Injection:

You can set the `istio-injection=enabled` label on a namespace to have the istio side car automatically injected into any pod that gets created in the labelled namespace

This is a more permanent solution:
Pods will need to be recreated for injection to occur

```bash
kubectl label namespace/default istio-injection=enabled

# restart all pods to get sidecar injected
kubectl delete pods --all
```

- Manual Injection:
This may only be temporary as your CI/CD system may roll out the previous YAML.
You may want to add this command to your CI/CD to keep only certain deployments part of the mesh.

```bash
kubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
playlists-api   1/1     1            1           8h 
playlists-db    1/1     1            1           8h 
videos-api      1/1     1            1           8h 
videos-db       1/1     1            1           8h 
videos-web      1/1     1            1           8h 

# Lets manually inject istio sidecar into our Ingress Controller:

kubectl -n ingress-nginx get deploy nginx-ingress-controller  -o yaml | istioctl kube-inject -f - | kubectl apply -f -

# You can manually inject istio sidecar to every deployment like this:

kubectl get deploy playlists-api -o yaml | istioctl kube-inject -f - | kubectl apply -f -
kubectl get deploy playlists-db -o yaml | istioctl kube-inject -f - | kubectl apply -f -
kubectl get deploy videos-api -o yaml | istioctl kube-inject -f - | kubectl apply -f -
kubectl get deploy videos-db -o yaml | istioctl kube-inject -f - | kubectl apply -f -
kubectl get deploy videos-web -o yaml | istioctl kube-inject -f - | kubectl apply -f -


```

## Bookinfo Demo Application deployment

```bash
kubectl apply -f bookinfo/platform/kube/bookinfo.yaml

$ kubectl get services
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.0.0.212      <none>        9080/TCP   29s
kubernetes    ClusterIP   10.0.0.1        <none>        443/TCP    25m
productpage   ClusterIP   10.0.0.57       <none>        9080/TCP   28s
ratings       ClusterIP   10.0.0.33       <none>        9080/TCP   29s
reviews       ClusterIP   10.0.0.28       <none>        9080/TCP   29s

$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-558b8b4b76-2llld       2/2     Running   0          2m41s
productpage-v1-6987489c74-lpkgl   2/2     Running   0          2m40s
ratings-v1-7dc98c7588-vzftc       2/2     Running   0          2m41s
reviews-v1-7f99cc4496-gdxfn       2/2     Running   0          2m41s
reviews-v2-7d79d5bd5d-8zzqd       2/2     Running   0          2m41s
reviews-v3-7dbcdcbc56-m8dph       2/2     Running   0          2m41s

$ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

## Open the application to outside traffic

Associate this application with the Istio gateway:

```bash
$ kubectl apply -f bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

Ensure that there are no issues with the configuration:

```bash
$ istioctl analyze
✔ No validation issues found when analyzing namespace: default.
```

## Determining the ingress IP and ports

```bash
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   172.21.109.129   130.211.10.121  80:31380/TCP,443:31390/TCP,31400:31400/TCP   17h
```

Set the ingress IP and ports:

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

GKE:

```bash
export INGRESS_HOST=workerNodeAddress
```

You need to create firewall rules to allow the TCP traffic to the ingressgateway service’s ports. Run the following commands to allow the traffic for the HTTP port, the secure port (HTTPS) or both:

```bash
gcloud compute firewall-rules create allow-gateway-http --allow "tcp:$INGRESS_PORT"
gcloud compute firewall-rules create allow-gateway-https --allow "tcp:$SECURE_INGRESS_PORT"
```

Set GATEWAY_URL:

```bash
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

Ensure an IP address and port were successfully assigned to the environment variable:

```bash
$ echo "$GATEWAY_URL"
192.168.99.100:32194
```

Run the following command to retrieve the external address of the Bookinfo application.

```bash
echo "http://$GATEWAY_URL/productpage"
```

## TCP \ HTTP traffic

Let's run a `curl` loop to generate some traffic to our site
We'll make a call to `/home/` and to simulate the browser making a call to get the playlists,
we'll make a follow up call to `/api/playlists`

```bash
While ($true) {curl -UseBasicParsing http://34.139.13.107:80/productpage; Start-Sleep -Seconds 1;}
```

## Observability

### Prometheus

```bash
kubectl apply -n istio-system -f istio-1.13.1/samples/addons/prometheus.yaml
```

### Grafana

```bash
kubectl apply -n istio-system -f istio-1.13.1/samples/addons/grafana.yaml
```

We can see the components in the `istio-system` namespace:

```bash
kubectl -n istio-system get pods
```

Access grafana dashboards :

```bash
kubectl -n istio-system port-forward svc/grafana 3000
```

### Kiali

`NOTE: this may fail because CRDs need to generate, if so, just rerun the command:`

```bash
kubectl apply -f istio-1.13.1/samples/addons/kiali.yaml

kubectl -n istio-system get pods
kubectl -n istio-system port-forward svc/kiali 20001
```

## Request Routing

### Apply Default Destination routing

Run the following command to create default destination rules for the Bookinfo services:

```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml

kubectl get destinationrules -o yaml
```

### Virtual Services

Run the following command to apply the virtual services:

```bash
kubectl apply -f bookinfo/networking/virtual-service-all-v1.yaml

kubectl get virtualservices -o yaml

kubectl get destinationrules -o yaml
```

Route based on user identity

```bash
kubectl apply -f bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

## Fault Injection

Apply application version routing by either performing the request routing task or by running the following commands:

```bash
kubectl apply -f bookinfo/networking/virtual-service-all-v1.yaml
kubectl apply -f bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

With the above configuration, this is how requests flow:

productpage → reviews:v2 → ratings (only for user jason)
productpage → reviews:v1 (for everyone else)

Create a fault injection rule to delay traffic coming from the test user jason.

```bash
kubectl apply -f bookinfo/networking/virtual-service-ratings-test-delay.yaml

# Confirm the rule was created:

kubectl get virtualservice ratings -o yaml
```

Injecting an HTTP abort fault

Create a fault injection rule to send an HTTP abort for user jason:

```bash
kubectl apply -f bookinfo/networking/virtual-service-ratings-test-abort.yaml

Confirm the rule was created:

kubectl get virtualservice ratings -o yaml
```

## Traffic Shifting

Apply weight-based routing

To get started, run this command to route all traffic to the v1 version of each microservice.

```bash
 kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

Transfer 50% of the traffic from reviews:v1 to reviews:v3 with the following command:

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

Assuming you decide that the reviews:v3 microservice is stable, you can route 100% of the traffic to reviews:v3 by applying this virtual service:

```bash
 kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

## Querying Metrics from Prometheus

Verify that the prometheus service is running in your cluster.

In Kubernetes environments, execute the following command:

```bash
kubectl -n istio-system get svc prometheus
```

Execute a Prometheus query.

In the “Expression” input box at the top of the web page, enter the text:

`istio_requests_total`

Other queries to try:

Total count of all requests to the productpage service:

`istio_requests_total{destination_service="productpage.default.svc.cluster.local"}`

Total count of all requests to v3 of the reviews service:

`istio_requests_total{destination_service="reviews.default.svc.cluster.local", destination_version="v3"}`

This query returns the current total count of all requests to the v3 of the reviews service.

Rate of requests over the past 5 minutes to all instances of the productpage service:

`rate(istio_requests_total{destination_service=~"productpage.*", response_code="200"}[5m])`

## Uninstall

To delete the Bookinfo sample application and its configuration, see Bookinfo cleanup.

The Istio uninstall deletes the RBAC permissions and all resources hierarchically under the istio-system namespace. It is safe to ignore errors for non-existent resources because they may have been deleted hierarchically.

```bash
kubectl delete -f istio-1.13.1/samples/addons
istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -
istioctl tag remove default
```

The istio-system namespace is not removed by default. If no longer needed, use the following command to remove it:

```bash
kubectl delete namespace istio-system
```

The label to instruct Istio to automatically inject Envoy sidecar proxies is not removed by default. If no longer needed, use the following command to remove it:

```bash
kubectl label namespace default istio-injection-
```

<!-- ## Virtual Services

### Auto Retry

Let's add a fault in the `videos-api` by setting `env` variable `FLAKY=true`

```bash
kubectl edit deploy videos-api
```

```bash
kubectl apply -f kubernetes/servicemesh/istio/retries/videos-api.yaml
```

We can describe pods using `istioctl`

```bash
# istioctl x describe pod <videos-api-POD-NAME>

istioctl x describe pod videos-api-584768f497-jjrqd
Pod: videos-api-584768f497-jjrqd
   Pod Ports: 10010 (videos-api), 15090 (istio-proxy)
Suggestion: add 'version' label to pod for Istio telemetry.
--------------------
Service: videos-api
   Port: http 10010/HTTP targets pod port 10010
VirtualService: videos-api
   1 HTTP route(s)
```

Analyse our namespace:

```bash
istioctl analyze --namespace default
```

### Traffic Splits

Let's deploy V2 of our application which has a header that's under development

```bash
kubectl apply -f kubernetes/servicemesh/istio/traffic-splits/videos-web-v2.yaml

# we can see v2 pods
kubectl get pods

```

Let's send 50% of traffic to V1 and 50% to V2 by using a `VirtualService`

```bash
kubectl apply -f kubernetes/servicemesh/istio/traffic-splits/videos-web.yaml
```

### Canary Deployments

Traffic splits has its uses, but sometimes we may want to route traffic to other
parts of the system using feature toggles, for example, setting a `cookie`

Let's send all users that have the cookie value `version=v2` to V2 of our `videos-web`.

```bash
kubectl apply -f kubernetes/servicemesh/istio/canary/videos-web.yaml
```

We can confirm this works, by setting the cookie value `version=v2` followed by accessing <https://servicemesh.demo/home/> on a browser page -->
