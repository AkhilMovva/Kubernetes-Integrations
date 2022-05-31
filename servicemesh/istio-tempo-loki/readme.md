# Istio, Tempo, and Loki speed up debugging for microservices

## Istio

Istio up and running.

Let’s install the Istio operator:

```bash
istioctl operator init
```

Now, let’s instantiate the service mesh. Istio proxies include a traceID in the x-b3-traceid. Notice that you will set the access logs to inject that trace ID as part of the log message:

```bash
kubectl apply -f istio/service-mesh.yaml
```

## Install the demo application

```bash
kubectl create ns bookinfo

kubectl label namespace bookinfo istio-injection=enabled --overwrite

kubectl apply -n bookinfo -f  bookinfo/bookinfo.yaml
kubectl apply -n bookinfo -f  bookinfo/bookinfo-gateway.yaml

kubectl port-forward svc/istio-ingressgateway -n istio-system  8080:80
```

## Install Grafana Stack

Now, let’s create the Grafana components

### Tempo

Tempo, the tracing backend

```bash
kubectl create ns tracing

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

helm install tempo grafana/tempo --version 0.7.4 -n tracing -f grafana-stack/tempo-values.yaml
```

### Loki

```bash
helm install loki grafana/loki-stack --version 2.4.1 -n tracing -f grafana-stack/loki-values.yaml
```

### Opentelemetry Collector

use this component to distribute the traces across your infrastructure

```bash
kubectl apply -n tracing -f https://raw.githubusercontent.com/antonioberben/examples/master/opentelemetry-collector/otel.yaml

kubectl apply -n tracing -f grafana-stack/opentelemetry-collector-cm.yaml
```

### fluent-bit

use this component to scrap the log traces from your cluster:

(Note: In the configuration you are specifying to take only containers which match the following pattern /var/log/containers/*istio-proxy*.log)

```bash
helm repo add fluent https://fluent.github.io/helm-charts

helm repo update

helm install fluent-bit fluent/fluent-bit --version 0.16.1 -n tracing -f grafana-stack/fluentbit-values.yaml

```

### Grafana

the Grafana query. This component is already configured to connect to Loki and Tempo:

```bash
helm install grafana grafana/grafana -n tracing --version 6.13.5  -f grafana-stack/grafana-values.yaml
```

### Test

After the installations are completed, let’s open a tunnel to the Grafana query forwarding the port:

```bash
kubectl port-forward svc/grafana -n tracing 8081:80
```

Access it using the credentials you have configured when you installed it:

```text
user: admin
password: password
http://localhost:8081/explore
```

## Links

[blog](https://grafana.com/blog/2021/08/31/how-istio-tempo-and-loki-speed-up-debugging-for-microservices/)
