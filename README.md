# thanos-setup

* Install Ingress controller

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-controller ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

* Setup Minio

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install my-minio bitnami/minio \
  --set ingress.enabled=true --set auth.rootUser=minio --set auth.rootPassword=minio123 \
  --namespace minio --create-namespace

kubectl port-forward svc/my-minio 9001:9001 -n minio
```

* Create a bucket `thanos` after login to minio

* Setup Thanos

```
kubectl create ns thanos

## Create a file _thanos-s3.yaml_ containing the minio object storage config for tenant-a:
cat << EOF > thanos-s3.yaml
type: S3
config:
  bucket: "thanos"
  endpoint: "my-minio.minio.svc.cluster.local:9000"
  access_key: "minio"
  secret_key: "minio123"
  insecure: true
EOF

## Create secret from the file created above to be used with the thanos components e.g store, receiver
kubectl -n thanos create secret generic thanos-objectstorage --from-file=thanos-s3.yaml
kubectl -n thanos label secrets thanos-objectstorage part-of=thanos
```

* Setup Thanos Reciever

```
kubectl apply -f thanos/thanos-receiver-hashring-configmap-base.yaml
kubectl apply -f thanos/thanos-receive-controller.yaml

kubectl apply -f thanos/thanos-receive-default.yaml 
kubectl apply -f thanos/thanos-receive-hashring-0.yaml

kubectl apply -f thanos/thanos-receive-service.yaml
```

* Install Thanos Store

```
  kubectl apply -f thanos/thanos-store-shard-0.yaml
```

* Install Thanos Querier

```
  kubectl apply -f thanos/thanos-query.yaml
```

* Install Prometheus(es)

```
kubectl create ns sre
kubectl create ns tenant-a
kubectl create ns tenant-b
```

* Install kube-prometheus-stack

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --namespace sre --debug --install cluster-monitor prometheus-community/kube-prometheus-stack --version 56.8.2 \
  --set prometheus.ingress.enabled=true \                         
  --set 'prometheus.ingress.hosts[0]="cluster.prometheus.local"' \                                                            
  --set 'prometheus.prometheusSpec.remoteWrite[0].url="http://thanos-receive.thanos.svc.cluster.local:19291/api/v1/receive"' \
  --set 'alertmanager.ingress.enabled=true' \                         
  --set 'alertmanager.ingress.hosts[0]="cluster.alertmanager.local"' \
  --set 'grafana.ingress.enabled=true' \                            
  --set 'grafana.ingress.hosts[0]="grafana.local"'
```

* Setup Tenant-A

```
  kubectl apply -f tenant-a/
```

* Setup Tenant-B

```
  kubectl apply -f tenant-b
```

```
histogram_quantile(0.95, sum by(le,service, tenant_id) (rate(api_latency_milliseconds_bucket[5m])))
```