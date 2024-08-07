# Monitoring Applications in Kubernetes with Prometheus and Grafana.

In this project we will be setting up a Prometheus monitoring for a third party app in Kubernetes cluster. Before we start the implementation, we are going to explain some terms below.

[**Prometheus**](https://www.tigera.io/learn/guides/prometheus-monitoring/): rometheus is an open-source technology designed to provide monitoring and alerting functionality for cloud-native environments, including Kubernetes. It can collect and store metrics as time-series data, recording information with a timestamp. It can also collect and record labels, which are optional key-value pairs.

why is Prometheus important?

Modern DevOps becomes more and more complex to handle manually and therefore needs automation. Typically you have multiple servers that run containerized applications and there are hundreds of different processes running on that infrastructure and things are interconnected. Maintaining such setup to run smoothly without application downtimes is very challenging

Imagine having such a complex infrastucture with loads of servers distributed over many locations and you have no insight of what is happening on the hardware level or on application level like errors, responses latency, hardware down or overloaded or maybe running out of resources etc. If something happens application may becomes unavailable to user and you must be able to identify out of the hundred of of things in the cluster went wrong and that could be difficult and time-consuming when debugging the system manaually

What makes the process of debugging easier is to have a tool that  constantly monitor if services are running and alert the maintainer as soon as one service crashes so you know exactly what happened or even better it identify problems before they even occur and alerts the system administrators responsible for that infrasture to prevent that issue. 

How does prometheus work?

![alt text](images/1.2.png)

Prometheus consist of 3 main components
- Time Seriess Database: This is the main storage that prometheus uses for storing metrics data. Prometheus store agreggated metrics on **Disk(HDD/SSD)** and it can also optionally integrate with remote Storage Systems and the data is stored in a **Custom Time Series Format** and because of that you write prometheus into a relational database.  
- Data Retrieval Worker: This the components of the prometheus that pull and retrieve metrics from services and applications and forwards to Tthe Time Series Database
-  Prometheus Server: it has a web server or server API that accepts queries using PromQL queries for the stored data inthe in the database. the server api displays the metrics through Prometheus Web UI dashboard or any other data visualization applications such as Grafana.
 
 Below is the full Prometheus Architecture.
 

![alt text](images/1.1.png)

What does prometheus monitor?

Prometheus monitors everything such as Linus/Windows server, Single application, Apache Server, CPU Status, Services like Database and many more. Those things that prometheus monitors are called **Targets**. The units that the target monitors are called **Metrics**. And the metric could be CPU status, Memory/Disk Space Usage, Exception count, request count, request duration

How does Prometheus collect those metrics from target?

Prometheus collects metrics from target systems by scraping HTTP endpoints that expose these metrics in a specific format. Some of the methods are explain below:

1. **Target Configuration:** This can be achieve in 2 ways for different environments
- **Static Configuration:** You can configure Prometheus with a list of targets in its configuration file. This is useful for small or static environments where the list of targets doesn't change frequently.
```
scrape_configs:
  - job_name: 'my_job'
    static_configs:
      - targets: ['localhost:9090', 'example.com:8080']

```
- **Service Discovery:** For dynamic environments, Prometheus supports service discovery mechanisms (e.g., Kubernetes, Consul, EC2) to automatically discover targets.
```
scrape_configs:
  - job_name: 'kubernetes'
    kubernetes_sd_configs:
      - role: pod
```

2. **Metrics Endpoint** Each target system needs to expose an HTTP endpoint (typically /metrics) that outputs the current state of the system in a format that Prometheus understands. The output is usually in plain text, following a key-value pair format with optional labels for dimensional data.

Example of a Prometheus metrics endpoint:
```
# HELP http_requests_total The total number of HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="post",code="200"} 1027
http_requests_total{method="post",code="400"} 3
# HELP http_request_duration_seconds The HTTP request latencies in seconds.
# TYPE http_request_duration_seconds summary
http_request_duration_seconds{quantile="0.5"} 0.05
http_request_duration_seconds{quantile="0.9"} 0.1
http_request_duration_seconds{quantile="0.99"} 0.2
```
3. **Scraping Process:** 
- **Scraping**: Prometheus periodically scrapes the /metrics endpoint of each configured target. The scraping interval can be configured per job.

```
scrape_configs:
  - job_name: 'my_job'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9090', 'example.com:8080']
```
- **HTTP Request**: Prometheus makes an HTTP GET request to the target’s metrics endpoint and parses the response.

4. **Storing Metrics**: After scraping the metrics, Prometheus stores them in its time-series database. Each metric is stored along with its labels and the timestamp of when it was scraped.
5. **Querying and Alerting**
 - **Querying:** Users can query the stored metrics using PromQL (Prometheus Query Language) to gain insights into the system’s performance and behavior.
```
sum(rate(http_requests_total[5m]))
```
- **Alerting:** Prometheus can evaluate alerting rules defined in its configuration and send alerts to various notification channels when certain conditions are met.

```
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'my_job'
    static_configs:
      - targets: ['localhost:9090', 'example.com:8080']

```

Some servers exposes prometheus metrics by themselves, but some don't. For the Applications that don't exposes metrics, `Exporters` can be used. **Exporters** are lightweight binaries that fetch metrics from a system and expose them in the Prometheus format. Prometheus Exporter list can be found [here](https://prometheus.io/docs/instrumenting/exporters/). 

What if you want to monitor your own applications such as how many request? how many exceptions? or how many server resources are used? prometheus have a [**Client Libary**](https://prometheus.io/docs/instrumenting/clientlibs/) for diffrent languages like Go, java, Python and so on.

 
### How to deploy the different parts of the prometheus in Kubernetes cluster

There are several ways to deploy prometheus in kubernetes cluster: some are listed below
1. The first methods is to create all the configurations YAML files yourself and execute them in right order. This can work but it is not adviceable because it involves a lot of work by configuring each part such as [Prometheus statefulSets, Grafana, Alert manager, configMap, secrets and more]. you ahve to configure them correctly and executing them in the right order because of dependencies. 
2. Operator: we can think of Operator as the manager of all prometheus components we are deploying on the kubernetes cluster. You will find [Prometheus operator](https://prometheus-operator.dev/) and deploy in k8s cluster.
 3. Using [Helm chat](https://helm.sh/) to deploy operator which is the most efficient way.  We will be using Helm chat for this project
 - Helm will initiate the setup 
 - Operator will manage the setup

We will use Minikube to set up our cluster and Kubectl to interract with our cluster.

Minikube is a tool that sets up a Kubernetes environment on a local PC or laptop. It’s technically a Kubernetes distribution, but because it addresses a different type of use case than most other distributions (like Rancher, OpenShift, and EKS), it’s more common to hear folks refer to it as a tool rather than a distribution. Read more about [Minikube here](https://sysdig.com/learn-cloud-native/what-is-minikube/).

[Kubectl](https://www.getambassador.io/blog/what-is-kubectl) is a command line tool used to run commands against Kubernetes clusters. It does this by authenticating with the Master Node of your cluster and making API calls to do a variety of management actions.

Start minikube
```
minikube start
```
check the minikube status
```
minikube status
```
check to ensure if there's any resources on the cluster
```
kubectl get pod
```
![alt text](images/1.3.png)

You can get helm chat to install prometheus operator [here](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
```
![alt text](images/1.4.png)

```
helm install prometheus prometheus-community/kube-prometheus-stack

```
![alt text](images/1.5.png)

```
kubectl get pod
```
![alt text](images/1.6.png)

To check every part that was created 
```
kubectl get all
```
we have the following:

2 Statefulsets
```
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus  # the actual prometheus server
```

3 Deployments
```
deployment.apps/prometheus-grafana                   
deployment.apps/prometheus-kube-prometheus-operator   
deployment.apps/prometheus-kube-state-metrics  # 
```

3 Replicaset
```
replicaset.apps/prometheus-grafana-648c7ff88d                    
replicaset.apps/prometheus-kube-prometheus-operator-6476b7f669   
replicaset.apps/prometheus-kube-state-metrics-95bc57444  
```

1 daemonset
```
daemonset.apps/prometheus-prometheus-node-exporter 
```

9 Services
```
service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,909

service/kubernetes                                ClusterIP   10.96.0.1        <none>        443/TCP             

service/prometheus-grafana                        ClusterIP   10.106.62.118    <none>        80/TCP               

service/prometheus-kube-prometheus-alertmanager   ClusterIP   10.101.105.32    <none>        9093/TCP,8080/TCP     

service/prometheus-kube-prometheus-operator       ClusterIP   10.101.148.163   <none>        443/TCP           

service/prometheus-kube-prometheus-prometheus     ClusterIP   10.107.101.248   <none>        9090/TCP,8080

service/prometheus-kube-state-metrics             ClusterIP   10.109.250.127   <none>        8080/TCP          

service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP      

service/prometheus-prometheus-node-exporter       ClusterIP   10.109.90.87     <none>        9100/TCP       
```

6 pods
```
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          3m44s
pod/prometheus-grafana-648c7ff88d-t9pq4                      3/3     Running   0          4m8s
pod/prometheus-kube-prometheus-operator-6476b7f669-nb7xw     1/1     Running   0          4m8s
pod/prometheus-kube-state-metrics-95bc57444-rx55q            1/1     Running   0          4m8s
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          3m43s
pod/prometheus-prometheus-node-exporter-clwrw                1/1     Running   0          4m8s
```

![alt text](images/1.7.png)

check the statefulset created and get the information in it
````
kubectl get statefulset
````
![alt text](images/1.8.png)

download the content in a yaml file
```
kubectl describe statefulset prometheus-prometheus-kube-prometheus-prometheus > prom.yaml
```
```
kubectl describe statefulset alertmanager-prometheus-kube-prometheus-alertmanager  > alert.yaml
```
check throught the prom.yaml. let's get check under the container where the pods are running

We can see the `image` is built on, the `port` is running on and under the `mount` where it gets its configuration data.

configuration file is the place where prometheus define what endpoints to scrape, 

It has all addresses of application where it gets expose/metrics

![alt text](images/1.9.png)

![alt text](images/1.10.png)

> [!NOTE]
> You may not need to know how all the files and the configuration works, but know **how to add/adjust alert rules** and **how to adjust Prometheus configuration**

### Access Grafana

```
kubectl get service
```
![alt text](images/1.11.png)

 All the services are clusterIp type which is the internal service. They are not open to external service request, they are all close. In production what we do is to configure ingress and point the ingress rules to prometheus grafana. But in this instance we are going to access grafana using port forward. 

 So, we are going to need deployment for that.
 ```
 kubectl get deployment
 ```
 ![alt text](images/1.12.png)

 ```
 kubectl get pod
 ```
![alt text](images/1.13.png)

 So we are going to check the port grafana is running
 ```
 kubectl logs prometheus-grafana-648c7ff88d-t9pq4 
 ```
 From the logs, the grafana is running on port 3000

 ![alt text](images/1.14.png)

 ```
 kubectl port-forward deployment/prometheus-grafana 3000
 ```
![alt text](images/1.15.png)

 The default login: User = admin, password = prom-operator 

 ![alt text](images/1.16.png)

 Under the manage we can see all the metrics that the prometheus is scrapping already through `Node Exporter`. We only one node which is minikube being manage currently, we can find it under Dashbord

 We can see the IP address of the minikube on the dashboard

  ![alt text](images/1.17.png)

 ![alt text](images/1.18.png)

 ![alt text](images/1.19.png)

 ![alt text](images/1.20.png)

 In addition to grafana, Prometheus also have its own UI, and we can access it also using port forward. From the prom.yaml files we downloaded earlier, we can see that prometheus is running on port `9090`

![alt text](images/1.21.png)

```
kubectl get pod
```
![alt text](images/1.22.png)

```
kubectl port-forward prometheus-prometheus-kube-prometheus-prometheus-0 9090
```
![alt text](images/1.23.png)

![alt text](images/1.24.png)

![alt text](images/1.25.png)

![alt text](images/1.26.png)

### Monitor mongoDB with prometheus 

We are going to deploy MongoDB App on the kubernetes cluster, and expose the data or metrics of the MongoDB application using MongoDB Exporter. Once the metrics are expose we are going to allow prometheus to scrape the metrics. We are going to do that using another component called **Service Monitor** 

```
kubectl get servicemonitor
```
![alt text](images/1.27.png)

These are the services that generated all the targets on the prometheus UI 

Let take one of the servicemonitor and check what it contain
```
kubectl get servicemonitor prometheus-kube-prometheus-alertmanager  -oyaml
```
![alt text](images/1.28.png)

You don't have to understand all that is written in the service monitor but there are some important label you must understand.

The `release` label, it allows prometheus to find service in the cluster and register them so that it can start scrapig the application or endpoint of the application that service monitor is pointing to.



This logic of how prometheus actually discovers service monitors using this attribute is configured in prometheus itself. 

To check the configuration we have to check the crd
```
kubectl get crd
```
![alt text](images/1.29.png)

These are the crds that prometheus has created, they are basically custom component for prometheus. Let's check one of them
```
kubectl get  prometheuses.monitoring.coreos.com  -oyaml
```

If we scroll down to the matchLabels under serviceMonitorSelector section, it basically says match all the service monitor that has **label** `release:prometheus`. Which means if you created a serviceMonitorComponent which doesn't have this label it will not be dicovered by prometheus.

![alt text](images/1.30.png)

so let's deploy MongoDB and configure it so that it becomes one of the target for prometheus to collect its metrics.

We have have a mongodb.yaml file in this folder, it contains depoyment and service we are going to use for this deployment.

```
kubectl apply -f mongodb.yaml
```
```
kubectl get pod
```
![alt text](images/1.31.png)


Now we have mongodb pod running and we monitor it with prometheus, the way to do that is by using a component called **Exporter**

check for exporter [here](https://prometheus.io/docs/instrumenting/exporters/). Search for mongoDB, and if you click on it will take us to the github repository for mongodb_exporter. The exporter are also available as contaier and we can search for them on [docker hub](https://hub.docker.com/explore).

There are 3 components we need to have when deploying an exporter
- Exporter application in docker image: This will exposes/metric endpoint
- A Service: for connecting to the exporter
- ServiceMonitor: That will tell prometheus that their a new endpoint to be scrapped

We can do this by writing a yaml file for the component configuration files, but the better way to do it is to get a helm chart that has everything in it and install it with one command. So, let's check for the helm chart. The search will take us to github repo of the [**Community**](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-mongodb-exporter) maintaining the helm chart.

![alt text](images/1.32.png)

![alt text](images/1.33.png)

![alt text](images/1.34.png)

![alt text](images/1.35.png)

![alt text](images/1.36.png)

**helm show values  prometheus-community/kube-prometheus-stack > values.yaml**


add the repo 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
![alt text](images/1.37.png)

we need to check which of the default configuration values we can override before installing the chart.
```
helm show values  prometheus-community/prometheus-mongodb-exporter > values.yaml
```

from the values.yaml we downloaded, we need to override the mongodb uri and additionallabels under the serviceMonitor. We need to add the release label to it to enable prometheus to scrap its metrics.

let's check for the service and see if the `mongodb service uri` configured in the value.yaml file we downloaded is correct with our service's port. You can open the downloaded file in Visual studio code editor.

![alt text](images/1.38.png)

![alt text](images/1.39.png)


We are going to delete all the other elements in the values file we don't need to override leaving the serviceMonitor. Save the file.

![alt text](images/1.41.png)

Install the mongodb-exporter chart and pass the parameters with the configure values.yaml

```
helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter -f values.yaml
```
![alt text](images/1.42.png)

let's check what we have installed with helm  now

```
helm ls
```
check the pod
```
kubectl get pod
```
check the Service
```
kubectl get svc
```
![alt text](images/1.43.png)

![alt text](images/1.44.png)

![alt text](images/1.45.png)


check serviceMonitor
```
kubectl get servicemonitor
```

> [!NOTE]
> the mongodb-exporter service monitor was not created. After troubleshooting, I notice i include **enabled: true** in the override configuration in the value.yaml file. The service was created after it was included.

![alt text](images/1.46.png)

![alt text](images/1.47.png)

![alt text](images/1.48.png)

Forwards the mongodb-exporter to the service port

![alt text](images/1.49.png)

```
kubectl port-forward service/mongodb-exporter-prometheus-mongodb-exporter 9216
```
![alt text](images/1.50.png)

let check the port in our web browser to see what data it's collecting
```
http://127.0.0.1:9216/
```
click on the metrics to check the data

![alt text](images/1.51.png)

we can see it is collecting metrics from mongodb

![alt text](images/1.52.png)

let check our prometheus, it must have discovered the mongodb-exporter through the serviceMonitor and scrap the mongodb metrics
```
127.0.0.1:9090 -> 9090
```
![alt text](images/1.53.png)

![alt text](images/1.54.png)


Lets check grafana to see if is displaying the metrics from mongodb
```
kubectl get deployment
```
![alt text](images/1.55.png)

forward the deployment to port 30

kubectl port-forward deployment/prometheus-grafana 3000

![alt text](images/1.56.png)

**Login with: username = admin**
**To get password, use the following command**
```
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

click on Dashboard > pod

![alt text](images/1.57.png)

![alt text](images/1.58.png)

![alt text](images/1.59.png)

![alt text](images/1.60.png)



