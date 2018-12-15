# Prometheus Monitoring on Kubernetes
This repository demonstrates how Prometheus can be used to monitor Kubernetes clusters.

Prometheus is an open source monitoring framework. As Kubernetes is spunned from Google's Borg and Prometheus draws a lot of inspiration from Borgmon(Google's internal monitoring system), it is actively used by community to monitor Kubernetes clusters.

## Pre-requisites:
Following are the pre-requisites required to run this project:
- Up and running Kubernetes cluster

## Setup:
- So let's get started with the setup:
- We are going to use Prometheus' official docker-image available in their official docker hub account(https://hub.docker.com/r/prom/prometheus/).

### Create a Namespace
- First we are going to create a Kubernetes namespace for all the monitoring components.
- Please execute following command to create a namespace:
```
kubectl create namespace monitoring
```
- After we need to assign permissions to this namespace so that Prometheus can fetch the metrices from Kubernetes API's.
- Create a file named clusterRole.yaml file as shown below:
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```
- Create the role using following command:
```
kubectl create -f clusterRole.yaml
```
### Create a Config Map
- We will create a config map with all the prometheus config and alerting rules, which will be mounted to the Prometheus container in /etc/prometheus as prometheus.yaml and prometheus.rules files. The prometheus.yaml contains all the configuration to dynamically discover pods and services running in the kubernetes cluster. prometheus.rules will contain all the alert rules for sending alerts to alert manager.
- Create a file named config-map.yaml as shown below:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: devopscube demo alert
      rules:
      - alert: High Pod Meory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Memory Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
        - role: endpoints
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      
      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
```
- Execute the following command to create the config map in Kubernetes:
```
kubectl create -f config-map.yaml
```
### Create a Prometheus Deployment
- Create a file named prometheus-deployment.yaml as shown below. In this configuration, we are mounting the Prometheus config map as a file inside /etc/prometheus. It uses the official Prometheus image from docker hub.
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:v2.1.0
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
  
        - name: prometheus-storage-volume
          emptyDir: {}
```
- Create a Kubernetes deployment using following command:
```
kubectl create -f prometheus-deployment.yaml
```
- You can check the created deployment using the following command:
```
kubectl get deployments --namespace=monitoring
```
### Connecting to Prometheus
- We can connect to Prometheus in two ways:
1. Using kubectl port-forwarding
2. Exposing the Prometheus deployment as a service with NodePort or a Load Balancer.
#### Using kubectl port-forwarding
- Using kubectl port forwarding, the pod can be accessed from workstation using a selected port on the localhost.
- First, get the Prometheus pod name:
```
kubectl get pods --namespace=monitoring
```
The output will look like the following:
```
➜  kubectl get pods --namespace=monitoring
NAME                                     READY     STATUS    RESTARTS   AGE
prometheus-monitoring-3331088907-hm5n1   1/1       Running   0          5m
```
- Execute the following command with pod name to access Prometheus from localhost port 8080.
```
kubectl port-forward prometheus-monitoring-3331088907-hm5n1 8080:9090 -n monitoring
```
- Now, we can access Prometheus on http://localhost:8080 on our browser.
#### Exposing Prometheus as a service
- To access the Prometheus dashboard over a IP or a DNS name, we need to expose it as kubernetes service.
- Create a file named prometheus-service.yaml. We will expose Prometheus on all kubernetes node IP’s on port 30000.
```
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
spec:
  selector: 
    app: prometheus-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 9090 
      nodePort: 30000
```
**Note:** If you are on AWS or Google Cloud, you can use Loadbalancer type, which will create a load balancer and points it to the service.
- Create the service using the following command:
```
kubectl create -f prometheus-service.yaml
```
- Once created, we can access the Prometheus dashboard using any Kubernetes node IP on port 30000(if you are on the cloud, make sure you have the right firewall rules for accessing the apps).
- Now if you go to status –> Targets, you will see all the Kubernetes endpoints connected to Prometheus automatically using service discovery. So you will get all kubernetes container and node metrics in Prometheus.
- You can head over the homepage and select the metrics you need from the drop-down and get the graph for the time range you mention.

### Reference:
- Prometheus also provides whole range of other features like alerting, flexible query language, pushing time series data. Please refer https://prometheus.io/docs/introduction/overview/#features for detailed information.
