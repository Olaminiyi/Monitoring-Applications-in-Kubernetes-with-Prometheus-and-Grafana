# Monitoring Applications in Kubernetes with Prometheus and Grafana

In this project we will be setting up a Prometheus monitoring in Kubernetes cluster. Before we start the implementation, we are going to explain some terms below.

[**Prometheus**](https://www.tigera.io/learn/guides/prometheus-monitoring/): rometheus is an open-source technology designed to provide monitoring and alerting functionality for cloud-native environments, including Kubernetes. It can collect and store metrics as time-series data, recording information with a timestamp. It can also collect and record labels, which are optional key-value pairs.

why is Prometheus important?

modern DevOps becomes more and more complex to handle manually and therefore needs automation. So typically you have multiple servers that run containerized applications and there are hundreds of different processes running on that infrastructure and things are interconnected. So maintaining such setup to run smoothly without application downtimes is very challenging. imagine having such a complex infrastucture with loads of servers distributed over many locations and you have no insight of what is happening on the hardware level or on application level like errors, responses latency, hardware down or overloaded or maybe running out of resources etc. If something happens application may becomes unavailable to user and you must be able to identify out of the hundred of of things in the cluster went wrong and that could be difficult and time-consuming when debugging the system manaually.
What makes the process of debugging easier is to have a hool that  constantly monitor if services are running and alert the maintainer as soon as one service crashes so you know exactly what happened or even better it identify problems before they even occur and alerts the system administrators responsible for that infrasture to prevent that issue. 

How does prometheus work?

![alt text](images/1.2.png)

Prometheus consist of 3 main components
- Time Seriess Database: This is the main storage that prometheus uses for storing metrics data
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