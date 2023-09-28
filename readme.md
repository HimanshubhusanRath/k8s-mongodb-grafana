# MongoDB monitoring in Kubernetes using Prometheus and Grafana  

## Context:
* We want to monitor a MongoDB instance running in Kubernetes cluster.
* For regular application which exposes /metrics endpoint, this is pretty straightforward however as MongoDB does not expose any endpoint for its metrics, we need to check how we can monitor it.

* How MongoDB metrics will be available in Prometheus ?
  * We will use a Prometheus-Mongodb exporter which will read the metrics from MongoDB instance and convert it in a specific format which can be used by Prometheus.

* Once metrics is read by Prometheus, then it can also be available in Grafana and we can visualize the metrics.


# Steps:
## 1. Prometheus:
* Deploying different components of Prometheus Stack (alert manager, secrets, stateful sets, configs etc.) in one go using PROMETHEUS OPERATOR using HELM CHART.
  * `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
  * `helm repo update`
  * `helm install [RELEASE_NAME] prometheus-community/kube-prometheus-stack --> Use 'main' as RELEASE_NAME.`
    * `helm install main prometheus-community/kube-prometheus-stack`
`  * NOTE: This will take some time to deploy and make the PODs available.
  * Then when we do '`kubectl get all`', we can see the deployments, services, stateful sets, pods etc. created.
  * The below are the main areas to focus:
    * STATEFULSETS:
      * ***prometheus-stack-prometheus --> Prometheus 
      * ***prometheus-stack-alertmanager --> Alert Manager 


* Check the configurations for Prometheus server:
  * `kubectl get statefulset —> Get the promethus instance` 
  * `kubectl describe statefulset [Prometheus Instance] -oyaml`

* **How Prometheus discovers new endpoints :**
  * using ServiceMonitor

* **How Prometheus discovers the service monitors:**
  * Use ‘`kubectl get crd`’ and select the prometheus monitoring CRD which is ‘prometheuses.monitoring.coreos.com’ in my case. 
  * Use '`kubectl get prometheuses.monitoring.coreos.com -oyaml`’ command to see the configurations of this instance 
  * Check the  below configuration:
  ```
  serviceMonitorSelector:
      matchLabels:
        release: main  —> It means all the service monitors with 'release:main' label would be discovered in Prometheus so that their metrics data can be read.
  ```
  * So, in the custom ServiceMonitor instances, set the ‘release : main’ label so that those can be registered in Prometheus.


## 2. Monitor MongoDB using Prometheus:
### MongoDB instance:
  * This is the actual MongoDB instance. 
  * Installation:
    * `kubectl apply -f k8s/mongodb-instance.yml`

### MongoDB exporter:
  * This is a mongoDB exporter which performs below activities:
    * Pulls the metrics from MongoDB server. 
    * Converts these metrics to Prometheus metrics format. 
    * Expose these converted metrics as ‘/metrics' endpoint. 
  * Installation:
    * `helm show values prometheus-community/prometheus-mongodb-exporter`    —> Shows the list of available configurations
    * Select the below configurations from this list in a new file (e.g. helm-mongo-exporter.yaml).
    ```
      mongodb:
         uri: "mongodb://mongodb-service:27017"
      serviceMonitor:
         enabled: true
         additionalLabels:
         release: main
    ``` 

    * `helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter -f helm-mongo-exporter.yaml`
    * It deploys the below components:
      * MongoDB exporter Deployment
      * MongoDB exporter Service
      * MongoDB exporter Pods
      * MongoDB exporter Service Monitor


    
### MongoDB ServiceMonitor:
  * This is a service monitor used by Prometheus to read metrics for MongoDB server. 
  * This reads the /metrics endpoint exposed by MongoDB exporter.
  * Installation:
    * As part of MongoDB exporter

<img width="1038" alt="Screenshot 2023-09-28 at 6 50 56 AM" src="https://github.com/HimanshubhusanRath/k8s-mongodb-grafana/assets/40859584/e9f374c2-4586-42a0-9390-dc6d79d8fabb">



## 3. Access the components and check the metrics data:

* MongoDB Exporter:
  * `kubectl port-forward svc/mongodb-exporter-prometheus-mongodb-exporter 9216` 
  * Check 'http://localhost:9216/metrics' to see the metrics data pulled from MongoDB instance.
* Prometheus UI:
  * `kubectl port-forward pod/prometheus-main-kube-prometheus-stack-prometheus-0 9090` 
  * Check 'http://localhost:9090/targets?search=‘ —> MondoDB exporter should be visible with status ‘UP' --> This means the service monitor is registered in Prometheus.   
* Grafana UI:
  * Once the data is available in Prometheus, it will automatically appear in Grafana application. 
  * `kubectl port-forward deployment/main-grafana 3000` 
  * Check 'http://localhost:3000’ 
  * Go to -> Home —> Dashboards —> 'Kubernetes/Compute Resources/Pod' 
  * Select MongoDB Pod —> Metrics data should be visualized in the graphs



### NOTE: 
* Kubernetes version used in this project:
<img width="1547" alt="Screenshot 2023-09-28 at 8 14 23 AM" src="https://github.com/HimanshubhusanRath/k8s-mongodb-grafana/assets/40859584/637d463d-0847-42e9-b50f-bbb39cff7276">









  
 
