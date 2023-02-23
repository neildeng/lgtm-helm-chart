# LGTM-Stack Helm Chart
We have developed a plug-and-play Grafana's LGTM stack that will assist individuals in rapidly familiarizing themselves with it.


### Prerequisites

Make sure you have `Helm 3+` and `MinIO`

Create MinIO Server on cluster outside if you need
```shell
docker run -it -d --name minio \
   -p 9900:9900 \
   -p 9990:9990 \
   -v minio:/data \
   -e "MINIO_ROOT_USER=admin" \
   -e "MINIO_ROOT_PASSWORD=changeme" \
   quay.io/minio/minio server /data --console-address ":9990" --address ":9900"
```
and create below buckets:
- tempo
- loki
- mimir-tsdb
- mimir-ruler
- phlare

modify /etc/hosts
```text
## Change below to you want
192.168.X.X minio.homelab.test
192.168.X.X grafana.homelab.test
192.168.X.X alertmanager.homelab.test
192.168.X.X prometheus.homelab.test
192.168.X.X mimir.homelab.test
```

### Pre-config Ingress Hosts and MinIO(Optional)
   ```yaml
   # file name: values-ingresses.yaml
   kubeprometheusstack:
      alertmanager:
         ingress:
            enabled: true
            annotations:
               kubernetes.io/ingress.class: nginx
            hosts:
               - alertmanager.homelab.test
            tls: []
   
      prometheus:
         ingress:
            enabled: true
            annotations:
               kubernetes.io/ingress.class: nginx
            hosts:
               - prometheus.homelab.test
            tls: []
   
      grafana:
         ingress:
            enabled: true
            annotations:
               kubernetes.io/ingress.class: nginx
            hosts:
               - grafana.homelab.test
            tls: []
   
   mimir:
      nginx:
         ingress:
            enabled: true
            annotations:
               kubernetes.io/ingress.class: nginx
            hosts:
               - host: mimir.homelab.test
                 paths:
                    - path: /
                      pathType: Prefix
            tls: []
   
   minio:
      ingress:
         enabled: true
         annotations:
            kubernetes.io/ingress.class: nginx
         hosts:
            - host: minio.homelab.test
         tls: []
   
   ```

### Pre-config Object Storage Credential
```yaml
## values-object-storages.yaml
## such as:
        s3:
          endpoint: minio.homelab.test:9900
          insecure: true
          access_key: admin
          secret_key: changeme
          bucket: tempo
```

### Deploy Kube-Prometheus-Stack、LGTM、Phlare to your cluster and store date to MinIO

```shell
helm upgrade --install lgtm-aio ./ 
  --namespace observability --create-namespace \
  -f ./LGTM/values-grafana.yaml \
  -f ./LGTM/values-loki.yaml \
  -f ./LGTM/values-tempo.yaml \
  -f ./LGTM/values-phlare.yaml \
  -f ./LGTM/values-mimir.yaml \
  -f ./LGTM/values-object-storages.yaml \
  -f ./LGTM/values-ingresses.yaml
```

### Deploy Kube-Prometheus-Stack、Grafana to your cluster

```shell
helm upgrade --install lgtm-aio ./ 
  --namespace observability --create-namespace \
  -f ./LGTM/values-grafana.yaml \
  -f ./LGTM/values-ingresses.yaml
```

### Deploy Kube-Prometheus-Stack、Grafana、Phlare to your cluster and store date to MinIO

```shell
helm upgrade --install lgtm-aio ./ 
  --namespace observability --create-namespace \
  -f ./LGTM/values-grafana.yaml \
  -f ./LGTM/values-phlare.yaml \
  -f ./LGTM/values-object-storages.yaml \
  -f ./LGTM/values-ingresses.yaml
```

### Deploy with custom dashboards

1. create directory in dashboards, such as `custom`
2. place dashboard json in above created direcroy, like `custom-default.json`
3. config `values-grafana.yaml`
    ```yaml
    kubeprometheusstack:
      grafana:
        dashboardProviders:
          dashboardroviders.yaml:
            apiVersion: 1
            providers:
   
            ## ... appended 
              - name: 'custom'
                folder: 'custom'
                allowUiUpdates: true
                disableDeletion: false
                editable: true
                options:
                  path: /var/lib/grafana/dashboards/custom
            ## ...
        
        dashboardsConfigMaps:
          custom: "custom-dashboards"  ## ... appended   
        
    
    ```