
# End To End observability Stack

This project implements a comprehensive end-to-end observability stack. We utilized Prometheus for collecting infrastructure and application-level metrics, with Grafana deployed for advanced visualization and dashboarding of these metrics. For application metrics, we implemented service discovery to scrape data from endpoints instrumented with OpenTelemetry.Alertmanager was configured to send notifications based on the state of the metrics. For log collection, we deployed the EFK stack (Elasticsearch, Fluent Bit, and Kibana) to aggregate and visualize application logs, also instrumented with OpenTelemetry. Jaeger was employed for distributed tracing to monitor and analyze system performance.OpenTelemetry served as the instrumentation framework for metrics, logs, and tracing. The solution was deployed on an Amazon EKS Kubernetes cluster, with Amazon EBS used as the persistent storage solution for Elasticsearch.



# Installation & Configurations

## Step 1: Create EKS Cluster
## Prerequisites
- Download and Install AWS Cli.
- Setup and configure AWS CLI using the aws configure command.
- Install and configure eksctl using the steps mentioned here.
- Install and configure kubectl as mentioned here.

```bash
  eksctl create cluster --name=observability \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
```
```bash
  eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster observability \
    --approve
```
```bash
  eksctl create nodegroup --cluster=observability \
                        --region=us-east-1 \
                        --name=observability-ng-private \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=20 \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking

# Update ./kube/config file
aws eks update-kubeconfig --name observability
```
## Step 2: Install kube-prometheus-stack
```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
## Step 3: Deploy the chart into a new namespace "monitoring"
```bash
 kubectl create ns monitoring
```
```bash
 cd EKS-PROMETHEUS-INSTALLATION

helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
-f ./custom_kube_prometheus_stack.yml
```
## Step 4: Verify the Installation
```bash
 kubectl get all -n monitoring
```
- Prometheus UI:
```bash
 kubectl port-forward service/prometheus-operated -n monitoring 9090:9090
```
NOTE: If you are using an EC2 Instance or Cloud VM, you need to pass --address 0.0.0.0 to the above command. Then you can access the UI on instance-ip:port
- Grafana UI: password is prom-operator
```bash
kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
```
- Alertmanager UI:
```bash
kubectl port-forward service/alertmanager-operated -n monitoring 9093:9093
```
## Example
You can use this to query your prometheus to get infrastructure metrics
```bash
container_cpu_usage_seconds_total{namespace="kube-system", endpoint="https-metrics"}
```
## Implement custom metrics
```bash
NOTE: Update your email id and password in the alertmangerconfig.yml before you deploy
 cd alerts-alertmanager-servicemonitor-manifest

kubectl apply -k alerts-alertmanager-servicemonitor-manifest
```
# EFK Stack (Elasticsearch, Fluentbit, Kibana) Installation
## Create IAM Role for Service Account
```bash
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster observability \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```
## Retrieve IAM Role ARN
```bash
ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)
```
## Deploy EBS CSI Driver
```bash
eksctl create addon --cluster observability --name aws-ebs-csi-driver --version latest \
    --service-account-role-arn $ARN --force
```
## Create Namespace for Logging
```bash
kubectl create namespace logging
```
## Install Elasticsearch on K8s
```bash
helm repo add elastic https://helm.elastic.co

helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true elastic/elasticsearch -n logging
```
## Retrieve Elasticsearch Username & Password
```bash
# for username
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# for password
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
```
## Install Kibana
```bash
helm install kibana --set service.type=LoadBalancer elastic/kibana -n logging
```
## Install Fluentbit with Custom Values/Configurations
 Note: Please update the HTTP_Passwd field in the fluentbit-values.yml file with the password retrieved earlier in step 6: (i.e NJyO47UqeYBsoaEU)"
```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging
```
# Setting Up Jaeger
## Export Elasticsearch CA Certificate
```bash
kubectl get secret elasticsearch-master-certs -n logging -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca-cert.pem
```
## Create Tracing Namespace
```bash
kubectl create ns tracing
```
## Create ConfigMap for Jaeger's TLS Certificate
```bash
kubectl create configmap jaeger-tls --from-file=ca-cert.pem -n tracing
```
## Create Secret for Elasticsearch TLS
```bash
kubectl create secret generic es-tls-secret --from-file=ca-cert.pem -n tracing
```
## Add Jaeger Helm Repository
```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts

helm repo update
```
## Install Jaeger with Custom Values
Note: Please cd JAEGER and update the password field and other related field in the jaeger-values.yaml file with the elastic search password retrieved earlier
```bash
helm install jaeger jaegertracing/jaeger -n tracing --values jaeger-values.yaml

helm repo update
```
## Port Forward Jaeger Query Service
```bash
kubectl port-forward svc/jaeger-query 8080:80 -n tracing
```
##  Clean Up
```bash
eksctl delete cluster --name observability
```
