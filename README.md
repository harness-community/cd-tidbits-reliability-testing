# CD | Tidbits | Reliability Testing — Integrate Chaos Testing Post-Deployment to Validate Resilience

> **Bite-sized how-to** | ~30 min setup

---

## What is Harness Reliability Testing?

Harness Reliability Testing combines two modules — **Chaos Engineering (HCE)** and **Continuous Verification (CV)** — into a post-deployment gate inside your CD pipeline. Instead of assuming a deployment is healthy because "the pods came back," you inject a real failure and measure whether the application actually recovered.

**Chaos Engineering** lets you inject controlled faults — like deleting pods — directly into your Kubernetes cluster. An HTTP probe runs continuously during the fault window and checks whether the app stays reachable. This produces a **resilience score** that the pipeline uses as a gate.

**Continuous Verification** runs after chaos and queries your observability tool (Prometheus in this Tidbit) for a configured duration. It compares current metrics — error rate, response time — against a pre-chaos baseline. If the metrics haven't recovered, CV fails the pipeline and triggers an automatic rollback.

**How the reliability gate works:**

1. The pipeline deploys the app to staging.
2. The **Chaos step** triggers a pod delete experiment — one pod deleted, the surviving replica keeps serving traffic.
3. An **HTTP probe** continuously checks the app's health during the fault. If the app stays reachable, the resilience score is calculated as passing.
4. The **CV step** starts immediately after chaos and queries Prometheus for 5 minutes.
5. CV compares error rate and response time against the deployment baseline.
6. If both pass — pipeline succeeds. If either fails — rollback triggers automatically.

---

## What does this Tidbit demonstrate?

A complete reliability gate built into a CD pipeline — using a pod delete chaos experiment and CV backed by Prometheus.

1. **Set up connectors** — GitHub, Docker Hub, and Prometheus connectors.
2. **Install chaos infrastructure** — Harness chaos agent deployed to the `cd-chaos-staging` namespace via Project Settings → Resilience Test Infrastructure.
3. **Create the chaos experiment** — pod delete fault (50% pods affected, 60 seconds) with an HTTP probe.
4. **Set up CV** — Prometheus health source and monitored service for the app.
5. **Import the pipeline** — Build → Deploy to Staging → Reliability Validation (Chaos step → CV step).
6. **Pass path** — 50% pod delete, one replica survives, probe passes, CV confirms metrics at baseline, pipeline succeeds.
7. **Failure path** — 100% pod delete, both pods down, probe fails, resilience score = 0, automatic rollback.

---

## How the reliability gate works technically

The Reliability Validation stage has two steps in sequence:

```
Pod Delete Experiment  →  CV After Chaos
     (Chaos step)           (Verify step)
```

**Chaos step — what it does:**

The pod delete fault targets `ecommerce-app` in the `cd-chaos-staging` namespace. At 50% pods affected, one of two replicas is deleted. The HTTP probe hits the app's internal service URL throughout the 60-second fault window:

```
Probe URL: http://ecommerce-app.cd-chaos-staging.svc.cluster.local/
Method: GET  |  Expected: 200  |  Interval: 2s  |  Timeout: 5s
```

If the probe gets a 200 — the surviving replica is handling traffic — the resilience score is calculated as passing and the Chaos step passes.

**CV step — what it shows:**

Immediately after chaos, CV queries Prometheus for 5 minutes and compares two metrics against the pre-chaos baseline:

| Metric | Query |
|---|---|
| HTTP Error Rate | `rate(http_server_requests_seconds_count{status=~"5..",job="ecommerce-app"}[5m])` |
| Response Time P95 | `histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{job="ecommerce-app"}[5m]))` |

If both metrics returned to baseline — CV passes. Pipeline succeeds.

**Failure path — what changes:**

Change Pod Affected Percentage from `50` to `100`. Both pods are deleted. The HTTP probe gets no response — resilience score drops to 0. The Chaos step fails. The failure strategy triggers an automatic rollback.

---

## Prerequisites

Before you start, make sure you have:

- A Harness account with CD, Chaos Engineering, and CV modules enabled
- A Kubernetes cluster with a Harness Delegate installed
- Prometheus running in the cluster (or accessible from it)
- A Docker Hub account
- A GitHub Personal Access Token with `repo` scope

---

## Project Structure

```
cd-tidbits-reliability-testing/
├── .harness/
│   └── pipeline.yaml    — Build + Deploy to Staging + Reliability Validation pipeline
├── k8s/
│   ├── deployment.yaml  — Kubernetes deployment manifest (2 replicas, tuned probes)
│   ├── service.yaml     — Kubernetes service manifest
│   └── values.yaml      — Harness artifact image injection
├── src/                 — Java Spring Boot application source
├── Dockerfile           — Two-stage Docker build
├── pom.xml              — Maven build configuration
└── README.md
```

