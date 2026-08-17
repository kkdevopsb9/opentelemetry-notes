For Docker + Docker Compose on AWS EC2, I recommend doing the **full OpenTelemetry Demo on a single Ubuntu EC2 instance** first. The official Docker guide currently requires Docker, Docker Compose v2+, about **6 GB RAM**, and **14 GB disk space**. The reduced “minimal mode” needs about **3 GB RAM**. ([OpenTelemetry][1])

For a training/demo setup, use this EC2 configuration:

| Item           | Recommendation              |
| -------------- | --------------------------- |
| AMI            | Ubuntu Server 24.04 LTS     |
| Architecture   | x86_64                      |
| Instance       | **t3.large**                |
| vCPU           | 2                           |
| RAM            | 8 GB                        |
| Disk           | **30 GB gp3**               |
| Region         | ap-south-1                  |
| Public IPv4    | Enable                      |
| Security Group | SSH 22 + HTTP app port 8080 |

`t3.large` is the safest starting point because the full demo itself expects about 6 GB RAM. I would **not use t2.small/t3.small** for the full demo; memory pressure can cause services such as the Collector, product catalog, Kafka, or frontend dependencies to fail. ([OpenTelemetry][1])

## 1. Create the EC2 instance

In AWS:

```text
EC2
→ Launch Instance
```

Use:

```text
Name: opentelemetry-demo

AMI:
Ubuntu Server 24.04 LTS

Architecture:
64-bit x86

Instance type:
t3.large

Storage:
30 GB gp3
```

Security Group:

```text
22    TCP    My IP
8080  TCP    My IP
```

For classroom use you can temporarily allow:

```text
8080 TCP 0.0.0.0/0
```

but avoid exposing unnecessary backend ports publicly.

The good thing about the current demo is that the main browser-facing services are routed through the frontend proxy on **port 8080**, including Grafana and Jaeger. ([OpenTelemetry][1])

---

# 2. Connect to Ubuntu EC2

From your laptop:

```bash
ssh -i kk.pem ubuntu@EC2-PUBLIC-IP
```

Example:

```bash
ssh -i kk.pem ubuntu@13.233.10.20
```

Verify:

```bash
whoami
hostname
free -h
df -h
```

You should see roughly:

```text
RAM: ~8 GB
Disk: ~30 GB
```

---

# 3. Update the server

```bash
sudo apt update
sudo apt upgrade -y
```

Install utilities:

```bash
sudo apt install -y \
git \
curl \
wget \
ca-certificates \
gnupg \
make
```

---

# 4. Install Docker

Install Docker's repository key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Give read permission:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Add Docker repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update:

```bash
sudo apt update
```

Install Docker Engine and Compose:

```bash
sudo apt install -y \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
```

---

# 5. Start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Check:

```bash
sudo systemctl status docker
```

Press:

```text
q
```

to exit.

---

# 6. Allow Ubuntu user to use Docker

```bash
sudo usermod -aG docker ubuntu
```

Apply the new group immediately:

```bash
newgrp docker
```

Test:

```bash
docker ps
```

Then:

```bash
docker version
```

and:

```bash
docker compose version
```

The official demo requires Docker Compose **v2.0.0+**. ([OpenTelemetry][1])

---

# 7. Test Docker

```bash
docker run hello-world
```

You should see:

```text
Hello from Docker!
```

---

# 8. Clone the official OpenTelemetry Demo

The official guide currently uses:

```bash
git clone https://github.com/open-telemetry/opentelemetry-demo.git
```

Then:

```bash
cd opentelemetry-demo
```

([OpenTelemetry][1])

Check:

```bash
ls
```

You should see files/directories such as:

```text
docker-compose.yml
docker-compose.minimal.yml
src
Makefile
.env
```

---

# 9. Before starting, verify resources

Run:

```bash
free -h
```

Then:

```bash
df -h
```

and:

```bash
docker system df
```

For the full demo you ideally want:

```text
RAM:
8 GB EC2

Free RAM before starting:
6+ GB

Disk:
30 GB

Free disk:
20+ GB
```

The official minimum is approximately **6 GB RAM and 14 GB disk**. ([OpenTelemetry][1])

---

# 10. Start OpenTelemetry Demo

You have two supported options.

Using Make:

```bash
make start
```

Or directly using Compose:

```bash
docker compose up \
  --force-recreate \
  --remove-orphans \
  --detach
```

Both approaches are documented by OpenTelemetry. ([OpenTelemetry][1])

For teaching Docker Compose, I recommend using:

```bash
docker compose up \
  --force-recreate \
  --remove-orphans \
  -d
```

because students can directly understand what Docker Compose is doing.

---

# 11. Check containers

```bash
docker compose ps
```

Also:

```bash
docker ps
```

You should see many containers because this is a complete microservices demo.

The architecture includes services written in several languages communicating through HTTP/gRPC, plus a load generator that automatically creates application traffic. ([OpenTelemetry][2])

Typical services include:

