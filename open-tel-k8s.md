# OpenTelemetry Demo on AWS EKS — Complete Setup

## Architecture we are going to build

```text
Your Laptop
    |
    |
AWS Account
    |
    +--- EC2 Management Server
    |       |
    |       +--- AWS CLI
    |       +--- kubectl
    |       +--- eksctl
    |       +--- Helm
    |
    |
    +--- Amazon EKS
            |
            +--- Managed Node Group
                    |
                    +--- OpenTelemetry Demo
                            |
                            +--- Frontend
                            +--- Product Catalog
                            +--- Cart
                            +--- Checkout
                            +--- Payment
                            +--- Shipping
                            +--- Recommendation
                            +--- Kafka
                            +--- Valkey
                            +--- OpenTelemetry Collector
                            +--- Prometheus
                            +--- Grafana
                            +--- Jaeger
                            +--- Load Generator
                            +--- Flagd
                            +--- Other services
```

The OpenTelemetry Demo is a microservice-based distributed application containing services written in multiple languages and generates telemetry such as traces, metrics, and logs. ([OpenTelemetry][3])

---

# 1. Which AWS instances should we use?

The OpenTelemetry docs require at least **6 GB free RAM** for the application itself. ([OpenTelemetry][1])

Therefore, do **not** try this on:

```text
t2.micro
t3.micro
t3.small
```

For a teaching/demo environment, I recommend:

```text
EKS Nodes
-------------------------------
Instance type : t3.large
vCPU          : 2
RAM           : 8 GB
Nodes         : 2
Minimum       : 2
Maximum       : 3
Disk          : 30 GB gp3
```

So approximately:

```text
2 × t3.large

Total:
4 vCPU
16 GB RAM
```

This provides enough room for the OpenTelemetry demo pods plus Kubernetes system components.

For a smoother lab, you could alternatively use:

```text
2 × t3.xlarge

Total:
8 vCPU
32 GB RAM
```

But `2 x t3.large` is normally a reasonable starting point for the demo.

---

# 2. Create an EC2 management server

You can run the commands from your laptop, but for your classroom/lab setup I recommend a dedicated EC2 administration machine.

Go to:

```text
AWS Console
→ EC2
→ Launch instance
```

Use:

```text
Name             : eks-management
AMI              : Amazon Linux 2023
Instance Type    : t3.small
Disk             : 20 GB gp3
Region           : ap-south-1
```

Security group:

```text
22 / SSH / My IP
```

Launch the instance.

SSH into it:

```bash
ssh -i your-key.pem ec2-user@PUBLIC-IP
```

---

# 3. Configure AWS permissions

For a simple training lab, the IAM user/role running `eksctl` needs sufficient permissions to create:

```text
EKS
EC2
VPC
IAM
CloudFormation
Auto Scaling
ELB
```

AWS notes that `eksctl` creates several AWS resources automatically, including cluster and node resources. ([AWS Documentation][4])

If this is purely your personal lab, using an administrative IAM role temporarily is the simplest approach.

Verify AWS credentials:

```bash
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/..."
}
```

---

# 4. Install AWS CLI

Check first:

```bash
aws --version
```

If AWS CLI is not installed:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

Install unzip:

```bash
sudo dnf install unzip -y
```

Extract:

```bash
unzip awscliv2.zip
```

Install:

```bash
sudo ./aws/install
```

Verify:

```bash
aws --version
```

Configure if you're using IAM access keys:

```bash
aws configure
```

Enter:

```text
AWS Access Key ID:
AWS Secret Access Key:
Default region name: ap-south-1
Default output format: json
```

Prefer an EC2 IAM role instead of long-lived access keys when possible.

---

# 5. Install kubectl

AWS currently provides Kubernetes 1.36 kubectl binaries, including `1.36.2`. ([AWS Documentation][5])

Install:

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.36.2/2026-07-05/bin/linux/amd64/kubectl
```

Give execute permission:

```bash
chmod +x kubectl
```

Move it:

```bash
sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

