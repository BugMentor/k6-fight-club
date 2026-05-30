# 🥊 K6 Fight Club

🥊 "K6 Fight Club" Monorepo (Grafana & Friends Mendoza). A Grafana k6 performance testing suite comparing extreme concurrency handling across Go (Gin/GraphQL) and Java (Spring Boot GraphQL & SOAP). Built to demonstrate JVM GC tuning, K8s OOMKills, and infrastructure limits during live stress tests!

---

## 🏗️ Architecture

```mermaid
graph TD
    k6[("k6 Load Test")] -->|Attacks| Ingress["K8s Ingress"]
    Ingress --> Pods["Scaled Pods (Go/Java)"]
    Pods --> DB[("PostgreSQL")]
    
    subgraph "Observability (LGTM)"
        Mimir["Mimir (Metrics)"]
        Loki["Loki (Logs)"]
        Tempo["Tempo (Traces)"]
    end
    
    Pods --> Mimir
    Pods --> Loki
    Pods --> Tempo
    
    Mimir & Loki & Tempo --> Grafana["Grafana"]
```

---

## 🐳 Floci EKS Simulation

All three contenders run on **Floci** — a Docker-based EKS simulator that mocks the full AWS EKS + ELB lifecycle using LocalStack, K3s, and Metrics Server for HPA-driven scaling.

```mermaid
graph TD
    subgraph Host["Docker Host"]
        subgraph FlociServices["Floci Infrastructure"]
            FL["Floci Container<br/>floci/floci:latest<br/>📦 AWS API (port 4566)<br/>☸️ K3s (port 6443)<br/>📊 Metrics Server"]
            REG["Registry<br/>registry:2<br/>port 5000"]
        end

        subgraph EKS["EKS Cluster (K3s inside Floci)"]
            subgraph payments["namespace: payments"]
                ELB["NLB Service<br/>port 80 → 8080"]
                HPA["HPA<br/>min:2 max:30<br/>CPU>80% / RAM>60%"]
                PG["PostgreSQL 16<br/>4Gi/2CPU"]
                PS1["payment-svc pod 1<br/>1Gi/1CPU"]
                PS2["payment-svc pod 2<br/>1Gi/1CPU"]
                PSX["… up to 30 pods"]
            end
        end

        K6["k6 Load Test"] -->|attacks| ELB
        ELB --> PS1 & PS2
        PS1 --> PG
        PS2 --> PG
        HPA -.->|scales| PS1
        HPA -.->|scales| PS2
        FL -->|aws eks + elbv2 mock| ELB
        REG -.->|image pull| PS1
    end
```

### Floci Boot Sequence

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant DC as docker compose
    participant FL as Floci Container
    participant K3s as K3s (in Floci)
    participant AWS as AWS API Mock
    
    Dev->>DC: docker compose -f benchmark/floci/docker-compose.yaml up
    DC->>FL: start floci/floci:latest
    DC->>FL: start registry:2
    FL->>K3s: bootstrap K3s cluster
    FL->>AWS: enable EKS, ELBv2, EC2 APIs
    
    Dev->>AWS: aws eks create-cluster
    AWS-->>Dev: cluster ARN
    
    Dev->>Dev: docker build -t payment-service:latest
    Dev->>REG: docker push payment-service:latest
    
    Dev->>Dev: kubectl apply -f payment-service-elb.yaml
    Dev->>Dev: kubectl apply -f hpa.yaml
    
    Dev->>K6: k6 run benchmark/k6/max-capacity-test.js
    K6->>ELB: 5000 VU attack
    ELB->>K3s: distribute to pods
    K3s->>HPA: CPU > 80%?
    HPA-->>K3s: scale up!
