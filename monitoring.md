# Production-Grade EKS Monitoring Stack

This repository documents a production-style monitoring and observability setup for an Amazon EKS cluster using Prometheus, Grafana, Alertmanager, Slack, CloudWatch Container Insights, Fluent Bit, AWS Load Balancer Controller, ACM, Route 53, EBS CSI Driver, and gp3 persistent storage.

> Current scope: **cluster/platform monitoring**.  
> Application-specific monitoring will be added after application deployment.

---

## Table of Contents

1. [Architecture](#architecture)
2. [What Was Implemented](#what-was-implemented)
3. [Prerequisites](#prerequisites)
4. [Namespaces](#namespaces)
5. [Helm Installation](#helm-installation)
6. [ACM Wildcard Certificate](#acm-wildcard-certificate)
7. [EKS Cluster Variables](#eks-cluster-variables)
8. [IAM OIDC Provider](#iam-oidc-provider)
9. [AWS Load Balancer Controller](#aws-load-balancer-controller)
10. [AWS Load Balancer Controller Troubleshooting](#aws-load-balancer-controller-troubleshooting)
11. [EBS CSI Driver](#ebs-csi-driver)
12. [gp3 StorageClass](#gp3-storageclass)
13. [Grafana Admin Secret](#grafana-admin-secret)
14. [kube-prometheus-stack Values](#kube-prometheus-stack-values)
15. [Install kube-prometheus-stack](#install-kube-prometheus-stack)
16. [Grafana Ingress and ALB](#grafana-ingress-and-alb)
17. [Route 53 DNS Record](#route-53-dns-record)
18. [Prometheus Validation in Grafana](#prometheus-validation-in-grafana)
19. [Production Prometheus Alert Rules](#production-prometheus-alert-rules)
20. [CloudWatch Observability Add-on](#cloudwatch-observability-add-on)
21. [CloudWatch IAM Fix](#cloudwatch-iam-fix)
22. [CloudWatch Log Group Validation](#cloudwatch-log-group-validation)
23. [CloudWatch Metrics Validation](#cloudwatch-metrics-validation)
24. [Grafana CloudWatch Datasource](#grafana-cloudwatch-datasource)
25. [Grafana CloudWatch IRSA Role](#grafana-cloudwatch-irsa-role)
26. [Alertmanager Slack Integration](#alertmanager-slack-integration)
27. [Test Slack Alert](#test-slack-alert)
28. [Slack Troubleshooting](#slack-troubleshooting)
29. [Cleanup Test Alert](#cleanup-test-alert)
30. [Final Completed Status](#final-completed-status)
31. [Pending Application Monitoring](#pending-application-monitoring)
32. [Production Hardening Recommendations](#production-hardening-recommendations)

---

## Architecture

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
  |     |-- Creates ALB for Grafana
  |
  |-- EBS CSI Driver
  |     |-- Creates gp3 volumes for Prometheus/Grafana/Alertmanager PVCs
  |
  |-- CloudWatch Observability Add-on
  |     |-- CloudWatch Agent
  |     |-- Fluent Bit
  |     |-- Container Insights metrics/logs
  |
  |-- Alertmanager
        |-- Sends alerts to Slack
```

Final Grafana access pattern:

```text
https://grafana.<your-domain>
```

Example:

```text
https://grafana.learnwithmedevops.online
```

---

## What Was Implemented

```text
Prometheus
Grafana
AWS Load Balancer Controller
ACM certificate
Route 53
EBS CSI Driver
gp3 StorageClass
CloudWatch Observability add-on
CloudWatch logs/metrics
Grafana CloudWatch datasource
Alertmanager
Slack alerts
Troubleshooting fixes
```

---

## Prerequisites

Required tools:

```text
awscli
kubectl
eksctl
helm
```

Required AWS resources:

```text
EKS cluster
Route 53 hosted zone
ACM certificate
Worker node IAM role
OIDC provider associated with EKS
```

Example cluster:

```text
Cluster name: day20-eks
Region: us-east-1
```

---

## Namespaces

Create namespaces:

```bash
kubectl create namespace monitoring
kubectl create namespace logging
```

Explanation:

```text
monitoring namespace:
  Prometheus
  Grafana
  Alertmanager
  kube-state-metrics
  node-exporter

logging namespace:
  reserved for ELK/logging later
```

---

## Helm Installation

If Helm is missing:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

Add Prometheus community Helm repo:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Add AWS EKS Helm repo:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

---

## ACM Wildcard Certificate

Create one wildcard certificate for all first-level subdomains:

```bash
aws acm request-certificate \
  --domain-name "*.learnwithmedevops.online" \
  --subject-alternative-names "learnwithmedevops.online" \
  --validation-method DNS \
  --region us-east-1
```

Example certificate ARN:

```text
arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERTIFICATE_ID>
```

Get DNS validation record:

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERTIFICATE_ID> \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions[*].ResourceRecord"
```

Example DNS validation CNAME:

```text
Name:
_<TOKEN>.learnwithmedevops.online.

Type:
CNAME

Value:
_<TOKEN>.acm-validations.aws.
```

Create this CNAME record in Route 53.

Check certificate status:

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERTIFICATE_ID> \
  --region us-east-1 \
  --query "Certificate.Status" \
  --output text
```

Expected:

```text
ISSUED
```

This wildcard certificate can be used for:

```text
grafana.learnwithmedevops.online
kibana.learnwithmedevops.online
argocd.learnwithmedevops.online
jenkins.learnwithmedevops.online
app.learnwithmedevops.online
```

---

## EKS Cluster Variables

Set variables:

```bash
export CLUSTER_NAME=day20-eks
export REGION=us-east-1
export ACCOUNT_ID=<ACCOUNT_ID>
```

Verify nodes:

```bash
kubectl get nodes
```

Example nodes:

```text
ip-10-0-1-163.ec2.internal
ip-10-0-2-58.ec2.internal
ip-10-0-3-249.ec2.internal
```

---

## IAM OIDC Provider

Associate IAM OIDC provider:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --region $REGION \
  --approve
```

Expected if already present:

```text
IAM Open ID Connect provider is already associated with cluster
```

Why this is required:

```text
OIDC is required for IRSA.
IRSA allows Kubernetes ServiceAccounts to assume IAM roles.
Used for:
  AWS Load Balancer Controller
  Grafana CloudWatch datasource
```

---

## AWS Load Balancer Controller

### Download IAM policy

```bash
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

### Create IAM policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Policy ARN:

```text
arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

### Try creating IAM ServiceAccount

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

If it skips because of an existing or missing service account issue, use the troubleshooting section below.

### Install controller

Get VPC ID:

```bash
export VPC_ID=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

echo $VPC_ID
```

Install controller:

```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$REGION \
  --set vpcId=$VPC_ID
```

Verify:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get ingressclass
```

Expected:

```text
aws-load-balancer-controller   2/2
alb
```

---

## AWS Load Balancer Controller Troubleshooting

### Issue 1: Controller stuck at 0/2

Check ReplicaSet:

```bash
kubectl get rs -n kube-system | grep aws-load-balancer
kubectl describe rs -n kube-system <replicaset-name>
```

Example error:

```text
serviceaccount "aws-load-balancer-controller" not found
```

Fix:

```bash
kubectl create serviceaccount aws-load-balancer-controller -n kube-system
```

Annotate ServiceAccount:

```bash
kubectl annotate serviceaccount aws-load-balancer-controller \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKSLoadBalancerControllerRole \
  --overwrite
```

### Issue 2: IAM role does not exist

Error:

```text
NoSuchEntity: The role with name AmazonEKSLoadBalancerControllerRole cannot be found
```

Generate OIDC variables:

```bash
export OIDC_ID=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)

export OIDC_PROVIDER="oidc.eks.${REGION}.amazonaws.com/id/${OIDC_ID}"
```

Create trust policy:

```bash
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

Create IAM role:

```bash
aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document file://alb-controller-trust-policy.json
```

Attach policy:

```bash
aws iam attach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

Restart controller:

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
kubectl rollout status deployment aws-load-balancer-controller -n kube-system
```

Expected:

```text
aws-load-balancer-controller 2/2 Running
```

---

## EBS CSI Driver

Create IAM role for EBS CSI:

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

Get role ARN:

```bash
export EBS_CSI_ROLE_ARN=$(aws iam get-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --query "Role.Arn" \
  --output text)

echo $EBS_CSI_ROLE_ARN
```

Create EKS add-on:

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --region $REGION \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn $EBS_CSI_ROLE_ARN
```

Verify:

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

Expected:

```text
ebs-csi-controller running
ebs-csi-node running on all nodes
```

---

## gp3 StorageClass

Create gp3 StorageClass:

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

Verify:

```bash
kubectl get storageclass
```

Expected:

```text
gp2
gp3
```

Explanation:

```text
Prometheus, Grafana, and Alertmanager need persistent storage.
gp3 volumes are provisioned through the EBS CSI Driver.
```

---

## Grafana Admin Secret

Create Grafana admin secret:

```bash
kubectl create secret generic grafana-admin-secret \
  -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='CHANGE_ME_STRONG_PASSWORD'
```

Explanation:

```text
Do not hardcode Grafana passwords directly in Helm values.
Use Kubernetes Secret or External Secrets.
```

---

## kube-prometheus-stack Values

Create `kube-prometheus-values.yaml`:

```yaml
grafana:
  enabled: true

  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/GrafanaCloudWatchDatasourceRole

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
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERTIFICATE_ID>
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
```

---

## Install kube-prometheus-stack

Install monitoring stack:

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f kube-prometheus-values.yaml
```

Verify pods:

```bash
kubectl get pods -n monitoring
```

Expected:

```text
alertmanager-monitoring-kube-prometheus-alertmanager-0
alertmanager-monitoring-kube-prometheus-alertmanager-1
monitoring-grafana
monitoring-kube-prometheus-operator
monitoring-kube-state-metrics
monitoring-prometheus-node-exporter
prometheus-monitoring-kube-prometheus-prometheus-0
prometheus-monitoring-kube-prometheus-prometheus-1
```

Verify services:

```bash
kubectl get svc -n monitoring
```

Verify PVCs:

```bash
kubectl get pvc -n monitoring
```

Expected:

```text
monitoring-grafana Bound 10Gi gp3
```

---

## Grafana Ingress and ALB

Check ingress:

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

Describe ingress:

```bash
kubectl describe ingress monitoring-grafana -n monitoring
```

Expected event:

```text
SuccessfullyReconciled
```

Expected ALB controller actions:

```text
created securityGroup
created targetGroup
created loadBalancer
created listener 80
created listener 443
created listener rule
registered targets
successfully deployed model
```

---

## Route 53 DNS Record

Create Route 53 record:

```text
Record name: grafana
Record type: A
Alias: Yes
Alias target:
k8s-platformtools-xxxx.us-east-1.elb.amazonaws.com
```

Access Grafana:

```text
https://grafana.learnwithmedevops.online
```

Login:

```text
Username: admin
Password: <GRAFANA_ADMIN_PASSWORD>
```

---

## Prometheus Validation in Grafana

Open:

```text
Grafana → Explore → Prometheus
```

Run:

```promql
kube_node_status_condition{condition="Ready",status="true"}
```

Expected:

```text
Value = 1
```

Meaning:

```text
All EKS worker nodes are Ready.
Prometheus is scraping kube-state-metrics.
Grafana can query Prometheus.
```

Other useful queries:

```promql
up
```

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

```promql
increase(kube_pod_container_status_restarts_total[10m])
```

---

## Production Prometheus Alert Rules

Create `production-alert-rules.yaml`:

```yaml
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
```

Apply:

```bash
kubectl apply -f production-alert-rules.yaml
```

Verify:

```bash
kubectl get prometheusrule -n monitoring | grep production
```

Expected:

```text
production-alert-rules
```

---

## CloudWatch Observability Add-on

Install add-on:

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --region $REGION \
  --addon-name amazon-cloudwatch-observability
```

Verify add-on status:

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

Check pods:

```bash
kubectl get pods -n amazon-cloudwatch
```

Expected:

```text
amazon-cloudwatch-observability-controller-manager Running
cloudwatch-agent Running on all nodes
fluent-bit Running on all nodes
```

---

## CloudWatch IAM Fix

Initial issue:

```text
AccessDeniedException:
not authorized to perform logs:PutLogEvents
```

Affected role:

```text
<WORKER_NODE_ROLE_NAME>
```

Attach CloudWatch Agent policy:

```bash
aws iam attach-role-policy \
  --role-name <WORKER_NODE_ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

Temporary unblock policy used during troubleshooting:

```bash
aws iam attach-role-policy \
  --role-name <WORKER_NODE_ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentAdminPolicy
```

> Production recommendation: replace `CloudWatchAgentAdminPolicy` with a least-privilege policy later.

Restart components:

```bash
kubectl rollout restart daemonset cloudwatch-agent -n amazon-cloudwatch
kubectl rollout restart daemonset fluent-bit -n amazon-cloudwatch
```

Check logs:

```bash
kubectl logs -n amazon-cloudwatch ds/cloudwatch-agent --tail=50
kubectl logs -n amazon-cloudwatch ds/fluent-bit --tail=50
```

Expected CloudWatch Agent message:

```text
Everything is ready. Begin running and processing data.
```

---

## CloudWatch Log Group Validation

Check log groups:

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

Check application log streams:

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

Check performance log streams:

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
ip-10-0-3-249.ec2.internal
ip-10-0-2-58.ec2.internal
ip-10-0-1-163.ec2.internal
```

---

## CloudWatch Metrics Validation

Check Container Insights metrics:

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

## Grafana CloudWatch Datasource

In Grafana UI:

```text
Grafana → Connections → Data sources → CloudWatch
```

Set:

```text
Default Region: us-east-1
Authentication Provider: AWS SDK Default
```

Save and test.

Initial issue:

```text
missing default region
```

Fix:

```text
Set Default Region = us-east-1
```

Second issue:

```text
AccessDenied:
cloudwatch:ListMetrics
```

Cause:

```text
Grafana was using EKS node IAM role.
```

Production fix:

```text
Create dedicated Grafana IRSA role for CloudWatch datasource.
```

---

## Grafana CloudWatch IRSA Role

Create trust policy:

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

Create IAM role:

```bash
aws iam create-role \
  --role-name GrafanaCloudWatchDatasourceRole \
  --assume-role-policy-document file://grafana-cloudwatch-trust-policy.json
```

Attach read-only policies:

```bash
aws iam attach-role-policy \
  --role-name GrafanaCloudWatchDatasourceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

aws iam attach-role-policy \
  --role-name GrafanaCloudWatchDatasourceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsReadOnlyAccess
```

Add annotation under `grafana:` in `kube-prometheus-values.yaml`:

```yaml
grafana:
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/GrafanaCloudWatchDatasourceRole
```

Apply Helm:

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f kube-prometheus-values.yaml
```

Restart Grafana:

```bash
kubectl rollout restart deployment monitoring-grafana -n monitoring
kubectl rollout status deployment monitoring-grafana -n monitoring
```

Verify annotation:

```bash
kubectl get sa monitoring-grafana -n monitoring -o yaml | grep -A5 annotations
```

Expected:

```text
eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/GrafanaCloudWatchDatasourceRole
```

Retest CloudWatch datasource:

```text
Grafana → CloudWatch datasource → Save & test
```

Expected:

```text
CloudWatch metrics API: success
CloudWatch logs API: success
```

---

## Alertmanager Slack Integration

Create Slack channel:

```text
#prod-alerts
```

Create Slack incoming webhook for `#prod-alerts`.

Create `alertmanager-slack-values.yaml`:

```yaml
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
```

Apply:

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f kube-prometheus-values.yaml \
  -f alertmanager-slack-values.yaml
```

Restart Alertmanager pods:

```bash
kubectl delete pod -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0
kubectl delete pod -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-1
```

Verify:

```bash
kubectl get pods -n monitoring | grep alertmanager
```

---

## Test Slack Alert

Create `test-alert-rule.yaml`:

```yaml
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
```

Apply:

```bash
kubectl apply -f test-alert-rule.yaml
```

Verify Prometheus has the alert:

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

Verify Alertmanager received the alert:

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

## Slack Troubleshooting

Issue:

```text
channel "#prod-alerts": unexpected status code 404: channel_not_found
```

Cause:

```text
Slack channel #prod-alerts did not exist or webhook was not connected to that channel.
```

Fix:

```text
Create/rename Slack channel to #prod-alerts
Add incoming webhook integration to #prod-alerts
Restart Alertmanager
```

Check logs:

```bash
kubectl logs -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0 \
  -c alertmanager --since=5m | grep -i "slack\|notify\|error\|webhook"
```

Expected:

```text
No channel_not_found error
```

Final result:

```text
Slack alert received successfully
```

---

## Cleanup Test Alert

After receiving Slack alert:

```bash
kubectl delete prometheusrule test-slack-alert -n monitoring
```

Verify:

```bash
kubectl get prometheusrule -n monitoring | grep test
```

Expected:

```text
No output
```

---

## Final Completed Status

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

## Pending Application Monitoring

No application has been deployed yet, so these are pending:

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

After app deployment:

```text
1. Add actuator dependency in Java app
2. Expose /actuator/prometheus
3. Create ServiceMonitor for app service
4. Add JVM dashboard
5. Add HTTP latency and 5xx alerts
6. Verify app logs in CloudWatch
7. Add app dashboards in Grafana
```

---

## Production Hardening Recommendations

Before calling it enterprise-ready:

```text
1. Rotate Slack webhook because it was exposed earlier.
2. Restrict Grafana ALB access using inbound-cidrs or VPN.
3. Disable/tune EKS-incompatible default alerts:
   KubeSchedulerDown
   KubeControllerManagerDown
4. Replace CloudWatchAgentAdminPolicy with least-privilege custom policy.
5. Store Slack webhook in Kubernetes Secret or External Secrets, not plain values file.
6. Move all YAML files into Git repo.
7. Use Argo CD/GitOps to manage monitoring stack.
8. Add dashboard backup/export process.
9. Add retention policies for CloudWatch Logs.
10. Configure Alertmanager inhibition/silencing rules.
```

---

## Final Production Summary

```text
This setup completes an EKS cluster-level production monitoring stack.

Prometheus collects Kubernetes and node metrics.
Grafana provides HTTPS dashboard access through ALB and Route 53.
CloudWatch collects AWS/EKS logs and Container Insights metrics.
Alertmanager sends production alerts to Slack.
All core platform monitoring components are running and validated.

Application-specific monitoring will be added after application deployment.
```

---

## Important Security Notes

Do not commit real secrets to GitHub.

Replace these before committing:

```text
AWS account IDs
Certificate ARNs
Slack webhook URLs
Grafana admin passwords
Private domain names if required
```

Use placeholders or external secret management:

```text
AWS Secrets Manager
External Secrets Operator
Sealed Secrets
SOPS
```
