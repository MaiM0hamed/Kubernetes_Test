# Multi-Container Pods
## Init Container & Sidecar

### 1. Overview

The objective of this task is to deploy an Nginx Hello page on a **3-node Minikube Kubernetes cluster**.

The page displays a welcome message with the current timestamp, which is automatically updated every 5 seconds by a Sidecar container.

### 2. Kubernetes Cluster

The cluster was created using Minikube with Docker driver.

| Configuration | Value |
|---|---|
| Kubernetes Distribution | Minikube |
| Kubernetes Version | v1.37.0 |
| Driver | Docker |
| Number of Nodes | 3 |
| CPU | 2 |
| Memory | 2048 MB |
| Replicas | 3 |

### 3. Pod Architecture

Each Pod contains three containers:

**1. init-done (Init Container)**
- Executes before the application containers.
- Prints `done` and exits successfully.
- Application containers start only after successful completion.

**2. nginx (Main Container)**
- Uses the official Nginx image.
- Creates the initial `index.html` file with the current timestamp.
- Serves the Hello page on port 80.

**3. time-updater (Sidecar Container)**
- Runs continuously alongside Nginx.
- Updates the HTML page every 5 seconds.
- Uses `/proc/<nginx-pid>/root` to access the Nginx document root.
- Does not require shared Volumes.

### 4. Key Kubernetes Configurations

- `shareProcessNamespace: true`
- No Volumes.
- 3 Deployment replicas.
- RollingUpdate strategy.
- `maxUnavailable: 0`
- Kubernetes Service of type ClusterIP.
- Resource requests and limits configured for containers.

### 5. Deployment Instructions

Apply the Kubernetes manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Check the Deployment status:

```bash
kubectl rollout status deployment/nginx-hello
```

Check Pods:

```bash
kubectl get pods -o wide
```

Check Services:

```bash
kubectl get services
```

### 6. Timestamp Verification

Forward the Service port:

```bash
kubectl port-forward svc/nginx-service 8080:80
```

Access the application:

```bash
curl http://localhost:8080
```

Expected output:

```text
hello in MLOPS MENA COMMUNITY : 2026-11-07 08:00:00
```

The timestamp is updated automatically every 5 seconds.

Check Sidecar logs:

```bash
kubectl logs <POD_NAME> -c time-updater
```

Expected output:

```text
Timestamp updated successfully
Timestamp updated successfully
Timestamp updated successfully
```

### 7. Init Container Failure Simulation

To test Kubernetes Pod lifecycle behavior, the Init Container command was intentionally modified.

Original command:

```yaml
command: ["sh", "-c", "echo done"]
```

Failure command:

```yaml
command: ["sh", "-c", "echo failed; exit 1"]
```

Apply the updated Deployment:

```bash
kubectl apply -f deployment.yaml
```

Monitor the Pods:

```bash
kubectl get pods -w
```

Expected behavior:

- The new Pod fails during initialization.
- Kubernetes repeatedly restarts the Init Container.
- The Pod eventually enters `Init:CrashLoopBackOff`.
- The Nginx and Sidecar containers in the failed Pod do not start.

### 8. Failure Diagnosis

Inspect the failed Pod:

```bash
kubectl describe pod <POD_NAME>
```

Check Init Container logs:

```bash
kubectl logs <POD_NAME> -c init-done
```

Verify the exit code:

```bash
kubectl get pod <POD_NAME> -o jsonpath='{.status.initContainerStatuses[0].lastState.terminated.exitCode}'
```

Expected output:

```text
1
```

### 9. Zero-Downtime Verification

During the failure simulation:

- The three previously running Pods remained healthy.
- The existing Nginx containers continued serving HTTP requests.
- The Service continued responding to requests.
- The failed new Pod did not replace the healthy replicas.

This behavior was supported by the Deployment RollingUpdate configuration:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### 10. Recovery

The original Init Container command was restored:

```yaml
command: ["sh", "-c", "echo done"]
```

Apply the corrected Deployment:

```bash
kubectl apply -f deployment.yaml
```

Wait for the rollout:

```bash
kubectl rollout status deployment/nginx-hello
```

Verify the Pods:

```bash
kubectl get pods -o wide
```

**Final result:** All three replicas successfully returned to `Running` status with `2/2` ready application containers.

### 11. Screenshots

The following screenshots document the implementation and verification process.

#### 1. Kubernetes Cluster Nodes

![Cluster Nodes](screenshots/01-cluster-nodes.png)

#### 2. Running Multi-Container Pods

![Running Pods](screenshots/02-running-pods.png)

#### 3. Timestamp Updates

![Timestamp Updates](screenshots/03-timestamp-update.png)

#### 4. Init Container Failure

![Init Container Failure](screenshots/04-init-failure.png)

#### 5. Init:CrashLoopBackOff

![CrashLoopBackOff](screenshots/05-crashloopbackoff.png)

#### 6. Successful Recovery

![Recovery](screenshots/06-recovery.png)

---

### 12. Project Structure

```text
Task-1/
├── deployment.yaml
├── service.yaml
├── README.md
└── screenshots/
    ├── 01-cluster-nodes.png
    ├── 02-running-pods.png
    ├── 03-timestamp-update.png
    ├── 04-init-failure.png
    ├── 05-crashloopbackoff.png
    └── 06-recovery.png
```

### 13. Conclusion

This task demonstrates practical Kubernetes concepts, including:

- Multi-container Pods.
- Init Containers.
- Sidecar Containers.
- Shared Process Namespace.
- Container communication through `/proc`.
- Deployment Rolling Updates.
- Pod lifecycle and `CrashLoopBackOff`.
- Zero-downtime behavior during failed updates.
- Application recovery.

**Task Status: Completed**