```text
frontend
frontend-proxy
product-catalog
cart
checkout
currency
payment
shipping
recommendation
ad
email
accounting
fraud-detection
load-generator
otel-collector
grafana
jaeger
prometheus
kafka
flagd
```

---

# 12. Check container health

Run:

```bash
docker compose ps
```

You don't want to see repeated:

```text
Restarting
Exited
Unhealthy
```

Check all container names:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

---

# 13. Open the application

Get EC2 public IP:

```bash
curl ifconfig.me
```

Suppose:

```text
13.233.100.20
```

Open:

```text
http://13.233.100.20:8080/
```

The official demo routes the web store through port 8080. ([OpenTelemetry][1])

## Main application

```text
http://PUBLIC-IP:8080/
```

Example:

```text
http://13.233.100.20:8080/
```

This is the OpenTelemetry Astronomy Shop.

---

# 14. Grafana

Open:

```text
http://PUBLIC-IP:8080/grafana/
```

Example:

```text
http://13.233.100.20:8080/grafana/
```

The official demo stores metric dashboards in Grafana. ([OpenTelemetry][1])

---

# 15. Jaeger

Open:

```text
http://PUBLIC-IP:8080/jaeger/ui/
```

Example:

```text
http://13.233.100.20:8080/jaeger/ui/
```

Jaeger receives generated traces from the demo. ([OpenTelemetry][1])

This is where I suggest explaining:

```text
Trace
   ↓
Span
   ↓
Parent Span
   ↓
Child Span
   ↓
Service latency
   ↓
Distributed request flow
```

---

# 16. OpAMP UI

The current August 2026 documentation also exposes:

```text
http://PUBLIC-IP:8080/opamp/
```

([OpenTelemetry][1])

---

# 17. Feature Flag UI

Open:

```text
http://PUBLIC-IP:8080/feature
```

([OpenTelemetry][1])

You can later use this for fault-injection scenarios.

---

# 18. How the architecture works

You can explain the lab to students like this:

```text
                         USER
                          |
                          v
                  +----------------+
                  | Frontend Proxy |
                  |     Envoy      |
                  +--------+-------+
                           |
                           v
                     +----------+
                     | Frontend |
                     +----+-----+
                          |
          +---------------+---------------+
          |               |               |
          v               v               v

 Product Catalog       Cart           Checkout
       |                |                 |
       |                |                 |
       v                v                 v

 Recommendation      Currency          Payment
                                         |
                                         v
                                      Shipping

          Application Microservices
                    |
                    |
             OTLP telemetry
                    |
                    v

          +---------------------+
          | OpenTelemetry       |
          | Collector           |
          +----------+----------+
                     |
          +----------+-----------+
          |                      |
          v                      v
       Jaeger                Prometheus
       Traces                 Metrics
                                  |
                                  v
                               Grafana
```

OpenTelemetry itself is responsible for generating, collecting, processing, and exporting telemetry; tools such as Jaeger, Prometheus and Grafana provide storage/querying/visualization functions around those signals. ([OpenTelemetry][3])

---

# 19. The three main signals

This is important for your class.

### Traces

```text
User
 ↓
Frontend
 ↓
Product Catalog
 ↓
Cart
 ↓
Checkout
 ↓
Payment
```

You see this in:

```text
Jaeger
```

### Metrics

Examples:

```text
Request count
CPU usage
Memory
Latency
HTTP error rate
Service request duration
```

You mainly explore them through:

```text
Prometheus
Grafana
```

### Logs

Application-generated logs provide events and diagnostic messages.

The demo currently demonstrates observability across traces, metrics and logs, while Grafana stores its dashboards, Jaeger handles traces, and Prometheus scrapes generated metrics/exemplars. ([OpenTelemetry][4])

---

# 20. OpenTelemetry Collector

This component is especially important.

Conceptually:

```text
Application
    |
    |
    | OTLP
    v
+---------------------+
| OTel Collector      |
|                     |
| Receivers           |
|     ↓               |
| Processors          |
|     ↓               |
| Exporters           |
+----------+----------+
           |
     +-----+------+
     |            |
     v            v
  Jaeger       Prometheus
```

You can show its config:

```bash
cd ~/opentelemetry-demo
```

Then:

```bash
cat src/otel-collector/otelcol-config.yml
```

The current demo also merges:

```text
otelcol-config.yml
otelcol-config-extras.yml
```

when configuring the Collector. ([OpenTelemetry][1])

---

# 21. Check Collector logs

```bash
docker compose logs otel-collector
```

Live:

```bash
docker compose logs -f otel-collector
```

Press:

```text
Ctrl+C
```

to exit log-follow mode.

---

# 22. Check frontend logs

```bash
docker compose logs frontend
```

For frontend proxy:

```bash
docker compose logs frontend-proxy
```

---

# 23. Check product catalog

Since you had trouble with this service previously, this will be one of our first checks:

```bash
docker compose logs product-catalog
```

Live:

```bash
docker compose logs -f product-catalog
```

---

# 24. Look at resource usage

Very important:

```bash
docker stats
```

Watch:

```text
CPU %
MEM USAGE
MEM %
NET I/O
```

