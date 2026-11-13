# fintech-fraud-detection

A financial transaction processor that flags amounts of $10,000 CAD or more (the FINTRAC reporting threshold), deployed on a self-hosted 3-node Kubernetes cluster running on KVM/QEMU virtual machines.

The app is deliberately small (one rule). The point of the project is the platform around it: self-healing, autoscaling, load balancing and rolling updates on a cluster I built myself.

**Key results** (from the screenshots below, recorded 10–11 March 2026):

- 3/3 nodes Ready on k3s v1.34.4.
- A deleted pod was replaced and Running again **2 s** later, with no manual step.
- Under load the HPA scaled **1 → 5 pods** (each at 94–111m CPU against a 50m target), then back to **1 pod** (3m CPU) once the load stopped.

## Architecture

<p align="center"><img src="docs/architecture.svg" width="100%" alt="A client sends POST /transaction to a Kubernetes Service (LoadBalancer :80 to container port 8000, NodePort 31456 in the demos). The Service spreads requests over 1 to 5 FastAPI pods of the transaction-processor Deployment in the fintech namespace. An HPA adds pods when CPU is above 50 percent of the 100m request. Everything runs on k3s v1.34.4 across three Ubuntu 22.04 VMs (one control plane, two workers) on KVM/QEMU on Nobara Linux."></p>

**Why this architecture**

| Choice | Why |
|---|---|
| k3s on 3 KVM VMs (`infrastructure/create-vms.sh`) | Runs on my own laptop with no cloud account; the Deployments, Services and HPA are the same objects as on any Kubernetes cluster. |
| 1 control plane + 2 workers | Pods have more than one node to land on, so scheduling and recovery are real, not single-machine. |
| Small CPU request (100m) and limit (250m) | The HPA measures CPU as a % of the request; with 100m, a curl loop is enough to cross the 50% target and show scaling. |
| Pod name injected with the Downward API (`POD_NAME`) | Every reply says which pod served it, so load balancing is visible without extra tools. |
| Stateless FastAPI app with one rule | Any pod can answer any request, so pods can be added, killed and replaced safely; the focus stays on Kubernetes behaviour. |
| Rolling update as a second manifest that only changes `APP_VERSION` | Shows the rollout mechanics on their own, without a new image build. |

## What This Demonstrates

| Concept | How |
|---|---|
| **Self-healing** | Delete a pod — Kubernetes detects the failure and automatically schedules a replacement |
| **Horizontal autoscaling (HPA)** | Flood with traffic — pods scale from 1 → 5 automatically, then back to 1 when idle |
| **Load balancing** | Every response includes `handled_by: <pod-name>` showing traffic distributed across replicas |
| **Rolling updates** | Apply v2 manifest — old pods replaced one by one with zero downtime |
| **Namespaces** | All workloads isolated in the `fintech` namespace |

## Infrastructure

- **Hypervisor:** KVM/QEMU on Nobara Linux
- **Nodes:** 3x Ubuntu 22.04 VMs (k3s-control, k3s-worker-1, k3s-worker-2)
- **Kubernetes:** k3s v1.34.4
- **App:** Python 3.11 + FastAPI

## Results

What was tested on the cluster and what the screenshots show. There are no automated tests or CI in this repo; these are manual runs.

| Test | How | Result | Evidence |
|---|---|---|---|
| Cluster health | `kubectl get nodes` | 3/3 nodes Ready, v1.34.4+k3s1 | `screenshots/cluster-nodes.png` |
| Rule above threshold | POST $15,000 (TD Bank) | `flagged`, `fintrac_report_required: true` | `screenshots/transaction-flagged.png` |
| Rule below threshold | POST $8 (Tim Hortons) | `approved`, `fintrac_report_required: false` | `screenshots/transaction-approved.png` |
| Self-healing | `kubectl delete pod` | Replacement pod 1/1 Running at age 2 s, 0 restarts | `screenshots/self-healing.png` |
| Scale up | Parallel curl loops against NodePort 31456 | 5/5 pods Running (HPA max), 94–111m CPU and 35–38 Mi memory each | `screenshots/hpa-scale-up.png` |
| Scale down | Stop the load | 1 pod left, 3m CPU (screenshot taken about 6.5 min after the scale-up one) | `screenshots/hpa-scale-down.png` |
| Rolling update v1 → v2 | `scripts/demo-rolling-update.sh` | Not recorded: no screenshot or log of this run is in the repo | – |

## What worked and what didn't

