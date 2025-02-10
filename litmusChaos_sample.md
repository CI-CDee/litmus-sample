### **Sample Tutorial: Day 2 – Advanced Chaos Workflow for Distributed Systems**  
*(Multi-Experiment Scenario with Observability Integration)*  

---

#### **Objective**  
Demonstrate how to design a **chaos workflow** that simulates cascading failures in a distributed application, validate system recovery using observability tools (Prometheus/Grafana), and automate resilience analysis.  

**Target Audience**: SREs/Platform Engineers managing production-grade Kubernetes clusters.  
**Tools**: LitmusChaos 3.0, Prometheus, Grafana, Redis (stateful application).  

---

### **1. Pre-Requisites**  
- A Kubernetes cluster (e.g., AKS, EKS) .  
- LitmusChaos 3.0 installed with ChaosCenter and Chaos Deployments configured .  
- Prometheus & Grafana integrated for metric collection .  

---

### **2. Scenario Setup**  
**Application**: Deploy a Redis cluster with leader-follower replication.  
```yaml  
# redis-leader.yaml  
apiVersion: apps/v1  
kind: StatefulSet  
metadata:  
  name: redis-leader  
  labels:  
    app: redis  
spec:  
  serviceName: redis  
  replicas: 3  
  selector:  
    matchLabels:  
      app: redis  
  template:  
    metadata:  
      labels:  
        app: redis  
    spec:  
      containers:  
      - name: redis  
        image: redis:6.2  
        ports:  
        - containerPort: 6379  
---  
# redis-service.yaml  
apiVersion: v1  
kind: Service  
metadata:  
  name: redis  
spec:  
  ports:  
  - port: 6379  
  selector:  
    app: redis  
```  

**Observability**:  
- Configure Prometheus to scrape Redis metrics (e.g., `redis_connected_clients`, `redis_commands_processed`).  
- Create a Grafana dashboard to track latency, error rates, and leader-election status .  

---

### **3. Chaos Workflow Design**  
**Experiment Sequence**:  
1. **Pod Delete** (Leader Node): Force-delete the Redis leader pod to test Kubernetes’ self-healing.  
2. **CPU Hog**: Inject CPU stress on follower pods to simulate resource contention.  
3. **Network Latency**: Add 500ms latency between pods to test replication resilience.  

**Litmus Workflow YAML** *(Customized for Multi-Experiment Execution)*:  
```yaml  
apiVersion: litmuschaos.io/v1alpha1  
kind: ChaosEngine  
metadata:  
  name: redis-resilience-test  
spec:  
  appinfo:  
    appns: default  
    applabel: app=redis  
    appkind: statefulset  
  engineState: active  
  experiments:  
  - name: pod-delete  
    spec:  
      components:  
        env:  
        - name: TARGET_POD  
          value: redis-leader-0  # Target leader pod  
  - name: pod-cpu-hog  
    spec:  
      components:  
        env:  
        - name: TARGET_PODS  
          value: "redis-leader-1,redis-leader-2"  # Followers  
        - name: CPU_CORES  
          value: "2"  
  - name: network-latency  
    spec:  
      components:  
        env:  
        - name: LATENCY  
          value: "500ms"  
```  

---

### **4. Observability Integration**  
**Probes & Metrics**:  
- **Steady-State Hypothesis**:  
  - Redis cluster should elect a new leader within 30 seconds.  
  - Latency spikes ≤ 1s during network chaos.  
- **Grafana Alerts**:  
  - Trigger alerts if `redis_connected_clients` drops below 2 for >1 minute.  
  - Track `redis_leader_election_duration_seconds` .  

**Post-Chaos Validation**:  
```bash  
# Check leader election status  
kubectl exec -it redis-leader-0 -- redis-cli info replication  
# Output should show a new leader elected  
```  

---

### **5. Analysis & Iteration**  
**Failure Modes Identified**:  
- **Issue**: Leader election took 45s (exceeding hypothesis).  
- **Root Cause**: Kubernetes `terminationGracePeriod` delayed pod rescheduling.  
- **Fix**: Adjust `terminationGracePeriodSeconds` in StatefulSet to 15s.  

**Automation**:  
- Use GitOps to trigger this workflow on every Redis deployment update .  
- Integrate reliability scores into CI/CD pipelines (e.g., block deploys if score < 80%).  

---

### **6. Why This Approach Works**  
1. **Real-World Relevance**: Targets stateful apps (Redis) used in production, unlike generic "podtato-head" examples .  
2. **Observability-Driven**: Links chaos outcomes to measurable metrics, aligning with SRE practices .  
3. **Modular Design**: Combines 3 experiments into a single workflow, reducing manual steps .  

---

### **7. Anticipated Edge Cases & Mitigations**  
- **Blast Radius Control**: Use `annotationCheck: true` in ChaosEngine to limit chaos to test namespaces .  
- **False Positives**: Validate Grafana dashboards pre-chaos to ensure metric accuracy.  
- **Multi-Cluster Testing**: Extend the workflow to simulate cross-cluster failures (e.g., AZ outages) .  

---