For example:

```text
CONTAINER            CPU      MEM
otel-collector       10%      300MB
frontend             2%       150MB
product-catalog      3%       120MB
grafana              1%       200MB
...
```

If total RAM approaches your EC2 limit, you will start seeing failures.

---

# 25. Check host memory

```bash
free -h
```

I would also install:

```bash
sudo apt install -y htop
```

Then:

```bash
htop
```

---

# 26. Minimal mode

If you want to save AWS cost, you could use minimal mode.

The official command is:

```bash
docker compose \
  -f docker-compose.minimal.yml \
  up \
  --force-recreate \
  --remove-orphans \
  --detach
```

Minimal mode uses approximately **3 GB RAM** and excludes:

```text
accounting
fraud-detection
flagd-ui
kafka
```

([OpenTelemetry][1])

For minimal mode you could reasonably use:

```text
t3.medium
2 vCPU
4 GB RAM
30 GB gp3
```

But for your training, I recommend the full version.

---

# 27. My AWS recommendation

Use:

```text
Ubuntu 24.04
       |
       v
t3.large
2 vCPU
8 GB RAM
30 GB gp3
       |
       v
Docker Engine
       |
       v
Docker Compose v2
       |
       v
OpenTelemetry Demo
```

### Don't start with

```text
t2.micro
t3.micro
t2.small
t3.small
```

for the complete demo.

### Better

```text
t3.large
```

### If you want extra safety during teaching

```text
t3.xlarge
4 vCPU
16 GB RAM
```

especially if you plan to demonstrate load generation while keeping Grafana/Jaeger/Collector responsive.

---

# 28. Ports in AWS

For the basic lab, your Security Group can be very simple:

```text
22     SSH
8080   OpenTelemetry Demo
```

The browser URLs become:

```text
Application
http://PUBLIC-IP:8080/

Grafana
http://PUBLIC-IP:8080/grafana/

Jaeger
http://PUBLIC-IP:8080/jaeger/ui/

OpAMP
http://PUBLIC-IP:8080/opamp/

Feature Flags
http://PUBLIC-IP:8080/feature
```

This routing is intentional in the current demo: the frontend proxy handles browser traffic on the primary port, which defaults to 8080. ([OpenTelemetry][1])

---

# 29. Useful Docker Compose commands

Check:

```bash
docker compose ps
```

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose stop
```

Restart:

```bash
docker compose restart
```

Logs:

```bash
docker compose logs
```

Specific service:

```bash
docker compose logs product-catalog
```

Live logs:

```bash
docker compose logs -f
```

Remove containers:

```bash
docker compose down
```

Start fresh:

```bash
docker compose down
```

```bash
docker compose up \
  --force-recreate \
  --remove-orphans \
  -d
```

---

# 30. If the frontend doesn't open

First:

```bash
docker compose ps
```

Then:

```bash
docker compose logs frontend-proxy
```

Check port:

```bash
sudo ss -lntp | grep 8080
```

Check locally:

```bash
curl http://localhost:8080
```

If localhost works but:

```text
http://PUBLIC-IP:8080
```

doesn't work, the problem is likely:

```text
AWS Security Group
```

rather than Docker.

---

# 31. If products don't load

Check:

```bash
docker compose logs product-catalog
```

Then:

```bash
docker compose logs frontend
```

Then:

```bash
docker compose logs otel-collector
```

Then:

```bash
docker stats
```

Then:

```bash
free -h
```

This sequence is particularly important because service timeouts combined with Collector memory errors can be symptoms of host memory pressure.

---

# 32. Stop everything after class

```bash
cd ~/opentelemetry-demo
```

```bash
docker compose down
```

Check:

```bash
docker ps
```

---

# 33. Delete Docker data if needed

Be careful with this command:

```bash
docker system prune -a
```

It removes unused images, containers and other Docker data.

For your training machine:

```bash
docker system df
```

first.

---

## Complete learning flow I recommend for your students

Don't only install the demo. Teach it in this order:

```text
DAY 1

Observability
   ↓
Monitoring vs Observability
   ↓
Logs / Metrics / Traces
   ↓
OpenTelemetry
   ↓
Instrumentation
   ↓
OpenTelemetry SDK
   ↓
OTLP
   ↓
Collector
   ↓
Receiver
   ↓
Processor
   ↓
Exporter


DAY 2

AWS EC2
   ↓
Docker
   ↓
Docker Compose
   ↓
OpenTelemetry Demo
   ↓
Microservices architecture
   ↓
Generate traffic


DAY 3

Jaeger
   ↓
Distributed tracing
   ↓
Trace ID
   ↓
Span ID
   ↓
Parent/Child spans
   ↓
Latency analysis


DAY 4

Prometheus
   ↓
Metrics
   ↓
Grafana
   ↓
Dashboards
   ↓
Service monitoring


DAY 5

Collector configuration
   ↓
Receivers
   ↓
Processors
   ↓
Exporters
   ↓
OTLP gRPC / HTTP
   ↓
Troubleshooting
```