You should see something similar to:

```text
Client Version: v1.36.2
```

AWS specifically recommends placing `kubectl` somewhere in your `PATH`. ([AWS Documentation][5])

---

# 6. Install eksctl

Install prerequisites:

```bash
sudo dnf install tar gzip -y
```

Download eksctl:

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
```

Extract:

```bash
tar -xzf eksctl_${PLATFORM}.tar.gz
```

Move:

```bash
sudo mv eksctl /usr/local/bin/
```

Verify:

```bash
eksctl version
```

AWS describes `eksctl` as the CLI used to create and manage EKS clusters. ([AWS Documentation][6])

---

# 7. Install Helm

OpenTelemetry requires **Helm 3.14+** for this deployment method. ([OpenTelemetry][1])

Install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Check:

```bash
helm version
```

You should see Helm 3.x.

---

# 8. Create the EKS cluster

You can create it directly using `eksctl`.

For our demo:

```bash
eksctl create cluster \
--name otel-demo-cluster \
--region ap-south-1 \
--version 1.36 \
--nodegroup-name otel-nodes \
--node-type t3.large \
--nodes 2 \
--nodes-min 2 \
--nodes-max 3 \
--node-volume-size 30 \
--managed
```

This will create:

```text
VPC
Subnets
Internet Gateway
Route Tables
EKS Control Plane
IAM Roles
Managed Node Group
EC2 worker nodes
Security Groups
CloudFormation stacks
```

AWS recommends `eksctl` as one of the fastest ways to get started with EKS. ([AWS Documentation][7])

---

# 9. Verify EKS cluster

Check cluster:

```bash
eksctl get cluster
```

Expected:

```text
NAME                 REGION
otel-demo-cluster    ap-south-1
```

Check nodes:

```bash
kubectl get nodes
```

You should get something like:

```text
NAME                                          STATUS   ROLES    AGE
ip-192-168-xx-xx.ap-south-1.compute.internal Ready    <none>   5m
ip-192-168-xx-xx.ap-south-1.compute.internal Ready    <none>   5m
```

Check Kubernetes version:

```bash
kubectl get nodes -o wide
```

Also:

```bash
kubectl version
```

---

# 10. Check available resources

This step is important.

Run:

```bash
kubectl top nodes
```

If metrics-server isn't present, this command may initially fail; that's not required for the OpenTelemetry demo.

Check allocatable resources instead:

```bash
kubectl describe nodes | grep -A 6 "Allocatable"
```

We need enough free memory because OpenTelemetry officially specifies approximately **6 GB free RAM** for the demo. ([OpenTelemetry][1])

---

# 11. Create OpenTelemetry namespace

I prefer keeping the demo isolated.

```bash
kubectl create namespace otel-demo
```

Check:

```bash
kubectl get namespaces
```

You should see:

```text
default
kube-node-lease
kube-public
kube-system
otel-demo
```

---

# 12. Add OpenTelemetry Helm repository

The official documentation gives this repository command: ([OpenTelemetry][1])

```bash
helm repo add open-telemetry \
https://open-telemetry.github.io/opentelemetry-helm-charts
```

Update Helm repositories:

```bash
helm repo update
```

Verify:

```bash
helm search repo open-telemetry
```

You should see charts such as:

```text
open-telemetry/opentelemetry-demo
open-telemetry/opentelemetry-collector
open-telemetry/opentelemetry-operator
```

---

# 13. Check OpenTelemetry Demo chart version

Run:

```bash
helm search repo open-telemetry/opentelemetry-demo
```

OpenTelemetry notes that chart version **0.11.0 or greater** is required for all the usage methods described in its current Kubernetes documentation. ([OpenTelemetry][1])

---

# 14. Install OpenTelemetry Demo

The official basic install is:

```bash
helm install my-otel-demo \
open-telemetry/opentelemetry-demo
```

But because we're using our own namespace, use:

```bash
helm install my-otel-demo \
open-telemetry/opentelemetry-demo \
--namespace otel-demo
```

The official chart installation method comes directly from the current OpenTelemetry Kubernetes documentation. ([OpenTelemetry][1])

---

# 15. Watch pods starting

Run:

```bash
kubectl get pods -n otel-demo
```

For continuous monitoring:

```bash
kubectl get pods -n otel-demo -w
```

Initially you might see:

```text
ContainerCreating
Init:0/1
Pending
Running
```

Eventually most pods should become:

```text
Running
```

Use:

```bash
kubectl get pods -n otel-demo
```

You will see many components.

Something similar to:

```text
accounting
ad
cart
checkout
currency
email
flagd
flagd-ui
fraud-detection
frontend
frontend-proxy
image-provider
kafka
load-generator
otel-collector
payment
product-catalog
quote
recommendation
shipping
```

plus observability components.

---

# 16. See everything deployed

Run:

```bash
kubectl get all -n otel-demo
```

Also:

```bash
kubectl get deployments -n otel-demo
```

```bash
kubectl get svc -n otel-demo
```

```bash
kubectl get pods -n otel-demo -o wide
```

---

# 17. Check Helm installation

```bash
helm list -n otel-demo
```

Expected:

```text
NAME            NAMESPACE
my-otel-demo    otel-demo
```

Check:

```bash
helm status my-otel-demo -n otel-demo
```

---

# 18. Access the OpenTelemetry application

By default, the services are primarily internal `ClusterIP` services.

OpenTelemetry recommends either:

```text
kubectl port-forward
```

or

```text
LoadBalancer / Ingress
```

to expose the demo. ([OpenTelemetry][1])

For your first test, use port-forward.

---

# 19. Port-forward frontend-proxy

Official documentation uses the `frontend-proxy` service on port `8080`. ([OpenTelemetry][1])

Run:

```bash
kubectl port-forward \
-n otel-demo \
svc/frontend-proxy \
8080:8080 \
--address 0.0.0.0
```

Because you're running from EC2, `--address 0.0.0.0` lets you access it through the EC2 machine.

---

# 20. Open port 8080 on EC2

Go to:

```text
EC2
→ eks-management
→ Security
→ Security Group
→ Edit inbound rules
```

Add:

```text
Type       Custom TCP
Port       8080
Source     My IP
```

Don't use:

```text
0.0.0.0/0
```

for a permanent setup.

Now browse:

```text
http://EC2-PUBLIC-IP:8080
```

You should see the OpenTelemetry Astronomy Shop.

---

# 21. Access Grafana

Because `frontend-proxy` routes several demo components, the official documentation provides Grafana under:

```text
/grafana/
```

([OpenTelemetry][1])

Open:

```text
http://EC2-PUBLIC-IP:8080/grafana/
```

---

# 22. Access Jaeger

Open:

```text
http://EC2-PUBLIC-IP:8080/jaeger/ui/
```

The OpenTelemetry docs expose Jaeger through that frontend-proxy path. ([OpenTelemetry][1])

Inside Jaeger you can inspect traces like:

```text
frontend
       |
       v
