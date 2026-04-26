# Production-Grade EKS Monitoring Stack  
## Prometheus + Grafana + Alertmanager + CloudWatch + Slack

This README explains the complete monitoring setup we built on Amazon EKS.

It is written for a beginner who does not just want commands, but also wants to understand:

- What each component does
- Why we installed it
- What problem it solves
- What errors happened
- How we diagnosed each error
- How we fixed each issue
- How to validate the setup
- What is still pending after application deployment

> **Current scope:** This setup covers cluster/platform monitoring. Application-specific monitoring will be added after an application is deployed.

---

## Table of Contents

1. [Final Architecture](#1-final-architecture)
2. [What We Built](#2-what-we-built)
3. [Important Concepts Before Starting](#3-important-concepts-before-starting)
4. [Namespaces](#4-namespaces)
5. [Helm Installation](#5-helm-installation)
6. [ACM Wildcard Certificate](#6-acm-wildcard-certificate)
7. [EKS Cluster Variables](#7-eks-cluster-variables)
8. [IAM OIDC Provider and IRSA](#8-iam-oidc-provider-and-irsa)
9. [AWS Load Balancer Controller](#9-aws-load-balancer-controller)
10. [Troubleshooting AWS Load Balancer Controller](#10-troubleshooting-aws-load-balancer-controller)
11. [EBS CSI Driver](#11-ebs-csi-driver)
12. [gp3 StorageClass](#12-gp3-storageclass)
13. [Grafana Admin Secret](#13-grafana-admin-secret)
14. [kube-prometheus-stack Values File](#14-kube-prometheus-stack-values-file)
15. [Install Prometheus, Grafana, and Alertmanager](#15-install-prometheus-grafana-and-alertmanager)
16. [Grafana Ingress and ALB](#16-grafana-ingress-and-alb)
17. [Route 53 DNS Record](#17-route-53-dns-record)
18. [Grafana and Prometheus Validation](#18-grafana-and-prometheus-validation)
19. [Prometheus Metrics Explained](#19-prometheus-metrics-explained)
20. [Production Prometheus Alert Rules](#20-production-prometheus-alert-rules)
21. [CloudWatch Observability Add-on](#21-cloudwatch-observability-add-on)
22. [Troubleshooting CloudWatch IAM Permissions](#22-troubleshooting-cloudwatch-iam-permissions)
23. [CloudWatch Logs and Metrics Validation](#23-cloudwatch-logs-and-metrics-validation)
24. [Grafana CloudWatch Datasource](#24-grafana-cloudwatch-datasource)
25. [Troubleshooting Grafana CloudWatch Datasource](#25-troubleshooting-grafana-cloudwatch-datasource)
26. [Alertmanager Slack Integration](#26-alertmanager-slack-integration)
27. [Slack Alert Test](#27-slack-alert-test)
28. [Troubleshooting Slack Alerts](#28-troubleshooting-slack-alerts)
29. [Cleanup Test Alert](#29-cleanup-test-alert)
30. [Final Status](#30-final-status)
31. [Pending After Application Deployment](#31-pending-after-application-deployment)
32. [Production Hardening](#32-production-hardening)
33. [How to Think Like a Troubleshooter](#33-how-to-think-like-a-troubleshooter)
34. [Interview Explanation](#34-interview-explanation)

---

# 1. Final Architecture

```text
EKS Cluster
  |
  |-- kube-prometheus-stack
  |     |-- Prometheus
  |     |-- Grafana
  |     |-- Alertmanager
  |     |-- kube-state-metrics
  |     |-- node-exporter
  |
  |-- AWS Load Balancer Controller
  |     |-- Creates AWS ALB for Grafana Ingress
  |
  |-- EBS CSI Driver
  |     |-- Creates gp3 EBS volumes for persistent monitoring data
  |
  |-- CloudWatch Observability Add-on
  |     |-- CloudWatch Agent
  |     |-- Fluent Bit
  |     |-- Container Insights metrics and logs
  |
  |-- Alertmanager
        |-- Sends alerts to Slack
```

Final Grafana URL:

```text
https://grafana.learnwithmedevops.online
```

---

# 2. What We Built

We built a production-style monitoring stack for an EKS cluster.

## Main Components

| Component | Purpose |
|---|---|
| Prometheus | Collects and stores metrics |
| Grafana | Visualizes metrics and logs using dashboards |
| Alertmanager | Sends alerts to Slack |
| kube-state-metrics | Exposes Kubernetes object status metrics |
| node-exporter | Exposes node CPU, memory, disk, network metrics |
| CloudWatch Agent | Sends EKS/container metrics to CloudWatch |
| Fluent Bit | Sends container logs to CloudWatch Logs |
| AWS Load Balancer Controller | Creates ALB from Kubernetes Ingress |
| EBS CSI Driver | Creates EBS volumes for Kubernetes PVCs |
| ACM | Provides HTTPS certificate |
| Route 53 | Provides DNS name for Grafana |
| Slack Webhook | Receives alert notifications |

---

# 3. Important Concepts Before Starting

## What is monitoring?

Monitoring means continuously collecting system health data such as:

```text
CPU
Memory
Disk
Network
Pod health
Node health
Container restarts
Application errors
Latency
HTTP 5xx errors
```

## What is observability?

Observability is broader than monitoring. It helps answer:

```text
What is broken?
Where is it broken?
Why is it broken?
When did it start?
What changed?
```

Observability usually has three pillars:

```text
Metrics
Logs
Traces
```

In this setup:

```text
Metrics = Prometheus + CloudWatch Container Insights
Logs    = CloudWatch Logs using Fluent Bit
Alerts  = Alertmanager + Slack
Dashboards = Grafana
```

## What is Prometheus?

Prometheus is a metrics database. It periodically scrapes HTTP endpoints that expose metrics.

Example:

```text
node-exporter exposes node metrics
kube-state-metrics exposes Kubernetes metrics
Prometheus scrapes those metrics
Grafana queries Prometheus
```

## What is Grafana?

Grafana is a dashboard and visualization tool. It does not usually collect metrics by itself. It connects to data sources like:

```text
Prometheus
CloudWatch
Loki
Elasticsearch
```

In this setup:

```text
Grafana reads Kubernetes metrics from Prometheus
Grafana reads AWS metrics/logs from CloudWatch
```

## What is Alertmanager?

Alertmanager receives alerts from Prometheus and sends notifications to channels like:

```text
Slack
Email
PagerDuty
Microsoft Teams
Opsgenie
```

## What is CloudWatch Observability Add-on?

It is an AWS managed add-on for EKS that installs:

```text
CloudWatch Agent
Fluent Bit
```

It collects:

```text
EKS node metrics
Pod metrics
Container metrics
Container logs
Performance logs
```

## What is IRSA?

IRSA means **IAM Roles for Service Accounts**.

In Kubernetes, pods run using ServiceAccounts. In AWS, permissions are given using IAM roles.

IRSA connects them:

```text
Kubernetes ServiceAccount
        ↓
IAM Role
        ↓
AWS permissions
```

We used IRSA for:

```text
AWS Load Balancer Controller
Grafana CloudWatch datasource
```

## What is ALB Ingress?

In Kubernetes, an Ingress exposes applications externally.

In AWS EKS, the AWS Load Balancer Controller watches Ingress resources and creates an AWS Application Load Balancer.

Flow:

```text
Grafana Ingress
      ↓
AWS Load Balancer Controller
      ↓
AWS ALB
      ↓
Route 53 DNS
      ↓
https://grafana.learnwithmedevops.online
```

---

# 4. Namespaces

## Command

```bash
kubectl create namespace monitoring
kubectl create namespace logging
```

## Why we did this

Namespaces separate workloads logically.

```text
monitoring namespace:
  Prometheus
  Grafana
  Alertmanager
  kube-state-metrics
  node-exporter

logging namespace:
  reserved for future ELK/logging stack
```

## How to validate

```bash
kubectl get ns
```

Expected:

```text
monitoring
logging
```

---

# 5. Helm Installation

## Problem

When we tried to use Helm:

```text
helm: command not found
```

## What this means

Helm was not installed on the Ubuntu machine.

Helm is required because we installed Prometheus/Grafana and AWS Load Balancer Controller using Helm charts.

## Fix

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

## Add Prometheus Helm repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## Add AWS EKS Helm repo

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

## Why we added these repos

```text
prometheus-community repo:
  contains kube-prometheus-stack chart

eks repo:
  contains aws-load-balancer-controller chart
```

---

# 6. ACM Wildcard Certificate

## Goal

Use one certificate for multiple subdomains:

```text
grafana.learnwithmedevops.online
kibana.learnwithmedevops.online
argocd.learnwithmedevops.online
jenkins.learnwithmedevops.online
```

## Command

```bash
aws acm request-certificate \
  --domain-name "*.learnwithmedevops.online" \
  --subject-alternative-names "learnwithmedevops.online" \
  --validation-method DNS \
  --region us-east-1
```

## Why wildcard certificate?

A wildcard certificate like:

```text
*.learnwithmedevops.online
```

can secure:

```text
grafana.learnwithmedevops.online
kibana.learnwithmedevops.online
argocd.learnwithmedevops.online
jenkins.learnwithmedevops.online
```

It does not cover deeper levels like:

```text
api.dev.learnwithmedevops.online
```

For that, you need:

```text
*.dev.learnwithmedevops.online
```

## Certificate ARN

```text
arn:aws:acm:us-east-1:590999018668:certificate/3d3f6e84-fa39-44e0-bf21-c65b70874867
```

## Get DNS validation record

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:590999018668:certificate/3d3f6e84-fa39-44e0-bf21-c65b70874867 \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions[*].ResourceRecord"
```

Validation record:

```text
Name:
_d34bac15f0db3f123a32c0608c22c134.learnwithmedevops.online.

Type:
CNAME

Value:
_208d2ba95cf076f8bc334cdd2ae8f6e9.jkddzztszm.acm-validations.aws.
```

## What we did in Route 53

Created CNAME record:

```text
_d34bac15f0db3f123a32c0608c22c134.learnwithmedevops.online
→
_208d2ba95cf076f8bc334cdd2ae8f6e9.jkddzztszm.acm-validations.aws.
```

## Validate certificate

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:590999018668:certificate/3d3f6e84-fa39-44e0-bf21-c65b70874867 \
  --region us-east-1 \
  --query "Certificate.Status" \
  --output text
```

Expected:

```text
ISSUED
```

## Why certificate must be issued before ALB HTTPS works

The ALB HTTPS listener needs a valid ACM certificate. If the certificate is still:

```text
PENDING_VALIDATION
```

then ALB cannot correctly serve HTTPS.

---

# 7. EKS Cluster Variables

## Commands

```bash
export CLUSTER_NAME=day20-eks
export REGION=us-east-1
export ACCOUNT_ID=590999018668
```

## Why we used variables

Instead of typing the same values repeatedly, we exported variables.

Example:

```bash
aws eks describe-cluster --name $CLUSTER_NAME --region $REGION
```

## Validate nodes

```bash
kubectl get nodes
```

Observed nodes:

```text
ip-10-0-1-163.ec2.internal
ip-10-0-2-58.ec2.internal
ip-10-0-3-249.ec2.internal
```

## What this means

Your Kubernetes cluster had 3 worker nodes ready to run pods.

---

# 8. IAM OIDC Provider and IRSA

## Command

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --region $REGION \
  --approve
```

Output:

```text
IAM Open ID Connect provider is already associated with cluster "day20-eks" in "us-east-1"
```

## What this means

OIDC was already enabled for this EKS cluster.

## Why OIDC matters

Without OIDC, pods cannot assume IAM roles through IRSA.

IRSA flow:

```text
Pod
  ↓
Kubernetes ServiceAccount
  ↓
IAM OIDC Provider
  ↓
IAM Role
  ↓
AWS permissions
```

If OIDC or trust policy is wrong, you will see errors like:

```text
AccessDenied: sts:AssumeRoleWithWebIdentity
```

We saw this later for AWS Load Balancer Controller.

---

# 9. AWS Load Balancer Controller

## Why we need it

Grafana Ingress uses:

```yaml
ingressClassName: alb
```

That means Kubernetes expects AWS Load Balancer Controller to create an AWS ALB.

Without this controller:

```text
Ingress exists
but ALB is not created
ADDRESS remains empty
```

## Download IAM policy

```bash
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

## Create IAM policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Policy ARN:

```text
arn:aws:iam::590999018668:policy/AWSLoadBalancerControllerIAMPolicy
```

## Try creating IAM service account

```bash
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --region=$REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## Issue

`eksctl` skipped the service account because it thought it already existed.

This caused problems later.

## Get VPC ID

```bash
export VPC_ID=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

echo $VPC_ID
```

VPC:

```text
vpc-0d785093f6a1c1286
```

## Install controller

```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$REGION \
  --set vpcId=$VPC_ID
```

## Validate

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get ingressclass
```

Expected:

```text
aws-load-balancer-controller 2/2 Running
alb
```

---

# 10. Troubleshooting AWS Load Balancer Controller

## Problem 1: Deployment was 0/2

Output:

```text
aws-load-balancer-controller 0/2
```

## Check pods

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```

No pods were shown.

## Check ReplicaSet

```bash
kubectl get rs -n kube-system | grep aws-load-balancer
kubectl describe rs -n kube-system <replicaset-name>
```

## Error found

```text
Error creating pods:
serviceaccount "aws-load-balancer-controller" not found
```

## Meaning

Deployment was trying to create pods using this ServiceAccount:

```text
kube-system/aws-load-balancer-controller
```

But the ServiceAccount did not exist.

## Fix

```bash
kubectl create serviceaccount aws-load-balancer-controller -n kube-system
```

Annotate it with IAM role:

```bash
kubectl annotate serviceaccount aws-load-balancer-controller \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::590999018668:role/AmazonEKSLoadBalancerControllerRole \
  --overwrite
```

---

## Problem 2: IAM role did not exist

Command failed:

```bash
aws iam update-assume-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-document file://alb-controller-trust-policy.json
```

Error:

```text
NoSuchEntity:
The role with name AmazonEKSLoadBalancerControllerRole cannot be found.
```

## Meaning

The ServiceAccount was pointing to an IAM role that was not created.

So pods could not assume the role.

## Create trust policy

```bash
export OIDC_ID=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)

export OIDC_PROVIDER="oidc.eks.${REGION}.amazonaws.com/id/${OIDC_ID}"

cat > alb-controller-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com",
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
EOF
```

## Create role

```bash
aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document file://alb-controller-trust-policy.json
```

## Attach policy

```bash
aws iam attach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::590999018668:policy/AWSLoadBalancerControllerIAMPolicy
```

## Restart controller

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
kubectl rollout status deployment aws-load-balancer-controller -n kube-system
```

Final result:

```text
aws-load-balancer-controller 2/2 Running
```

## Troubleshooting lesson

When an EKS pod needs AWS permission, check:

```text
1. Does the Kubernetes ServiceAccount exist?
2. Does the ServiceAccount have eks.amazonaws.com/role-arn annotation?
3. Does the IAM role exist?
4. Does the IAM role trust policy allow this ServiceAccount?
5. Does the IAM role have required permissions?
```

---

# 11. EBS CSI Driver

## Why we need it

Prometheus, Grafana, and Alertmanager need persistent storage.

Without persistent storage:

```text
Grafana dashboards/config can be lost
Prometheus metrics can be lost
Alertmanager state can be lost
```

In EKS, Kubernetes needs EBS CSI Driver to dynamically create EBS volumes.

## Create IAM role for EBS CSI

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --region $REGION \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

## Get role ARN

```bash
export EBS_CSI_ROLE_ARN=$(aws iam get-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --query "Role.Arn" \
  --output text)

echo $EBS_CSI_ROLE_ARN
```

## Install add-on

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --region $REGION \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn $EBS_CSI_ROLE_ARN
```

## Validate

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

Expected:

```text
ebs-csi-controller Running
ebs-csi-node Running
```

---

# 12. gp3 StorageClass

## Why gp3?

`gp3` is the newer general-purpose EBS volume type. It is commonly used in production because it provides good performance and cost efficiency.

## Create StorageClass

```bash
cat > gp3-storageclass.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  encrypted: "true"
EOF

kubectl apply -f gp3-storageclass.yaml
```

## Validate

```bash
kubectl get storageclass
```

Expected:

```text
gp3 ebs.csi.aws.com
```

## What `WaitForFirstConsumer` means

The EBS volume is created only when a pod needs it.

This helps Kubernetes choose the correct Availability Zone.

---

# 13. Grafana Admin Secret

## Why we used a Secret

We did not want to hardcode the Grafana password directly into Helm values.

## Command

```bash
kubectl create secret generic grafana-admin-secret \
  -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='Grafana@12345'
```

## Production note

For real production:

```text
Use stronger password
Store secret in External Secrets / AWS Secrets Manager
Do not commit passwords to Git
```

---

# 14. kube-prometheus-stack Values File

This file controls the monitoring stack installation.

Create:

```bash
cat > kube-prometheus-values.yaml << 'EOF'
grafana:
  enabled: true

  admin:
    existingSecret: grafana-admin-secret
    userKey: admin-user
    passwordKey: admin-password

  persistence:
    enabled: true
    size: 10Gi
    storageClassName: gp3

  service:
    type: ClusterIP

  ingress:
    enabled: true
    ingressClassName: alb
    hosts:
      - grafana.learnwithmedevops.online
    path: /
    annotations:
      alb.ingress.kubernetes.io/group.name: platform-tools
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:590999018668:certificate/3d3f6e84-fa39-44e0-bf21-c65b70874867
      alb.ingress.kubernetes.io/ssl-redirect: "443"

prometheus:
  prometheusSpec:
    replicas: 2
    retention: 15d
    retentionSize: 40GB

    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Gi

    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2
        memory: 4Gi

    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

alertmanager:
  enabled: true
  alertmanagerSpec:
    replicas: 2

    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi

kube-state-metrics:
  enabled: true

prometheus-node-exporter:
  enabled: true
  tolerations:
    - operator: Exists
EOF
```

## Explanation of important fields

### Grafana

```yaml
grafana:
  enabled: true
```

Installs Grafana.

### Grafana persistence

```yaml
persistence:
  enabled: true
  size: 10Gi
  storageClassName: gp3
```

Keeps Grafana data on EBS volume.

### Grafana service

```yaml
service:
  type: ClusterIP
```

Grafana is not exposed directly by LoadBalancer service. It is exposed through Ingress and ALB.

### Grafana ingress

```yaml
ingressClassName: alb
```

Tells Kubernetes to use AWS Load Balancer Controller.

### ALB group

```yaml
alb.ingress.kubernetes.io/group.name: platform-tools
```

This allows multiple ingresses to share the same ALB later.

### HTTPS certificate

```yaml
alb.ingress.kubernetes.io/certificate-arn: <ACM ARN>
```

Attaches ACM certificate to ALB HTTPS listener.

### Prometheus replicas

```yaml
replicas: 2
```

Runs two Prometheus pods for better availability.

### Prometheus retention

```yaml
retention: 15d
retentionSize: 40GB
```

Keeps metrics for 15 days or until storage limit.

### Prometheus storage

```yaml
storage: 50Gi
```

Each Prometheus replica gets persistent EBS storage.

### Alertmanager replicas

```yaml
replicas: 2
```

Runs two Alertmanager pods for high availability.

---

# 15. Install Prometheus, Grafana, and Alertmanager

## Command

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f kube-prometheus-values.yaml
```

## Validate pods

```bash
kubectl get pods -n monitoring
```

Expected:

```text
monitoring-grafana
prometheus-monitoring-kube-prometheus-prometheus-0
prometheus-monitoring-kube-prometheus-prometheus-1
alertmanager-monitoring-kube-prometheus-alertmanager-0
alertmanager-monitoring-kube-prometheus-alertmanager-1
monitoring-kube-state-metrics
monitoring-prometheus-node-exporter
monitoring-kube-prometheus-operator
```

## Validate services

```bash
kubectl get svc -n monitoring
```

## Validate PVCs

```bash
kubectl get pvc -n monitoring
```

Expected:

```text
monitoring-grafana Bound 10Gi gp3
Prometheus PVCs Bound
Alertmanager PVCs Bound
```

---

# 16. Grafana Ingress and ALB

## Check ingress

```bash
kubectl get ingress -n monitoring
```

Expected:

```text
monitoring-grafana
CLASS: alb
HOST: grafana.learnwithmedevops.online
ADDRESS: k8s-platformtools-xxxx.us-east-1.elb.amazonaws.com
```

## Check details

```bash
kubectl describe ingress monitoring-grafana -n monitoring
```

Expected event:

```text
SuccessfullyReconciled
```

## What this means

AWS Load Balancer Controller successfully:

```text
Created ALB
Created target group
Created listener on 80
Created listener on 443
Attached ACM certificate
Registered Grafana pod IP as target
```

## If ADDRESS is empty

Check:

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller --all-containers=true --tail=100
kubectl describe ingress monitoring-grafana -n monitoring
```

Common causes:

```text
AWS Load Balancer Controller not running
Wrong IAM role
Wrong IRSA trust policy
Missing subnet tags
ACM certificate not issued
Invalid ingress annotations
```

---

# 17. Route 53 DNS Record

## Created record

```text
Record name: grafana
Record type: A
Alias: Yes
Alias target:
k8s-platformtools-9b008a2a90-238508984.us-east-1.elb.amazonaws.com
```

## Why Route 53 is needed

Without DNS, you can access only ALB DNS name.

With Route 53:

```text
grafana.learnwithmedevops.online
```

points to ALB.

## Access Grafana

```text
https://grafana.learnwithmedevops.online
```

---

# 18. Grafana and Prometheus Validation

## Open Grafana

```text
https://grafana.learnwithmedevops.online
```

Login:

```text
Username: admin
Password: Grafana@12345
```

## Validate Prometheus datasource

In Grafana:

```text
Connections → Data sources → Prometheus → Save & test
```

Expected:

```text
Successfully queried the Prometheus API
```

## Validate using Explore

Go to:

```text
Grafana → Explore → Prometheus → Code
```

Run:

```promql
kube_node_status_condition{condition="Ready",status="true"}
```

Expected:

```text
value = 1
```

Meaning:

```text
Kubernetes nodes are Ready.
Grafana can query Prometheus.
Prometheus is scraping kube-state-metrics.
```

---

# 19. Prometheus Metrics Explained

Prometheus metric format:

```text
metric_name{label1="value1", label2="value2"} value
```

Example:

```promql
kube_node_status_condition{condition="Ready",status="true",node="ip-10-0-1-163.ec2.internal"} 1
```

Meaning:

```text
Node ip-10-0-1-163 is Ready.
```

## Important value meaning

```text
1 = true / healthy / active
0 = false / unhealthy / inactive
```

## Useful queries

### Prometheus scrape health

```promql
up
```

Meaning:

```text
1 = target is scrapeable
0 = target is down
```

### Node Ready status

```promql
kube_node_status_condition{condition="Ready",status="true"}
```

### Node CPU usage

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Node memory usage

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Pod restarts

```promql
increase(kube_pod_container_status_restarts_total[10m])
```

---

# 20. Production Prometheus Alert Rules

## Why custom rules?

Default kube-prometheus-stack has many built-in alerts, but we created custom simple production rules for learning and validation.

## Create rules

```bash
cat > production-alert-rules.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: production-alert-rules
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: production-kubernetes-alerts
      rules:
        - alert: KubernetesNodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Kubernetes node not ready"
            description: "Node {{ $labels.node }} is NotReady for more than 5 minutes."

        - alert: KubernetesPodCrashLooping
          expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod crash looping"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarted more than 3 times in 10 minutes."

        - alert: NodeCPUHigh
          expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High node CPU usage"
            description: "Node {{ $labels.instance }} CPU usage is above 85% for 10 minutes."

        - alert: NodeMemoryHigh
          expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High node memory usage"
            description: "Node {{ $labels.instance }} memory usage is above 85% for 10 minutes."
EOF

kubectl apply -f production-alert-rules.yaml
```

## Validate

```bash
kubectl get prometheusrule -n monitoring | grep production
```

Expected:

```text
production-alert-rules
```

---

# 21. CloudWatch Observability Add-on

## Why CloudWatch if we already have Prometheus?

Prometheus is great for Kubernetes metrics.

CloudWatch is important for AWS-native visibility:

```text
Container Insights
CloudWatch Logs
ALB metrics
EC2 metrics
RDS metrics
AWS service metrics
```

## Install add-on

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --region $REGION \
  --addon-name amazon-cloudwatch-observability
```

## Validate add-on

```bash
aws eks describe-addon \
  --cluster-name $CLUSTER_NAME \
  --region $REGION \
  --addon-name amazon-cloudwatch-observability \
  --query "addon.status" \
  --output text
```

Expected:

```text
ACTIVE
```

## Validate pods

```bash
kubectl get pods -n amazon-cloudwatch
```

Expected:

```text
cloudwatch-agent running on 3 nodes
fluent-bit running on 3 nodes
controller-manager running
```

---

# 22. Troubleshooting CloudWatch IAM Permissions

## Problem

CloudWatch Agent logs showed:

```text
AccessDeniedException:
not authorized to perform logs:PutLogEvents
```

## Meaning

CloudWatch Agent was running, but it did not have IAM permission to push logs/events to CloudWatch Logs.

The denied role:

```text
day20-eks-node-20260426093455140300000001
```

This was the EKS worker node IAM role.

## Fix

Attach CloudWatch Agent policy:

```bash
aws iam attach-role-policy \
  --role-name day20-eks-node-20260426093455140300000001 \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

Temporary unblock policy:

```bash
aws iam attach-role-policy \
  --role-name day20-eks-node-20260426093455140300000001 \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentAdminPolicy
```

## Restart pods

```bash
kubectl rollout restart daemonset cloudwatch-agent -n amazon-cloudwatch
kubectl rollout restart daemonset fluent-bit -n amazon-cloudwatch
```

## Validate

```bash
kubectl logs -n amazon-cloudwatch ds/cloudwatch-agent --tail=50
```

Expected:

```text
Everything is ready. Begin running and processing data.
```

## Troubleshooting lesson

For AWS add-ons, always check:

```text
1. Pod is running
2. Pod logs have no AccessDenied
3. IAM role has required AWS permissions
4. CloudWatch log groups/metrics are actually created
```

---

# 23. CloudWatch Logs and Metrics Validation

## Check log groups

```bash
aws logs describe-log-groups \
  --region $REGION \
  --log-group-name-prefix /aws/containerinsights/$CLUSTER_NAME \
  --query "logGroups[*].logGroupName" \
  --output table
```

Expected:

```text
/aws/containerinsights/day20-eks/application
/aws/containerinsights/day20-eks/dataplane
/aws/containerinsights/day20-eks/host
/aws/containerinsights/day20-eks/performance
```

## Check application log streams

```bash
aws logs describe-log-streams \
  --region $REGION \
  --log-group-name /aws/containerinsights/$CLUSTER_NAME/application \
  --order-by LastEventTime \
  --descending \
  --max-items 5 \
  --query "logStreams[*].[logStreamName,lastEventTimestamp]" \
  --output table
```

## Check performance log streams

```bash
aws logs describe-log-streams \
  --region $REGION \
  --log-group-name /aws/containerinsights/$CLUSTER_NAME/performance \
  --order-by LastEventTime \
  --descending \
  --max-items 5 \
  --query "logStreams[*].[logStreamName,lastEventTimestamp]" \
  --output table
```

Expected node streams:

```text
ip-10-0-1-163.ec2.internal
ip-10-0-2-58.ec2.internal
ip-10-0-3-249.ec2.internal
```

## Check Container Insights metrics

```bash
aws cloudwatch list-metrics \
  --namespace ContainerInsights \
  --region $REGION \
  --metric-name node_cpu_utilization \
  --dimensions Name=ClusterName,Value=$CLUSTER_NAME \
  --query "Metrics[*].MetricName" \
  --output table
```

Expected:

```text
node_cpu_utilization
```

---

# 24. Grafana CloudWatch Datasource

## Why add CloudWatch datasource?

Prometheus shows Kubernetes metrics.

CloudWatch datasource lets Grafana show AWS metrics/logs:

```text
ALB metrics
EC2 metrics
CloudWatch Logs
Container Insights metrics
RDS metrics
NAT Gateway metrics
Billing metrics
```

## First issue

Grafana showed:

```text
missing default region
```

## Fix

In Grafana:

```text
Connections → Data sources → CloudWatch
Default Region: us-east-1
Authentication Provider: AWS SDK Default
Save & test
```

## Second issue

Grafana showed:

```text
AccessDenied:
cloudwatch:ListMetrics
```

## Meaning

Grafana was using the worker node IAM role, which did not have CloudWatch read permissions.

Production fix:

```text
Create separate IAM role for Grafana using IRSA.
```

---

# 25. Troubleshooting Grafana CloudWatch Datasource

## Create Grafana trust policy

```bash
export OIDC_ID=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)

export OIDC_PROVIDER="oidc.eks.${REGION}.amazonaws.com/id/${OIDC_ID}"

cat > grafana-cloudwatch-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com",
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:monitoring:monitoring-grafana"
        }
      }
    }
  ]
}
EOF
```

## Create IAM role

```bash
aws iam create-role \
  --role-name GrafanaCloudWatchDatasourceRole \
  --assume-role-policy-document file://grafana-cloudwatch-trust-policy.json
```

## Attach read-only permissions

```bash
aws iam attach-role-policy \
  --role-name GrafanaCloudWatchDatasourceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

aws iam attach-role-policy \
  --role-name GrafanaCloudWatchDatasourceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsReadOnlyAccess
```

## Add role annotation in Helm values

Inside `grafana:` block:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::590999018668:role/GrafanaCloudWatchDatasourceRole
```

Full location:

```yaml
grafana:
  enabled: true

  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::590999018668:role/GrafanaCloudWatchDatasourceRole
```

## Apply

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f kube-prometheus-values.yaml
```

## Restart Grafana

```bash
kubectl rollout restart deployment monitoring-grafana -n monitoring
kubectl rollout status deployment monitoring-grafana -n monitoring
```

## Validate annotation

```bash
kubectl get sa monitoring-grafana -n monitoring -o yaml | grep -A5 annotations
```

Expected:

```text
eks.amazonaws.com/role-arn: arn:aws:iam::590999018668:role/GrafanaCloudWatchDatasourceRole
```

## Retest in Grafana

```text
CloudWatch datasource → Save & test
```

Expected:

```text
CloudWatch metrics API: success
CloudWatch logs API: success
```

---

# 26. Alertmanager Slack Integration

## Why Slack integration?

Monitoring without alerts is incomplete.

Grafana dashboards are useful only when someone is watching. Alerts notify the team when something breaks.

## Create Slack webhook

In Slack:

```text
Apps → Incoming Webhooks → Add to Slack
Choose channel: #prod-alerts
Copy Webhook URL
```

## Create Alertmanager values

```bash
cat > alertmanager-slack-values.yaml << 'EOF'
alertmanager:
  config:
    global:
      resolve_timeout: 5m

    route:
      group_by:
        - alertname
        - namespace
        - severity
      group_wait: 10s
      group_interval: 2m
      repeat_interval: 1h
      receiver: slack-default
      routes:
        - matchers:
            - severity="critical"
          receiver: slack-critical

        - matchers:
            - severity="warning"
          receiver: slack-warning

    receivers:
      - name: slack-default
        slack_configs:
          - api_url: 'YOUR_SLACK_WEBHOOK_URL'
            channel: '#prod-alerts'
            send_resolved: true
            title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
            text: |
              *Cluster:* day20-eks
              *Severity:* {{ .CommonLabels.severity }}
              *Namespace:* {{ .CommonLabels.namespace }}
              *Status:* {{ .Status }}

              {{ range .Alerts }}
              *Summary:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              {{ end }}

      - name: slack-critical
        slack_configs:
          - api_url: 'YOUR_SLACK_WEBHOOK_URL'
            channel: '#prod-alerts'
            send_resolved: true
            title: ':rotating_light: CRITICAL - {{ .CommonLabels.alertname }}'
            text: |
              *Cluster:* day20-eks
              *Severity:* critical
              *Namespace:* {{ .CommonLabels.namespace }}
              *Status:* {{ .Status }}

              {{ range .Alerts }}
              *Summary:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              {{ end }}

      - name: slack-warning
        slack_configs:
          - api_url: 'YOUR_SLACK_WEBHOOK_URL'
            channel: '#prod-alerts'
            send_resolved: true
            title: ':warning: WARNING - {{ .CommonLabels.alertname }}'
            text: |
              *Cluster:* day20-eks
              *Severity:* warning
              *Namespace:* {{ .CommonLabels.namespace }}
              *Status:* {{ .Status }}

              {{ range .Alerts }}
              *Summary:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              {{ end }}
EOF
```

## Apply

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f kube-prometheus-values.yaml \
  -f alertmanager-slack-values.yaml
```

## Restart Alertmanager

```bash
kubectl delete pod -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0
kubectl delete pod -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-1
```

## Validate pods

```bash
kubectl get pods -n monitoring | grep alertmanager
```

---

# 27. Slack Alert Test

## Create test alert

```bash
cat > test-alert-rule.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: test-slack-alert
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: test.rules
      rules:
        - alert: TestSlackAlert
          expr: vector(1)
          for: 1m
          labels:
            severity: critical
            namespace: monitoring
          annotations:
            summary: "Test Slack alert from Alertmanager"
            description: "This is a test alert to verify Slack notification routing from Alertmanager."
EOF

kubectl apply -f test-alert-rule.yaml
```

## Verify Prometheus has the alert

```bash
kubectl run prom-check \
  -n monitoring \
  --rm -i --restart=Never \
  --image=curlimages/curl \
  -- sh -c 'curl -s http://monitoring-kube-prometheus-prometheus:9090/api/v1/alerts | grep -o "TestSlackAlert" | head'
```

Expected:

```text
TestSlackAlert
```

## Verify Alertmanager received the alert

```bash
kubectl run am-check \
  -n monitoring \
  --rm -i --restart=Never \
  --image=curlimages/curl \
  -- sh -c 'curl -s http://alertmanager-operated:9093/api/v2/alerts | grep -o "TestSlackAlert" | head'
```

Expected:

```text
TestSlackAlert
```

---

# 28. Troubleshooting Slack Alerts

## Problem

Alert existed in Prometheus.

Alert existed in Alertmanager.

But Slack did not receive it.

## What this means

The issue was not Prometheus.

The issue was not Alertmanager receiving alerts.

The issue was Alertmanager sending to Slack.

## Check Alertmanager logs

```bash
kubectl logs -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0 \
  -c alertmanager --since=5m | grep -i "slack\|notify\|error\|webhook"
```

Error:

```text
channel "#prod-alerts": unexpected status code 404: channel_not_found
```

## Meaning

Slack rejected the message because:

```text
#prod-alerts did not exist
or webhook was not connected to that channel
```

## Fix

In Slack:

```text
Create or rename channel to #prod-alerts
Add Incoming Webhook integration to #prod-alerts
```

Then restart Alertmanager:

```bash
kubectl delete pod -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0
kubectl delete pod -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-1
```

## Final result

Slack alert was received successfully.

## Troubleshooting lesson

For alert delivery, follow this order:

```text
1. Is Prometheus alert firing?
2. Did Alertmanager receive it?
3. Is Alertmanager config correct?
4. Is webhook URL correct?
5. Is Slack channel correct?
6. Is outbound network working?
```

---

# 29. Cleanup Test Alert

After Slack alert was received:

```bash
kubectl delete prometheusrule test-slack-alert -n monitoring
```

Validate:

```bash
kubectl get prometheusrule -n monitoring | grep test
```

Expected:

```text
No output
```

---

# 30. Final Status

```text
Prometheus:
  DONE - kube-prometheus-stack installed
  DONE - Prometheus running with 2 replicas
  DONE - kube-state-metrics running
  DONE - node-exporter running
  DONE - ServiceMonitors created
  DONE - PrometheusRule alerts created
  DONE - Prometheus scraping metrics

Grafana:
  DONE - Grafana installed
  DONE - Persistent storage enabled using gp3
  DONE - HTTPS access through ALB
  DONE - ACM certificate attached
  DONE - Route 53 configured
  DONE - Prometheus datasource working
  DONE - CloudWatch datasource working

Alertmanager:
  DONE - Alertmanager installed with 2 replicas
  DONE - Slack receiver configured
  DONE - Test alert fired
  DONE - Alert reached Prometheus
  DONE - Alert reached Alertmanager
  DONE - Alert received in Slack

CloudWatch:
  DONE - amazon-cloudwatch-observability add-on installed
  DONE - cloudwatch-agent running
  DONE - fluent-bit running
  DONE - CloudWatch log groups created
  DONE - Application log streams created
  DONE - Performance log streams created
  DONE - ContainerInsights metrics visible
  DONE - CloudWatch datasource connected to Grafana

AWS/EKS Dependencies:
  DONE - Helm installed
  DONE - Namespaces created
  DONE - ACM wildcard certificate created and issued
  DONE - AWS Load Balancer Controller installed
  DONE - AWS Load Balancer Controller IRSA fixed
  DONE - ALB created
  DONE - EBS CSI Driver installed
  DONE - gp3 StorageClass created
  DONE - Grafana CloudWatch IRSA role created
```

---

# 31. Pending After Application Deployment

No application has been deployed yet. So app-specific monitoring is pending.

After application deployment, add:

```text
Application ServiceMonitor
Application /actuator/prometheus scraping
Spring Boot JVM dashboard
Application 4xx/5xx alerts
Application latency alerts
Application logs
Application ALB request count/errors
Application business metrics
Application pod-specific dashboards
```

Application monitoring steps:

```text
1. Add Spring Boot actuator dependency
2. Enable /actuator/prometheus
3. Create ServiceMonitor for app service
4. Import JVM dashboard
5. Add HTTP latency and 5xx alerts
6. Verify app logs in CloudWatch
7. Add app dashboards in Grafana
```

---

# 32. Production Hardening

Before calling this enterprise-ready:

```text
1. Rotate Slack webhook because it was exposed earlier.
2. Restrict Grafana ALB access using inbound-cidrs or VPN.
3. Disable or tune EKS-incompatible alerts:
   KubeSchedulerDown
   KubeControllerManagerDown
4. Replace CloudWatchAgentAdminPolicy with least-privilege custom policy.
5. Store Slack webhook in Kubernetes Secret or External Secrets, not plain values file.
6. Move all YAML files into Git repo.
7. Manage monitoring stack using Argo CD/GitOps.
8. Add dashboard backup/export process.
9. Add CloudWatch Logs retention policies.
10. Configure Alertmanager inhibition and silencing rules.
```

---

# 33. How to Think Like a Troubleshooter

You copy-pasted commands during the setup. That is fine for first-time learning.

Now understand the troubleshooting mindset.

## Rule 1: Always identify which layer is failing

Example:

```text
Grafana not opening
```

Possible layers:

```text
DNS
ALB
Ingress
Service
Pod
Application
Security Group
Certificate
```

So check in order:

```bash
kubectl get ingress -n monitoring
kubectl describe ingress monitoring-grafana -n monitoring
kubectl get svc -n monitoring
kubectl get pods -n monitoring
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

## Rule 2: Read exact error messages

Example error:

```text
serviceaccount not found
```

Fix:

```text
Create ServiceAccount
```

Example error:

```text
NoSuchEntity IAM role cannot be found
```

Fix:

```text
Create IAM role
```

Example error:

```text
AccessDenied logs:PutLogEvents
```

Fix:

```text
Attach CloudWatch Logs permission
```

Example error:

```text
channel_not_found
```

Fix:

```text
Create Slack channel or correct channel name
```

## Rule 3: Separate symptom and root cause

Symptom:

```text
Ingress ADDRESS empty
```

Root cause:

```text
AWS Load Balancer Controller could not assume IAM role
```

Symptom:

```text
CloudWatch pods Running but no logs
```

Root cause:

```text
IAM role missing logs:PutLogEvents
```

Symptom:

```text
Alertmanager has alert but Slack no message
```

Root cause:

```text
Slack channel not found
```

## Rule 4: Validate after every fix

After fixing ALB Controller:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get ingress -n monitoring
```

After fixing CloudWatch:

```bash
aws logs describe-log-groups --log-group-name-prefix /aws/containerinsights/$CLUSTER_NAME
```

After fixing Slack:

```bash
kubectl run am-check ...
```

## Rule 5: Know the flow

### Grafana access flow

```text
Browser
  ↓
Route 53
  ↓
ALB
  ↓
Ingress
  ↓
Kubernetes Service
  ↓
Grafana Pod
```

### Prometheus metric flow

```text
node-exporter / kube-state-metrics
  ↓
Prometheus scrape
  ↓
Prometheus storage
  ↓
Grafana datasource
  ↓
Dashboard / Explore
```

### Alert flow

```text
PrometheusRule
  ↓
Prometheus evaluates rule
  ↓
Alert fires
  ↓
Alertmanager receives alert
  ↓
Alertmanager route chooses receiver
  ↓
Slack webhook sends message
```

### CloudWatch logs flow

```text
Container logs
  ↓
Fluent Bit
  ↓
CloudWatch Agent / CloudWatch output
  ↓
CloudWatch Logs
  ↓
Grafana CloudWatch datasource
```

---

# 34. Interview Explanation

You can explain the project like this:

```text
I implemented a production-style monitoring stack on Amazon EKS using kube-prometheus-stack, Grafana, Alertmanager, AWS CloudWatch Observability add-on, Fluent Bit, AWS Load Balancer Controller, ACM, Route 53, EBS CSI, gp3 volumes, IRSA, and Slack alerting.

Prometheus collects Kubernetes and node-level metrics using kube-state-metrics and node-exporter. Grafana is exposed securely over HTTPS using AWS ALB, ACM certificate, and Route 53. Alertmanager sends critical and warning alerts to Slack. CloudWatch Observability add-on collects Container Insights metrics and container logs using CloudWatch Agent and Fluent Bit.

During the setup, I troubleshot real production issues including AWS Load Balancer Controller failing due to missing ServiceAccount and missing IAM role, IRSA trust policy issues, CloudWatch PutLogEvents AccessDenied error, Grafana CloudWatch datasource ListMetrics AccessDenied error, and Slack channel_not_found alert delivery failure. I validated each layer using kubectl, AWS CLI, Grafana Explore, Prometheus alerts API, Alertmanager API, and CloudWatch log/metric queries.
```

---

# Final Summary

```text
This is a real-time production-style EKS cluster monitoring setup.

Cluster/platform monitoring is complete:
Prometheus
Grafana
Alertmanager
Slack
CloudWatch Logs
CloudWatch Metrics
Container Insights
ALB HTTPS access
IRSA-based AWS access

Application-specific monitoring will be added after application deployment.
```
