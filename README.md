# Strimzi/Kafka with Istio based Ingress

## Prequistes 

1. Install istio locally (I'm using 1.16.2)
2. Clone this repo and `cd` into it

# Prepare cluster

1. Using minikube
1. Install Istio with the Gateway APIs using [these instructions](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/):
```
kubectl get crd gateways.gateway.networking.k8s.io || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.5.1" | kubectl apply -f -; }
kubectl apply -f gatewayapi_alphas.yaml
istioctl install --set profile=minimal -y
```  
1. Install strimzi
```
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```




## Create Ingress Gateway

```
kubectl apply -f gateway.yaml
# Await ready
watch kubectl get gateway gateway -n istio-ingress
```

In another terminal start:
```
minikube tunnel
```

Wire up host side name resolution. This will allow the SNI names to resolve locally later.
```
INGRESS=$(kubectl get gateways gateway -n istio-ingress -ojsonpath='{.status.addresses[*].value}')
echo $INGRESS # It will probably look like 10.96.33.216

sudo sh -c 'cat >> /etc/hosts' << EOF
# Kafka/Ingress Gateway experiment
${INGRESS} my-cluster1-kafka-bootstrap.kafka my-cluster1-kafka-0.kafka my-cluster2-kafka-bootstrap.kafka my-cluster2-kafka-0.kafka
EOF
```





```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: gateway
  namespace: istio-ingress
spec:
  gatewayClassName: istio
  listeners:
  - allowedRoutes:
      kinds:
      - kind: TLSRoute
        group: gateway.networking.k8s.io
      namespaces:
        from: All
    name: mytcp
    port: 8090
    protocol: TLS
    tls:
      mode: Passthrough
```

## Create Kafka Instance

(Note: I've overridden the advertized host)

```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    config:
      default.replication.factor: 1
      inter.broker.protocol.version: "3.3"
      min.insync.replicas: 1
      offsets.topic.replication.factor: 1
      transaction.state.log.min.isr: 1
      transaction.state.log.replication.factor: 1
    listeners:
    - name: plain
      port: 9092
      tls: false
      type: internal
    - configuration:
        brokers:
        - advertisedHost: my-cluster-kafka-0.kafka
          advertisedPort: 8090
          broker: 0
      name: tls
      port: 9093
      tls: true
      type: internal
    replicas: 1
    storage:
      type: jbod
      volumes:
      - deleteClaim: false
        id: 0
        size: 100Gi
        type: persistent-claim
    version: 3.3.2
  zookeeper:
    replicas: 1
    storage:
      deleteClaim: false
      size: 100Gi
      type: persistent-claim
```


## Create routes for bootstrap and broker-0


```
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: my-cluster-tlsroute
spec:
  hostnames:
  - my-cluster-kafka-bootstrap.kafka
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: gateway
    namespace: istio-ingress
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: my-cluster-kafka-bootstrap
      port: 9093
      weight: 1
```




```
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: my-cluster-broker-0-tlsroute.kafka
spec:
  hostnames:
  - my-cluster-kafka-0
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: gateway
    namespace: istio-ingress
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: my-cluster-kafka-brokers
      port: 9093
      weight: 1
```


## Run a minikube tunnel

```
 minikube tunnel
```


## Get the Ingress IP

kubectl get gateways.gateway.networking.k8s.io gateway -n istio-ingress -ojsonpath='{.status.addresses[*].value}


## Munge your /etc/hosts

```
10.108.173.216 my-cluster-kafka-bootstrap.kafka my-cluster-kafka-0.kafka my-cluster2-kafka-bootstrap.kafka my-cluster2-kafka-0.kafka
```



## Run a command like this:

```
oc get secrets -n kafka  my-cluster-cluster-ca-cert -o json | jq -r '.data."ca.crt" | @base64d '  >> /tmp/trust
```

Create a config file
```
security.protocol=SSL
ssl.truststore.location=/tmp/trust
ssl.truststore.type=PEM
```


```
 kafka-console-producer --producer.config ./config -bootstrap-server my-cluster-kafka-bootstrap:8090 --topic foo
 
 ```
 
 ## Useful commands?
 
 ```
 istioctl proxy-config log gateway-5b4469658b-qjgjz.istio-ingress --level debug
 ```
 