cart
       |
       v
checkout
       |
       +------ payment
       |
       +------ shipping
       |
       +------ email
```

This is one of the main benefits of this demo.

---

# 23. Access Feature Flag UI

Open:

```text
http://EC2-PUBLIC-IP:8080/feature
```

This is also an officially documented frontend-proxy route. ([OpenTelemetry][1])

---

# 24. Better EKS approach: expose it through AWS LoadBalancer

For teaching, port-forward is okay.

For a more realistic AWS architecture, change `frontend-proxy` to:

```text
LoadBalancer
```

OpenTelemetry explicitly supports changing each component's `service.type`, and gives `LoadBalancer` for `frontend-proxy` as an example. ([OpenTelemetry][1])

Create:

```bash
vi values.yaml
```

Add:

```yaml
components:
  frontend-proxy:
    service:
      type: LoadBalancer
```

Then install using:

```bash
helm uninstall my-otel-demo -n otel-demo
```

Reinstall:

```bash
helm install my-otel-demo \
open-telemetry/opentelemetry-demo \
-n otel-demo \
-f values.yaml
```

---

# 25. Find AWS LoadBalancer

Run:

```bash
kubectl get svc -n otel-demo
```

Find:

```text
frontend-proxy
```

You should eventually see:

```text
TYPE           EXTERNAL-IP
LoadBalancer   xxxxx.elb.amazonaws.com
```

Example:

```text
frontend-proxy   LoadBalancer   10.100.10.10   abc123.ap-south-1.elb.amazonaws.com
```

Open:

```text
http://abc123.ap-south-1.elb.amazonaws.com:8080
```

Depending on the generated Service configuration, inspect the external port with:

```bash
kubectl get svc frontend-proxy -n otel-demo
```

---

# 26. Monitor pod health

Very important commands for your lab:

```bash
kubectl get pods -n otel-demo
```

```bash
kubectl get pods -n otel-demo -o wide
```

```bash
kubectl get events -n otel-demo --sort-by=.lastTimestamp
```

```bash
kubectl describe pod PODNAME -n otel-demo
```

```bash
kubectl logs PODNAME -n otel-demo
```

For containers with multiple containers:

```bash
kubectl logs PODNAME \
-n otel-demo \
-c CONTAINER-NAME
```

---

# 27. Check OpenTelemetry Collector

Find collector:

```bash
kubectl get pods -n otel-demo | grep collector
```

Then:

```bash
kubectl logs \
-n otel-demo \
$(kubectl get pods -n otel-demo \
-o name | grep collector | head -1)
```

The collector receives telemetry from the applications and routes it into the configured observability backends.

Conceptually:

```text
Applications
    |
    | OTLP
    |
    v