| | What | Detail |
|---|---|---|
| ✅ | Self-healing | The ReplicaSet replaced a deleted pod in about 2 s with no manual step. |
| ✅ | Autoscaling both ways | 1 → 5 pods under load and back to 1 when idle, driven only by `k8s/hpa.yaml`. |
| ✅ | Visible load balancing | `handled_by` in every reply shows which pod answered |
| ⚠️ | Slow scale-down | `hpa.yaml` has no `behavior` block, so Kubernetes' default 5-minute scale-down window applies: extra pods stay for minutes after the load stops. |
| ❌ | No health probes | `deployment.yaml` has no readiness or liveness probe, so Kubernetes sends traffic as soon as the container starts. "Zero downtime" for the rolling update is the design intent, not a measured result. |
| ❌ | Image name mismatch | The Deploy section builds `myviewsontech/fintech-fraud-detection:v1`, but the manifests pull `kchimbodza/fraud-detection-k8s:v1`. Tag the build to match the manifest (or edit it). |
| ❌ | `scripts/load-test.sh` is sequential | It sends 200 requests one at a time with a 0.1 s pause. The scale-up screenshot used the parallel loop from [Demos](#demos) instead. |
| ⚠️ | "Fraud detection" is one rule | The app checks one threshold (`amount >= 10000`). There is no scoring model and no tests for the rule. |

---

## Screenshots

### 1. The Cluster — 3 Nodes Ready
![Cluster Nodes](screenshots/cluster-nodes.png)
*The foundation of the project: a self-hosted 3-node Kubernetes cluster running on virtual machines. All three nodes (one control plane, two workers) show STATUS: Ready — meaning the cluster is healthy and able to schedule workloads.*

---

### 2. Application Deployed — Pod Running
![Pod Running](screenshots/pod-running.png)
*The transaction processor application is deployed and running inside the `fintech` namespace. STATUS: Running and READY: 1/1 confirms the app is live and accepting requests.*

---

### 3. Transaction Flagged — FINTRAC Threshold Triggered
![Transaction Flagged](screenshots/transaction-flagged.png)
*A $15,000 CAD transaction from TD Bank is processed. Because the amount exceeds the $10,000 FINTRAC reporting threshold, the API returns `"status": "flagged"` and `"fintrac_report_required": true`. The `handled_by` field shows which pod processed the request.*

---

### 4. Transaction Approved — Below Threshold
![Transaction Approved](screenshots/transaction-approved.png)
*An $8 CAD transaction from Tim Hortons is processed. The amount is below the threshold so the API returns `"status": "approved"` and `"fintrac_report_required": false`. This confirms the flagging logic correctly distinguishes between transactions.*

---

### 5. Self-Healing — Automatic Pod Recovery
![Self Healing](screenshots/self-healing.png)
*The pod is manually deleted to simulate a crash. Kubernetes immediately detects the missing pod and schedules a replacement — the new pod is Running within 2 seconds with zero manual intervention.*

---

### 6. HPA Scale-Up — 1 Pod Grows to 5 Under Load
![HPA Scale Up](screenshots/hpa-scale-up.png)
*50 parallel workers flood the API with requests, pushing CPU usage to ~110m per pod (above the 50m threshold). The Horizontal Pod Autoscaler automatically scales from 1 pod to 5 pods to handle the load.*

---

### 7. HPA Scale-Down — Back to 1 Pod When Idle
![HPA Scale Down](screenshots/hpa-scale-down.png)
*Once the load test stops, CPU drops back to ~3m. After a cooldown period, the HPA automatically terminates the extra pods and scales back down to 1 — releasing resources when they are no longer needed.*

---

## API

**Health check**
```
GET /
```
```json
{ "status": "ok", "pod": "transaction-processor-abc123", "version": "v1" }
```

**Process transaction**
```
POST /transaction
{ "merchant": "TD Bank", "amount": 15000, "currency": "CAD" }
```
```json
{
  "status": "flagged",
  "merchant": "TD Bank",
  "amount": 15000.0,
  "currency": "CAD",
  "fintrac_report_required": true,
  "handled_by": "transaction-processor-abc123",
  "version": "v1"
}
```

---

## Deploy

### 1. Build and push the image
```bash
cd app/
docker build -t myviewsontech/fintech-fraud-detection:v1 .
docker push myviewsontech/fintech-fraud-detection:v1
```

> `k8s/deployment.yaml` pulls `kchimbodza/fraud-detection-k8s:v1`. Use the same name when you tag, or change the `image:` line.

### 2. Deploy to cluster
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml
```

### 3. Verify
```bash
kubectl get pods -n fintech
kubectl get svc -n fintech
```

---

## Demos

### Self-healing
```bash
bash scripts/demo-self-healing.sh
```

### Load test (triggers HPA)
```bash
# Terminal 1 - watch pods scale
kubectl get pods -n fintech -w

# Terminal 2 - send traffic
for i in $(seq 1 50); do
  while true; do
    curl -s -X POST http://<NODE-IP>:31456/transaction \
      -H "Content-Type: application/json" \
      -d "{\"merchant\": \"Test\", \"amount\": $((RANDOM % 20000))}" > /dev/null
  done &
done
```

### Rolling update
```bash
bash scripts/demo-rolling-update.sh
```

---

## Project Structure

```
fintech-fraud-detection/
├── docs/
│   └── architecture.svg
├── app/
│   ├── main.py
│   ├── Dockerfile
│   └── requirements.txt
├── infrastructure/
│   └── create-vms.sh
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── rolling-update/
│       └── deployment-v2.yaml
├── screenshots/
└── scripts/
    ├── demo-self-healing.sh
    ├── demo-rolling-update.sh
    └── load-test.sh
```
