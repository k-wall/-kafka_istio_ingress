# Strimzi/Kafka with Red Hat Service Mesh based Ingress (OpenShift)

# How's it work?

* The [Gateway API](https://gateway-api.sigs.k8s.io/) is configured to allow to use SNI based routing (TLSRoute).  Istio responds by creating a Envoy instance, wired to a Kubernetes service which is exposed by using a External Load Balancer kube service.
* The Kafka CR is configured to expose `cluster-ip` TLS listeners, which provide 1 service per broker and 1 for bootstrap.  The `advertisedHost` feature is used to get Kafka to adveritize broker addresses that are publicly accessible.
* Alongside each Kafka CR we create [TLSRoute](https://gateway-api.sigs.k8s.io/concepts/api-overview/#tlsroute) objects (1 per service) that permit traffic to be ingressed to the bootstrap and broker address(es).


# Prequistes

1. Clone this repo and `cd` into the `openshift` directory.


# Prepare cluster

1. I'm using OSD 4.12.0
2. Install Red Hat Sevice Mesh as per https://docs.openshift.com/container-platform/4.12/service_mesh/v2x/installing-ossm.html
3. Install the Gateway API CRDs as per https://docs.openshift.com/container-platform/4.12/service_mesh/v2x/servicemesh-release-notes.html#kubernetes-gateway-api
4. Install strimzi
```
oc create namespace kafka
oc create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

Get the cluster base dns name:
```
CLUSTER=$(oc whoami --show-server | sed -e 's#https://api\.\(.*\):6443#\1#')
echo ${CLUSTER}
```
## Create Ingress Gateway

```
oc create namespace istio-ingress
oc apply -f gateway.yaml
# Await ready
watch oc get gateway gateway -n istio-ingress
```

Now get the external ingress DNS from the gateway:

```
INGRESS=$(kubectl get gateways gateway -n istio-ingress -ojsonpath='{.status.addresses[*].value}')
echo ${INGREESS} # It will look like a9a4e82db00c947f09ee00a47f51c720-215831897.us-east-1.elb.amazonaws.com
```

You'll need this in the next step.

# Create Public Route53 CNAMEs 

Use the AWS console, use Route53 to create wildcard CNAME from `*.kafka.${CLUSTER}` to `${INGERESS}`.

Test the names resolve on your local host:

```
nslookup my-cluster1-kafka-bootstrap.kafka.${CLUSTER}
nslookup my-cluster1-kafka-0..kafka.${CLUSTER}
```

## Create Kafka  with corresponding TLSRoutes and start producer/consumer

The important part here to note is that the domain names appearing in the advertised hosts tally the hostname used for
SNI based matching in the `TLSRoutes`.

```
CLUSTER=$CLUSTER envsubst | kubectl apply -f -
kubectl wait kafka/my-cluster1 --for=condition=Ready --timeout=300s -n kafka
kubectl get secrets -n kafka  my-cluster1-cluster-ca-cert -o json | jq -r '.data."ca.crt" | @base64d '  >> /tmp/my-cluster1.crt
```

And finally show it all works. In the first terminal run:


```
kafka-console-consumer  -bootstrap-server my-cluster1-kafka-bootstrap.kafka.${CLUSTER}:443 --topic foo -consumer-property security.protocol=SSL --consumer-property ssl.truststore.type=PEM --consumer-property ssl.truststore.location=/tmp/my-cluster1.crt --from-beginning
```

In the second run:
```
echo "hello" | kafka-console-producer  -bootstrap-server my-cluster1-kafka-bootstrap.kafka.${CLUSTER}:443 --topic foo --producer-property security.protocol=SSL --producer-property ssl.truststore.type=PEM --producer-property ssl.truststore.location=/tmp/my-cluster1.crt
```