OpenTelemetry Collector
    |
    +--------- Traces ---------> Jaeger
    |
    +--------- Metrics --------> Prometheus
    |
    +--------- Metrics --------> Grafana visualization
    |
    +--------- Logs -----------> Backend
```

---

# 28. Check Product Catalog specifically

Since Product Catalog is a common place to notice issues, check:

```bash
kubectl get pods -n otel-demo | grep product
```

Then:

```bash
kubectl logs \
-n otel-demo \
$(kubectl get pods -n otel-demo \
-o name | grep product-catalog | head -1)
```

If the products aren't loading, also check:

```bash
kubectl describe pod \
-n otel-demo \
$(kubectl get pods -n otel-demo \
-o name | grep product-catalog | head -1)
```

---

# 29. Check resource utilization

Install Metrics Server if needed:

```bash
kubectl apply -f \
https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Then after metrics are available:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods -n otel-demo
```

This becomes extremely useful because the demo contains many microservices.

---

# 30. If pods show Pending

Run:

```bash
kubectl get pods -n otel-demo
```

Then:

```bash
kubectl describe pod PODNAME -n otel-demo
```

Look at:

```text
Events:
```

Common causes include:

```text
Insufficient memory
Insufficient cpu
ImagePullBackOff
FailedScheduling
PVC issue
node capacity
```

If you see:

```text
0/2 nodes are available:
2 Insufficient memory
```

increase node capacity.

---

# 31. Scale EKS nodes if required

For example:

```bash
eksctl scale nodegroup \
--cluster otel-demo-cluster \
--name otel-nodes \
--nodes 3 \
--nodes-min 2 \
--nodes-max 4 \
--region ap-south-1
```

Then:

```bash
kubectl get nodes
```

---

# 32. Very important: don't use tiny nodes

Because OpenTelemetry explicitly asks for around **6 GB free RAM**, trying something like:

```text
2 x t3.small
```

gives only:

```text
4 GB total RAM
```

which is below the application's own requirement.

Even:

```text
2 x t3.medium
```

provides:

```text
8 GB total
```

but after:

```text
OS
kubelet
kube-proxy
VPC CNI
CoreDNS
system pods
```

the remaining capacity can become tight.

Therefore I would use:

```text
Minimum recommended for class:

