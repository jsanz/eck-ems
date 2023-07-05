# Deply Elastic Stack with ECK and EMS Server

These instructions help going through the process of deploying a simple Kubernetes cluster with the Elastic Cloud on Kubernets (ECK). This is a testing environment with self-signed certificates, just one node per resource and without a domain.

More details about ECK at https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html

## Installation of ECK

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html

```bash
# Create namespace and set it as default
kubectl apply -f namespace.yml
kubectl config set-context $(kubectl config current-context) --namespace=elastic-system
# Install CRD 
kubectl create -f https://download.elastic.co/downloads/eck/2.8.0/crds.yaml
# and RBAC
kubectl apply -f https://download.elastic.co/downloads/eck/2.8.0/operator.yaml
```

Monitor the operator logs

```bash
kubectl logs -f statefulset.apps/elastic-operator
```

## Install an Enterprise trial license (30 days)

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-licensing.html


```bash
kubectl apply -f elastic-license-trial.yml
```

When applied, get usage data and confirm you moved from `basic` to `enterprise_trial`.

```bash
kubectl get configmap elastic-licensing -o json | jq .data
```

## Install Elasticsearch

```bash
# Create the resource
kubectl apply -f elasticsearch.yml
```

Check the status from ECK:

```bash
# Check the status
kubectl get elasticsearch
```

Check the pod logs:

```bash
# Get the pod
export ES_POD=$(kubectl get pods --selector='elasticsearch.k8s.elastic.co/cluster-name=quickstart' --output jsonpath='{.items[0].metadata.name}') 

# Get the logs
kubectl logs -f $ES_POD
```

On a separate terminal start the port forwarding

```bash
kubectl port-forward service/quickstart-es-http 9200 
```

Now start hitting ES endpoint:

```bash
# Get the credentials
export PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' )

# License details, should be trial
curl -u "elastic:$PASSWORD" -k "https://localhost:9200/_license"
```

## Insatll EMS Server

Get the external IP of the cluster and update the tls section in the `ems.yml` file

```bash
kubectl apply -f ems.yml
```

Check the status

```bash
# From k8s
kubectl get ems
```

Check the pod:

```bash
export MAPS_POD=$(kubectl get pods --selector='maps.k8s.elastic.co/name=quickstart' --output jsonpath='{.items[0].metadata.name}')
# Get the logs
kubectl logs -f ${MAPS_POD}
```

Check the service

```bash
kubectl get service/quickstart-ems-http
```

And query the status endpoint extracting the IP first

```bash
export MAPS_IP=$(kubectl get service/quickstart-ems-http --output jsonpath="{.status.loadBalancer.ingress[0].ip}")
# With curl
curl -sk "https://${MAPS_IP}:8080/status" | jq .overall
```


## Install Kibana

Note the external IP of the cluster on the `tls` section for the self-signed certificate to be generated properly. Also update the `maps.emsUrl` to the external URL of the Elastic Maps Server.

```bash
kubectl apply -f kibana.yml
```


Get the logs

```bash
export KB_POD=$(kubectl get pods --selector='kibana.k8s.elastic.co/name=quickstart' --output jsonpath='{.items[0].metadata.name}')
# Get the logs
kubectl logs -f ${KB_POD}
```

Get the external IP and credentials to log from your browser:

```bash
# Get the Kibana IP
export KB_IP=$(kubectl get service/quickstart-kb-http --output jsonpath="{.status.loadBalancer.ingress[0].ip}")
# With curl
echo "Kibana status"
curl -sk -u "elastic:$PASSWORD" "https://${KB_IP}:5601/api/status"  | jq .status.overall
# Print details
echo "Kibana host:\t\thttps://${KB_IP}:5601\nUser:\t\t\telastic\nPassword:\t\t${PASSWORD}"
```
