### Project name:

Configure Alerting for our Application

### Technologies used:

Prometheus, Kubernetes, Linux

### Project Description:

1- Setup EKS cluster using eksctl

2- Deploy microservices to k8s cluster

3- Deploy Prometheus, Alert Manager and Grafana in cluster as part of the Prometheus Operator using Helm chart

4-Configure our Monitoring Stack to notify us whenever CPU usage > 50% or Pod cannot start

a-Configure Alert Rules in Prometheus Server

b-Configure Alertmanager with Email Receiver

### Instruction

###### Step 1: Create eksctl in default region

```
aws configure
```

```
eksctl create cluster -name my-cluster
```

# eksctl will configure the credentials with eks cluster for kubectl and save it to kubeconfig

![image](images/Screenshot%202023-04-25%20at%209.16.33%20am.png)
![image](images/Screenshot%202023-04-25%20at%209.21.19%20am.png)

###### Step 2: Deploy microservices to my-cluster

```
kubectl apply -f config-microservices.yaml
```

![image](images/Screenshot%202023-04-25%20at%209.19.21%20am.png)

###### Step 3: Deploy prometheus, alert manager and Grafana via helm chart

#Add repo

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

#Update the repo chart

```
helm repo update
```

#Create a new namespace for prometheus

```
kubectl create namespace monitoring
```

#Deploy prometheus via helm chart

```
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

![image](images/Screenshot%202023-04-25%20at%209.27.39%20am.png)

###### Step 4: Forward-port to Prometheus UI

```
kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &
```

![image](images/Screenshot%202023-04-25%20at%2011.00.16%20am.png)

###### Step 5: Forward-port to Grafana UI

```
kubectl port-forward service/monitoring-grafana 8080:80 -n monitoring &
```

![image](images/Screenshot%202023-04-25%20at%2011.02.09%20am.png)
![image](images/Screenshot%202023-04-25%20at%2010.19.02%20am.png)

###### Step 6: Simulate a request to microservices front-end to see the Grafana UI

#Create a test pod inside the cluster to send requests to front-end pod

```
kubectl run curl-test --image=radial/busyboxplus:curl -i --ty --rm
```

#Write a test.sh to send 100000 requests to front-end pod endpoint
![image](images/FireShot%20Capture%20028%20-%204%20-%20Introduction%20to%20Grafana%20-%2023_25%20-%2011_11_%20-%20techworld-with-nana.teachable.com.png)

#The output of Grafana
![image](images/Screenshot%202023-04-25%20at%2011.07.13%20am.png)

###### Step 7- Create two new alert rules to prometheus : CPU >50% & Pod cannot start

```
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main-rules
  namespace: monitoring
  labels:
    app: kube-prometheus-stack
    release: monitoring
spec:
  groups:
    - name: main.rules
      rules:
        - alert: HostHighCPULoad
          expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m]))*100) > 50
          for: 2m
          labels:
            severity: warning
            namespace: monitoring
          annotations:
            description: "CPU load on host is over 50%\n value={{$value}}\n Instance={{$labels.instance}}"
            summary: "Host CPU load high"
        - alert: KubernetesPodCrashLooping
          expr: kube_pod_container_status_restarts_total > 5
          for: 0m
          labels:
            severity: critical
            namespace: monitoring
          annotations:
            description: "Pod {{$labels.pod}} is crash looping\n value={{$value}}"
            summary: "Kubernetes pod crash loop"
```

#Apply the alert-rules.yaml to cluster so prometheus can pick them up

```
kubectl apply -f alert-rules.yaml
```

```
kubectl get PrometheusRule -n monitoring
```

![image](images/Screenshot%202023-04-25%20at%207.49.15%20pm.png)

###### Step 8- Deploy a test-pod into the current cluster to test cpu > 50 alert rule

#Deploy a test-pod

```
kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 30s --metrics-briefy
```

```
kubectl get pod
```

![images](images/Screenshot%202023-04-25%20at%208.28.16%20pm.png)

![images](images/Screenshot%202023-04-25%20at%208.29.50%20pm.png)