---

## Step 1 — Create the Connectors

**GitHub Connector**
1. Go to **Project Settings → Connectors → + New Connector → GitHub**
2. Name: `reliability-github-connector`
3. Authentication: **Username and Token** — GitHub username + Personal Access Token with `repo` scope
4. Connectivity Mode: **Connect through a Harness Delegate**
5. Test the connection → **Save**

**Docker Hub Connector**
1. Go to **Project Settings → Connectors → + New Connector → Docker Registry**
2. Name: `reliability-dockerhub-connector`
3. Provider: **DockerHub**, enter your Docker Hub credentials
4. Connectivity Mode: **Connect through a Harness Delegate**
5. Test the connection → **Save**

**Kubernetes Connector**
1. Go to **Project Settings → Connectors → + New Connector → Kubernetes Cluster**
2. Name: `reliability-k8s-connector`
3. Connection: **Use the credentials of a specific Harness Delegate**
4. Select your Delegate
5. Test the connection → **Save**

**Prometheus Connector**
1. Go to **Project Settings → Connectors → + New Connector → Prometheus**
2. Name: `reliability-prometheus-connector`
3. URL: your Prometheus endpoint (e.g. `http://prometheus.monitoring.svc.cluster.local:9090`)
4. Connectivity Mode: **Connect through a Harness Delegate**
5. Test the connection → **Save**

---

## Step 2 — Set Up Chaos Infrastructure

The chaos infrastructure is a Harness agent deployed into your Kubernetes cluster. It's what enables Harness to inject faults directly into the namespace.

1. Go to **Project Settings → Resilience Test Infrastructure → + New Infrastructure**
2. Select environment: **Staging** → click **Continue**
3. Provide infrastructure name: `staging-chaos-infra`
4. Select infrastructure type: **Kubernetes** (under Direct Connection)
5. Provide your Kubernetes cluster connector and set namespace: `cd-chaos-staging`
6. Click **Save**
7. Harness opens the **Onboard New Resilience Testing Infrastructure** window — click **Go** to discover services in your cluster. You can go through the discovery flow or click **Exit** to skip it.
8. Wait for the infrastructure status to show **Connected**

**Create the HTTP Probe**

While still in Project Settings, create the probe before building the experiment:

1. Go to **Project Settings → Chaos Engineering → Probes → + New Probe → HTTP**
2. Name: `app-health-check`
3. URL: `http://ecommerce-app.cd-chaos-staging.svc.cluster.local/`
4. Method: `GET`, Response Code: `200`
5. Timeout: `5s`, Interval: `2s`, Attempt: `3`
6. **Save**

---

## Step 3 — Create the Pod Delete Chaos Experiment

1. Go to **Chaos Experiments under Resilience Testing → Experiments tab → + New Experiment → Create from scratch**
2. Name: `pod-delete-ecommerce`
3. Select infrastructure type: **Kubernetes** → click the **Resilience Test Infrastructure** dropdown
4. Select environment: **Staging** → click **Next** → select `staging-chaos-infra` → click **Apply** → click **Next**
5. In the **Chaos Studio**, click the **+** symbol → **Add Fault → Pod Delete**
   - Target Namespace: `cd-chaos-staging`
   - Target Workload Names: `ecommerce-app`
   - Target Workload Labels: `app=ecommerce-app`
6. **Tune Fault** tab:
   - Total Chaos Duration: `60`
   - Pod Affected Percentage: `50`
7. Click **Apply Changes**
8. Hover over the fault → click **Add Parallel Node** → **Add Probe**
9. Select `app-health-check` → click **Add to Experiment** → click **Apply Changes**
10. **Save** the experiment and copy the **Experiment ID** from the URL — you'll need it in Step 5

---

## Step 4 — Set Up Continuous Verification (CV)

**Create the Harness Service**
1. Go to **Deployments → Services → + New Service**
2. Name: `reliability-ecommerce-service`
3. Add manifest: GitHub connector → repo `cd-tidbits-reliability-testing` → paths `k8s/deployment.yaml`, `k8s/service.yaml` → values `k8s/values.yaml`
4. Add artifact: Docker Hub connector → image `YOUR_DOCKER_USERNAME/ecommerce-reliability-app` → tag: `<+input>`
5. **Save**