```

Each project has its own `benchmark/floci/` directory with project-tuned manifests:
- `docker-compose.yaml` — Floci container + local registry
- `payment-service-elb.yaml` — Deployment, PostgreSQL, NLB service
- `hpa.yaml` — HPA with constitutional thresholds (CPU>80%, RAM>60%)
- `setup-eks.sh` — One-command EKS bootstrap script

---

## 🏆 The Contenders

| Contender | Language | Protocol | Characteristics |
| :--- | :--- | :--- | :--- |
| `k6-go-gin-graphql-example` | Go 1.26 | GraphQL | Lightweight, high concurrency, low footprint |
| `k6-java-spring-graphql-example` | Java 21 | GraphQL | Enterprise standard, Spring Boot 3.2, Virtual Threads |
| `k6-java-spring-soap-example` | Java 21 | SOAP | Legacy integration, heavy XML parsing |

---

## 🎙️ The Narrative of the Talk

### Phase 1: The Baseline Disaster
We deploy the Java application to Kubernetes with default/poor limits (0.5 CPU, 1Gi RAM). Under k6 load, it suffers from JVM Garbage Collection death spirals, CPU throttling, and instant K8s OOMKills.

### Phase 2: The High-Performance Profile
We recover the system by tuning the JVM and Infrastructure:
- **K8s**: Limits increased to 2 Cores, 4Gi RAM. HPA configured up to 30 replicas.
- **JVM**: Enabled Java 21 Virtual Threads, ZGC (Sub-millisecond pauses), and explicit container RAM allocation (`-XX:InitialRAMPercentage=50.0`, `-XX:MaxRAMPercentage=75.0`).
- **Database**: PostgreSQL connection limits aggressively tuned to handle the HPA scaling.

### Phase 3: The 10k VU Death Ramp
We run a ramping-vus k6 script reaching 10,000 concurrent users.

```mermaid
xychart-beta
    title "The 10k VU Death Ramp (0-5 min)"
    x-axis "Time (min)" [0, 1, 2, 3, 4, 5]
    y-axis "Virtual Users" 0 --> 10000
    line [0, 2000, 4000, 6000, 8000, 10000]
```

---

## 📊 Performance Metrics & Comparison
*(To be populated after load test runs)*
- [ ] Go Concurrency Ceiling
- [ ] Java GraphQL Pause Analysis (ZGC vs. G1GC)
- [ ] SOAP XML Overhead Comparison

---

## ⚠️ Critical Local Setup Warning (macOS)
**Warning:** Running 10k VUs locally on macOS requires modifying the OS open files limit first. The default limits are too low, and the host network stack will collapse before the cluster does.

Run this command before initiating your load tests:
```bash
ulimit -n 200000
```

---

## 🔧 Prerequisites

- **Docker & Compose**: For running the LGTM stack and local PostgreSQL.
- **k6**: Installed locally (`brew install k6`).
- **Java 21+ & Maven**: For Java projects.
- **Go 1.26+**: For the Go project.
- **Kubernetes Environment**: Minikube, Kind, or a remote cluster for Phase 2/3.

---

## 🛠️ How to Run

1. **Setup Infrastructure**: Navigate to `benchmark/` in the contender directory:
   ```bash
   docker-compose up -d
   ```
2. **Run Load Tests**:
   ```bash
   ./benchmark/run-benchmark.sh
   ```
3. **Monitor**: Open Grafana at `http://localhost:3000` (User: `admin`, Pass: `admin`).

---

## 🧪 Synthetic Data Generation

This monorepo includes a utility to populate the databases with 1 million synthetic records for consistent stress testing.

### Building the Generator
```bash
cd scripts/data-generator/
go build -o data-generator main.go
```

### Running the Generator
Ensure the respective PostgreSQL instance is running. Run individually for each project:

**1. Go GraphQL Project**
```bash
./scripts/data-generator/data-generator -db="postgres://payment:payment@localhost:5434/payments?sslmode=disable" -users=1000000 -payments=1000000 -clean=true
```

**2. Java Spring GraphQL Project**
```bash
./scripts/data-generator/data-generator -db="postgres://payment_user:payment_pass@localhost:55432/payment_db?sslmode=disable" -users=1000000 -payments=1000000 -clean=true
```

**3. Java Spring SOAP Project**
```bash
./scripts/data-generator/data-generator -db="postgres://payment_user:payment_pass@localhost:55432/payment_db?sslmode=disable" -users=1000000 -payments=1000000 -clean=true
```
*Note: Ensure credentials and ports match your specific `docker-compose.yaml` configuration for each project.*

---

## 🆘 Troubleshooting

- **PostgreSQL Connection Error**: Ensure DB container is healthy (`docker-compose ps`). Increase max connections if scaling HPA significantly.
- **JVM Memory Issues**: Adjust `-XX:InitialRAMPercentage` in `docker-compose.yaml` based on container limits.
- **OOMKills**: Check K8s memory requests/limits.

---

## 📜 License
*Built for "Grafana & Friends Mendoza" Meetup. Open Source under MIT License.*