2 × t3.large
```

---

# 33. OpenTelemetry Demo data flow

This is what you should explain to students:

```text
USER
 |
 v
Frontend Proxy
 |
 v
Frontend
 |
 +---------------------+
 |                     |
 v                     v
Product Catalog     Recommendation
 |
 v
Cart
 |
 v
Checkout
 |
 +---------+----------+----------+
 |         |          |          |
 v         v          v          v
Payment Shipping   Currency    Email


Every instrumented service
          |
          |
          | traces
          | metrics
          | logs
          v
+-----------------------------+
| OpenTelemetry Collector     |
+-----------------------------+
          |
     +----+----------------+
     |                     |
     v                     v
  Jaeger              Prometheus
  Traces                Metrics
                           |
                           v
                        Grafana
```

---

# 34. Useful verification commands

I suggest keeping these together:

```bash
kubectl get nodes
```

```bash
kubectl get pods -n otel-demo
```

```bash
kubectl get svc -n otel-demo
```

```bash
kubectl get deployments -n otel-demo
```

```bash
kubectl get all -n otel-demo
```

```bash
kubectl get events -n otel-demo \
--sort-by='.lastTimestamp'
```

```bash
helm list -n otel-demo
```

```bash
helm status my-otel-demo -n otel-demo
```

---

# 35. One important OpenTelemetry Helm limitation

The current OpenTelemetry documentation says the Demo Helm chart **does not support normal upgrades from one chart version to another**.

For a new chart version, they instruct you to remove the old release and install the new release instead. ([OpenTelemetry][1])

For example:

```bash
helm uninstall my-otel-demo -n otel-demo
```

then:

```bash
helm repo update
```

then reinstall:

```bash
helm install my-otel-demo \
open-telemetry/opentelemetry-demo \
-n otel-demo
```

---

# 36. Delete only OpenTelemetry Demo

When the class/lab is done:

```bash
helm uninstall my-otel-demo -n otel-demo
```

Then:

```bash
kubectl delete namespace otel-demo
```

---

# 37. Delete entire EKS cluster

When you no longer need the cluster:

```bash
eksctl delete cluster \
--name otel-demo-cluster \
--region ap-south-1
```

AWS documents `eksctl delete cluster` as the cleanup mechanism for clusters created this way. ([AWS Documentation][8])

This is important so EKS control-plane, EC2, networking, and load-balancer costs do not continue unnecessarily.

---

# Recommended setup for your KK FUNDA lab

I would use this exact configuration:

```text
Region:
ap-south-1

Cluster:
otel-demo-cluster

Kubernetes:
1.36

Node Group:
otel-nodes

Node Type:
t3.large

Desired Nodes:
2

Minimum:
2

Maximum:
3

Node Disk:
30 GB gp3

Namespace:
otel-demo

Helm Release:
my-otel-demo
```

And your main cluster creation command becomes:

```bash
eksctl create cluster \
--name otel-demo-cluster \
--region ap-south-1 \
--version 1.36 \
--nodegroup-name otel-nodes \
--node-type t3.large \
--nodes 2 \
--nodes-min 2 \
--nodes-max 3 \
--node-volume-size 30 \
--managed
```

Then essentially:

```bash
kubectl create namespace otel-demo
```

```bash
helm repo add open-telemetry \
https://open-telemetry.github.io/opentelemetry-helm-charts
```

```bash
helm repo update
```

```bash
helm install my-otel-demo \
open-telemetry/opentelemetry-demo \
-n otel-demo
```

```bash
kubectl get pods -n otel-demo -w
```

Then expose:

```bash
kubectl port-forward \
-n otel-demo \
svc/frontend-proxy \
8080:8080 \
--address 0.0.0.0
```

