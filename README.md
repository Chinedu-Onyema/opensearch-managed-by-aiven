# Introduction to OpenSearch and Log Analytics with Aiven and Fluent Bit

## INTRODUCTION
OpenSearch is a powerful, open-source search and analytics engine forked from Elasticsearch and Kibana in 2021 and governed under the Linux Foundation. 

Designed for enterprise scalability, OpenSearch provides a comprehensive suite for log analytics, trace storage, full-text search, and observability dashboards.  

When deploying production cloud infrastructure, organizations frequently leverage fully managed cloud data platforms like Aiven to eliminate the overhead of manual database maintenance, patching, high availability, and scaling. 

This phase details how to provision a managed OpenSearch cluster via Aiven, deploy a lightweight log-generating workload (busybox) in a Kubernetes cluster, configure Fluent Bit as a DaemonSet to tail container logs, securely ship logs to OpenSearch, and visualize application data using OpenSearch Dashboards.

#### ACCESS PROJECT MATERIALS
#### PDF GUIDE: [INTRODUCTION TO OPENSEARCH.pdf](https://github.com/user-attachments/files/34142587/INTRODUCTION.TO.OPENSEARCH.pdf)
#### WATCH VIDEO WALKTHROUGH HERE: https://youtu.be/o8EfK4sEJbc      


### STEP-BY-STEP DEPLOYMENT AND INTEGRATION GUIDE

##### STEP 1: Provision Managed OpenSearch on Aiven
1) Create a free trial account on Aiven without requiring a credit card to receive starting cloud credits.   
2) From your default project dashboard, navigate to Services and select OpenSearch.
3) Choose the Free service tier and click Create Service.
4) Wait for the virtual machine instance to reach a Running state.
5) Access the Connection Information tab to retrieve your OpenSearch endpoint credentials (Host, Port, HTTP_USER, and HTTP_PASSWD) required for log ingestion.


##### STEP 2: Deploy Log Generator Workload in Kubernetes
1) Initialize your local Minikube cluster:

   <PRE>minikube status</PRE> 
   <PRE>minikube start</PRE>

2) Create a working directory and write a Kubernetes deployment manifest (deployment-log-generator.yml) running a continuous log-producing busybox container:

   <PRE>mkdir -p opensearch && cd opensearch</PRE>
   
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  labels:
    app: log-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
      - name: log-generator
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            while true; do
              echo "$(date) INFO: Application started";
              echo "$(date) DEBUG: Debugging app logic";
              echo "$(date) ERROR: An error occurred!";
              sleep 5;
            done
```

3) Apply the deployment and verify pod logs:

   <PRE>eval $(minikube docker-env)</PRE>
   <PRE>kubectl apply -f deployment-log-generator.yml</PRE>
   <PRE>kubectl get pods</PRE>
   <PRE>kubectl logs deploy/log-generator</PRE>


##### STEP 3: Configure and Deploy Fluent Bit for OpenSearch Ingestion
1) Create a Kubernetes ConfigMap (fluent-bit-configuration.yml) in the logging namespace, inserting your Aiven OpenSearch connection parameters under the [OUTPUT] block:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Daemon        Off
        Log_Level     info
        Parsers_File  parsers.conf
    [INPUT]
        Name          tail
        Path          /var/log/containers/*.log
        Parser        docker
        Tag           kube.*
        Refresh_Interval 5
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
    [FILTER]
        Name          kubernetes
        Match         kube.*
        Kube_URL      https://kubernetes.default.svc:443
        Merge_Log     On
        K8S-Logging.Parser On
        K8S-Logging.Exclude Off
    [OUTPUT]
        Name          opensearch
        Match         *
        Host          <your-aiven-opensearch-host>
        Port          <your-aiven-opensearch-port>
        Index         fluentbit
        HTTP_User     <your-aiven-username>
        HTTP_Passwd   <your-aiven-password>
        TLS           On
        TLS.verify    Off
        Suppress_Type_Name On
        Include_Tag_Key On
        Logstash_Format On
        Logstash_Prefix kubernetes
        Replace_Dots  On
        Retry_Limit   False
        Write_Operation create
  parsers.conf: |
    [PARSER]
        Name          docker
        Format        json
        Time_Key      time
        Time_Format   %Y-%m-%dT%H:%M:%S.%L
        Time_Keep     On
        Decode_Field_As escaped_utf8 log do_next
        Decode_Field_As json log
```

2) Create the logging namespace, service account, and deploy Fluent Bit as a DaemonSet (fluent-daemonset-deployment.yml):

   <PRE>kubectl create ns logging</PRE>
   <PRE>kubectl apply -f fluent-bit-configuration.yml</PRE>
   <PRE>kubectl create sa fluent-bit -n logging</PRE>
   <PRE>kubectl apply -f fluent-daemonset-deployment.yml</PRE>
   <PRE>kubectl get pods -n logging</PRE>


##### STEP 4: Verify and Visualize Logs in OpenSearch DashboardsVerify Index Creation: 

1) Navigate to your Aiven OpenSearch service console, click Data > Indexes, and confirm the appearance of the kubernetes-<date> index populated by Fluent Bit.
2) Configure OpenSearch Dashboards:Access OpenSearch Dashboards from the Aiven overview tab using your authentication credentials.
3) Open the menu, select Stack Management > Index Patterns, and click Create index pattern.
4) Enter kubernetes* as the index pattern name, choose "I don't want to use the time filter", and save.
5) Query Application Logs: Go to the Discover tab, select your kubernetes* data view, and filter for search terms such as An error occurred! to inspect real-time Kubernetes cluster telemetry.

  
