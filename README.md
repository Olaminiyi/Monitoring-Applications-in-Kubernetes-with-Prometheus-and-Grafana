# Monitoring Applications in Kubernetes with Prometheus and Grafana

In this project we will be setting up a Prometheus monitoring for a third party app in Kubernetes cluster. Before we start the implementation, we are going to explain some terms below.

[**Prometheus**](https://www.tigera.io/learn/guides/prometheus-monitoring/): rometheus is an open-source technology designed to provide monitoring and alerting functionality for cloud-native environments, including Kubernetes. It can collect and store metrics as time-series data, recording information with a timestamp. It can also collect and record labels, which are optional key-value pairs.

why is Prometheus important?

Modern DevOps becomes more and more complex to handle manually and therefore needs automation. Typically you have multiple servers that run containerized applications and there are hundreds of different processes running on that infrastructure and things are interconnected. Maintaining such setup to run smoothly without application downtimes is very challenging. 

Imagine having such a complex infrastucture with loads of servers distributed over many locations and you have no insight of what is happening on the hardware level or on application level like errors, responses latency, hardware down or overloaded or maybe running out of resources etc. If something happens application may becomes unavailable to user and you must be able to identify out of the hundred of of things in the cluster went wrong and that could be difficult and time-consuming when debugging the system manaually.

What makes the process of debugging easier is to have a tool that  constantly monitor if services are running and alert the maintainer as soon as one service crashes so you know exactly what happened or even better it identify problems before they even occur and alerts the system administrators responsible for that infrasture to prevent that issue. 

How does prometheus work?

![alt text](images/1.2.png)

Prometheus consist of 3 main components
- Time Seriess Database: This is the main storage that prometheus uses for storing metrics data. Prometheus store agreggated metrics on **Disk(HDD/SSD)** and it can also optionally integrate with remote Storage Systems and the data is stored in a **Custom Time Series Format** and because of that you write prometheus into a relational database.  
- Data Retrieval Worker: This the components of the prometheus that pull and retrieve metrics from services and applications and forwards to Tthe Time Series Database
-  Prometheus Server: it has a web server or server API that accepts queries using PromQL queries for the stored data inthe in the database. the server api displays the metrics through Prometheus Web UI dashboard or any other data visualization applications such as Grafana.
 
 Below is the full Prometheus Architecture 

![alt text](images/1.1.png)

What does prometheus monitor?

Prometheus monitors everything such as Linus/Windows server, Single application, Apache Server, CPU Status, Services like Database and many more. Those things that prometheus monitors are called **Targets**. The units that the target monitors are called **Metrics**. And the metric could be CPU status, Memory/Disk Space Usage, Exception count, request count, request duration

How does Prometheus collect those metrics from target?

Prometheus collects metrics from target systems by scraping HTTP endpoints that expose these metrics in a specific format. Some of the methods are explain below:

1. **Target Configuration:** This can be achieve in 2 ways for different environments.
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

We will use Minikube to set up our cluster and Kubectl to interract with our cluster

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

You can get helm chat to install prometheus operator [here](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
```
```
helm install prometheus prometheus-community/kube-prometheus-stack

```
```
kubectl get pod
```
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