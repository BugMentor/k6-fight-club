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

## 🆘 Troubleshooting

- **PostgreSQL Connection Error**: Ensure DB container is healthy (`docker-compose ps`). Increase max connections if scaling HPA significantly.
- **JVM Memory Issues**: Adjust `-XX:InitialRAMPercentage` in `docker-compose.yaml` based on container limits.
- **OOMKills**: Check K8s memory requests/limits.

---

## 📜 License
*Built for "Grafana & Friends Mendoza" Meetup. Open Source under MIT License.*