**Create the Monitored Service**
1. Go to **Deployments → Service Reliability → Monitored Services → + New Monitored Service**
2. Select service: `reliability-ecommerce-service`, environment: `staging`
3. Add a **Health Source** → **Prometheus** → select `reliability-prometheus-connector`
4. Add two metrics — for each metric configure the following:
   - **HTTP Error Rate**
     - Query: `rate(http_server_requests_seconds_count{status=~"5..",job="ecommerce-app"}[5m])`
     - Assign to: **Continuous Verification**
     - Risk Category: **Errors**
     - Deviation Compared to Baseline: **Higher value is higher risk**
     - Service Instance Identifier: `pod`
   - **Response Time P95**
     - Query: `histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{job="ecommerce-app"}[5m]))`
     - Assign to: **Continuous Verification**
     - Risk Category: **Performance/Response Time**
     - Deviation Compared to Baseline: **Higher value is higher risk**
     - Service Instance Identifier: `pod`
5. **Save**

---

## Step 5 — Import the Pipeline

1. Open `.harness/pipeline.yaml` on GitHub and update the placeholders:
   - `YOUR_PROJECT_ID` → your Harness project identifier
   - `YOUR_ORG_ID` → your Harness org identifier
   - `YOUR_DOCKERHUB_CONNECTOR` → your Docker Hub connector identifier
   - `YOUR_DOCKER_USERNAME` → your Docker Hub username
   - Commit and push
2. Go to **Deployments → Pipelines → Import from Git**
3. Select `reliability-github-connector`, repo: `harness-community/cd-tidbits-reliability-testing`, file: `.harness/pipeline.yaml`
4. Click **Import**
5. Open the pipeline → update the service reference to `reliability-ecommerce-service` in both the Deploy and Reliability Validation stages
6. In the Chaos step → set `experimentRef` to your experiment ID from Step 3
7. Add the **CV step** (AI Verify) after the Chaos step:
   - CV Type: Rolling Update
   - Sensitivity: Medium
   - Duration: 5 min
   - Monitored Service: select your monitored service
8. **Save**

---

## Step 6 — Run the Pipeline (Pass Path)

1. Go to **Deployments → Pipelines → Reliability Testing Pipeline → Run**
2. Watch the execution:
   - **Build** — Docker image built and pushed to Docker Hub tagged with `<+pipeline.sequenceId>`
   - **Deploy to Staging** — approve when prompted; rolling deployment runs, both pods up in `cd-chaos-staging`
   - **Reliability Validation:**
     - Chaos step runs — one pod deleted, second replica keeps serving traffic
     - HTTP probe hits the app every 2 seconds — probe passes, resilience score calculated as passing
     - CV step starts — queries Prometheus for 5 minutes
     - Metrics return to baseline → CV passes
3. Pipeline succeeds

---

## Step 7 — Observe the Failure Path

1. Open the `pod-delete-ecommerce` experiment → **Tune Fault** → change **Pod Affected Percentage** from `50` to `100`
2. Save the experiment
3. Run the pipeline again
4. Both pods are deleted — the app is briefly unreachable
5. HTTP probe fails — no response from either replica
6. Resilience score = 0 → Chaos step fails
7. **Rollback on CV Failure** executes automatically

---

## Common Issues & Tips

**Chaos experiment returns resilience score -1**
- No probe is attached to the experiment. Add the HTTP probe as described in Step 3. Without a probe, Harness cannot calculate a score.

**HTTP probe fails with "no such host"**
- The probe URL uses the wrong namespace. Verify the app is running with `kubectl get svc -n cd-chaos-staging` and confirm the service name. The URL format is `http://<service-name>.<namespace>.svc.cluster.local/`.

**CV step shows "NO ANALYSIS"**
- Prometheus is not returning data for the configured queries. Verify the `job` label in the queries matches what your app exports. Check the Prometheus connector is reachable from the Harness delegate.

**Monitored service not appearing in CV step dropdown**
- The monitored service must be linked to the same service and environment as the pipeline stage. Verify the monitored service uses `reliability-ecommerce-service` + `staging`.

**Pipeline fails with environment not found**
- Create the `staging` environment and `staging_infra` infrastructure definition under it before importing the pipeline.

---

## Resources

- [Use HCE with Continuous Delivery](https://developer.harness.io/resilience-testing/chaos-engineering/integrations/cicd/harness-cd)
- [Continuous Verification overview](https://developer.harness.io/continuous-delivery/use-continuous-delivery/verify-deployments/verify-deployments-with-the-verify-step#why-use-continuous-verification)
- [Chaos Faults — Pod Delete](https://developer.harness.io/resilience-testing/chaos-engineering/new-to-chaos-engineering/quickstart#add-a-chaos-fault)
- [Harness Monitored Services](https://developer.harness.io/docs/service-reliability-management/monitored-service/create-monitored-service)
