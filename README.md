# Installing and Configuring Argo CD on Kubernetes | GitOps Lab

## 1. Overview

In this lab, I installed Argo CD on a two-node Kubernetes cluster using the official Argo CD installation manifest and configured it to deploy an Nginx application from a GitHub repository.

Argo CD is a declarative GitOps continuous delivery tool for Kubernetes. It continuously monitors the desired application state stored in a Git repository and compares it with the actual state running inside the Kubernetes cluster.

When changes are committed to Git, Argo CD can automatically synchronize those changes with the Kubernetes cluster.

### Lab environment

| Component | Configuration |
|---|---|
| Kubernetes cluster | kubeadm |
| Control-plane node | `k8s` |
| Control-plane IP | `192.168.64.4` |
| Worker node | `workernode2` |
| Worker IP | `192.168.64.5` |
| Operating system | Ubuntu 26.10 ARM64 |
| Container runtime | containerd |
| CNI | Calico |
| Argo CD namespace | `argocd` |
| Installation method | Official stable manifest |
| Git repository | GitHub |
| Application | Nginx |
| Application namespace | `nginx-demo` |

## 2. Prerequisites

Before installing Argo CD, ensure:

- Kubernetes cluster is initialized and operational.
- `kubectl` is installed and configured.
- All nodes are in Ready state.
- CoreDNS is installed and running.
- Cluster networking is functional.

Check the cluster:

```bash
kubectl get nodes -o wide
```

Check system pods:

```bash
kubectl get pods -n kube-system -o wide
```

## 3. Create the Argo CD namespace

We created a dedicated namespace to isolate Argo CD components from application workloads.

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get namespaces
```

Why a separate namespace?

- Keeps Argo CD components logically isolated.
- Makes resource management easier.
- Allows namespace-level monitoring and troubleshooting.
- Provides a dedicated location for Argo CD secrets and configuration.

## 4. Install Argo CD

We used the official Argo CD installation manifest from the stable release channel.

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This manifest installs the Argo CD components and their required Kubernetes resources, including:

- Deployments
- Services
- ConfigMaps
- Secrets
- ServiceAccounts
- RBAC resources
- Custom Resource Definitions (CRDs)

The standard installation includes the components required for the Argo CD UI, API, repository access, and application reconciliation.

**Note:** The `stable` URL follows the current stable release. For reproducible production deployments, pin the installation to a specific Argo CD version.

## 5. Verify the installation

Check all Argo CD pods:

```bash
kubectl get pods -n argocd -o wide
```

Check deployments:

```bash
kubectl get deployments -n argocd
```

Check services:

```bash
kubectl get services -n argocd
```

Check all resources:

```bash
kubectl get all -n argocd
```

In our lab, all seven Argo CD pods reached the Running state with containers Ready.

### Important Argo CD components

| Component | Responsibility |
|---|---|
| argocd-server | API server and web UI |
| argocd-repo-server | Clones Git repositories and generates application manifests |
| argocd-application-controller | Monitors applications and reconciles desired state |
| argocd-dex-server | Identity integration |
| argocd-redis | Caching |
| argocd-applicationset-controller | Generates and manages applications from ApplicationSet resources |
| argocd-notifications-controller | Application event notifications |

## 6. Access the Argo CD web UI

By default, Argo CD's server is not exposed externally through a LoadBalancer or Ingress.

For our local Kubernetes lab, we used `kubectl port-forward`.

Run:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443 \
  --address 0.0.0.0
```

Explanation:

- `svc/argocd-server`: Targets the Argo CD server Service.
- `-n argocd`: Selects the Argo CD namespace.
- `8080:443`: Maps port 8080 on the VM to HTTPS port 443 on the Service.
- `--address 0.0.0.0`: Allows connections to the forwarded port through the VM's network interfaces.

Keep this terminal running while accessing the UI.

Open the following URL from the Mac:

```text
https://192.168.64.4:8080
```

The default installation uses a self-signed TLS certificate, so the browser may display a certificate warning. This is expected for the initial lab setup.

**Security note:** Binding to `0.0.0.0` exposes the forwarded port on all VM interfaces. Use this only in a controlled lab network. For a more restricted setup, bind to `127.0.0.1` and access through an appropriate tunnel.

## 7. Retrieve the initial admin password

Argo CD generates an initial password for its `admin` account and stores it in a Kubernetes Secret.

Retrieve it using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Login credentials:

```text
Username: admin
Password: Output of the previous command
```

Enter these credentials in the Argo CD web UI.

**Security recommendation:** Change the initial admin password after the first login. Once the password has been changed, delete the initial password Secret:

```bash
kubectl delete secret argocd-initial-admin-secret -n argocd
```

## 8. Connect Argo CD to a Git repository

After installation, the next step is to connect Argo CD to a Git repository containing Kubernetes manifests.

For our lab, we used:

**GitHub repository:**

https://github.com/mhasif0786/mhasifkamal-infra.git

The repository contains the desired Kubernetes configuration.

Example repository structure:

```text
mhasifkamal-infra/
│
├── apps/
│   └── nginx-demo/
│       ├── deployment.yaml
│       └── service.yaml
│
└── README.md
```

The application directory contains the Kubernetes manifests that Argo CD monitors.

## 9. Create the Nginx Kubernetes manifests

### 9.1 Deployment manifest

File: `apps/nginx-demo/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx
  namespace: nginx-demo

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27

          ports:
            - containerPort: 80

          resources:
            requests:
              cpu: 100m
              memory: 64Mi

            limits:
              cpu: 500m
              memory: 128Mi
```

### 9.2 Service manifest

File: `apps/nginx-demo/service.yaml`

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx
  namespace: nginx-demo

spec:
  type: ClusterIP

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

The Deployment maintains two Nginx pod replicas.

The Service provides a stable internal endpoint to access the Nginx pods.

Commit and push the manifests to GitHub before configuring the Argo CD application.

## 10. Create an Argo CD Application

We created an application named `nginx-demo`.

The application configuration was:

| Setting | Value |
|---|---|
| Application name | nginx-demo |
| Project | default |
| Repository URL | `https://github.com/mhasif0786/mhasifkamal-infra.git` |
| Target revision | HEAD |
| Path | `apps/nginx-demo` |
| Destination cluster | `https://kubernetes.default.svc` |
| Destination namespace | nginx-demo |
| Sync policy | Automatic |
| Auto-create namespace | Enabled |

### Understanding the configuration

**Repository URL:** The Git repository containing the desired Kubernetes state.

**Target revision:** `HEAD` tells Argo CD to track the repository's current HEAD revision.

**Path:** Identifies the directory containing Kubernetes manifests.

**Destination cluster:** `https://kubernetes.default.svc` refers to the Kubernetes cluster where Argo CD is running.

**Destination namespace:** The namespace where the application resources should be deployed.

**Automatic synchronization:** Argo CD can automatically reconcile changes in Git with the cluster.

**Auto-create namespace:** Allows Argo CD to create the destination namespace if it does not already exist.

We used the existing `default` AppProject. The attempted custom project name `Portfolio-project` was not present in the cluster.

## 11. Troubleshooting: Argo CD could not access GitHub

During application creation, we encountered:

```text
InvalidSpecError: repository not accessible

unable to ls-remote HEAD

Get "https://github.com/.../info/refs?service=git-upload-pack":

dial tcp: lookup github.com on 10.96.0.10:53:
server misbehaving
```

This indicated a DNS resolution failure rather than a Git authentication problem.

### 11.1 Check CoreDNS pods

```bash
kubectl get pods -n kube-system -o wide
```

Both CoreDNS pods were Running.

However, Running status alone does not guarantee successful DNS resolution.

### 11.2 Inspect the CoreDNS configuration

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

We found:

```text
forward . /etc/resolv.conf {
    max_concurrent 1000
}
```

This forwards external DNS queries to the upstream resolvers listed in `/etc/resolv.conf`.

### 11.3 Inspect the node DNS configuration

```bash
resolvectl status
```

The node had these upstream DNS servers:

```text
192.168.64.1
fe80::8047:15ff:feae:f164%2
```

CoreDNS logs showed timeouts when forwarding queries to the IPv4 resolver and an error when using the IPv6 link-local resolver.

### 11.4 Compare DNS resolution from the host and pod

From the Kubernetes node:

```bash
nslookup github.com 192.168.64.1
```

This succeeded.

We then created a temporary diagnostic pod:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  --command -- sh
```

Inside the pod, we checked:

```sh
cat /etc/resolv.conf
```

The pod used:

```text
nameserver 10.96.0.10
```

Internal DNS resolution worked:

```sh
nslookup kubernetes.default.svc.cluster.local
```

But the direct query to the gateway DNS server failed:

```sh
nslookup github.com 192.168.64.1
```

We tested alternative upstream resolvers:

```sh
nslookup github.com 1.1.1.1
nslookup github.com 8.8.8.8
```

Both succeeded from the pod.

### Root cause

The upstream resolvers inherited by CoreDNS were not usable from the CoreDNS pod network for external DNS queries.

The host could resolve external domains, but the CoreDNS pods could not successfully use the same upstream resolvers.

### 11.5 Correct the CoreDNS forwarding configuration

We updated the CoreDNS ConfigMap.

Previous configuration:

```text
forward . /etc/resolv.conf {
    max_concurrent 1000
}
```

Updated configuration:

```text
forward . 1.1.1.1 8.8.8.8 {
    max_concurrent 1000
}
```

**Why did we change it?**

The original configuration dynamically reads upstream DNS servers from `/etc/resolv.conf`. However, the inherited resolver configuration was not working from the CoreDNS pod network.

We verified that `1.1.1.1` and `8.8.8.8` were reachable and returned valid DNS responses from a Kubernetes diagnostic pod.

We therefore configured CoreDNS to use these verified upstream resolvers directly.

### 11.6 Restart CoreDNS

After updating the ConfigMap, we restarted the CoreDNS deployment:

```bash
kubectl rollout restart deployment/coredns -n kube-system
```

Wait for the rollout:

```bash
kubectl rollout status deployment/coredns -n kube-system
```

Check the pods:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

### 11.7 Validate DNS resolution

We used a fresh diagnostic pod to validate external and internal DNS resolution.

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  --command -- sh
```

Inside the pod:

External DNS:

```sh
nslookup github.com
```

Internal Kubernetes DNS:

```sh
nslookup kubernetes.default.svc.cluster.local
```

Both internal and external DNS resolution should work.

### Resolution

The CoreDNS forwarding configuration was corrected to use reachable upstream DNS servers rather than the inherited resolver configuration.

The DNS configuration correction was applied, and CoreDNS was restarted.

Following this correction, Argo CD successfully accessed the GitHub repository and synchronized the application.

## 12. Verify Argo CD repository connectivity and application status

After correcting DNS, the `nginx-demo` application successfully synchronized.

We verified the application status in the Argo CD UI.

### Argo CD dashboard result

| Property | Observed status |
|---|---|
| Application | nginx-demo |
| Project | default |
| Sync status | Synced |
| Health status | Healthy |
| Operation status | Sync OK |
| Repository | `https://github.com/mhasif0786/mhasifkamal-infra.git` |
| Target revision | HEAD |
| Manifest path | `apps/nginx-demo` |
| Destination | in-cluster |
| Namespace | nginx-demo |

The dashboard showed:

- Synced: 1
- Healthy: 1
- Sync OK: 1
- OutOfSync: 0
- Degraded: 0
- Sync errors: 0

### What this confirms

- Argo CD successfully accessed the Git repository.
- Repository manifest retrieval and application processing succeeded.
- The application synchronization operation completed successfully.
- Argo CD reports the application as Healthy and Synced.
- The previous external DNS resolution issue no longer prevents this GitOps workflow.

<img width="1016" height="543" alt="image" src="https://github.com/user-attachments/assets/71129e90-6e6e-44ae-b36a-bbda70c71112" />
<img width="1024" height="568" alt="image" src="https://github.com/user-attachments/assets/91a5142a-519b-4932-9118-b3f96617d1ce" />
<img width="1006" height="222" alt="image" src="https://github.com/user-attachments/assets/ad7c77f8-96ce-4c58-9586-c850cfba2c6f" />

Useful commands:

```bash
kubectl get applications -n argocd
```

```bash
kubectl describe application nginx-demo -n argocd
```

If repository connectivity problems occur again:

```bash
kubectl logs deployment/argocd-repo-server -n argocd
```

## 13. Verify the Nginx application in Kubernetes

After successful synchronization, we verified the actual Kubernetes resources from the control-plane node.

### 13.1 Check all application resources

```bash
kubectl get all -n nginx-demo -o wide
```

Actual output:

```text
NAME                         READY   STATUS    RESTARTS   AGE   IP                NODE
pod/nginx-68d7965994-8xdrh   1/1     Running   0          19m   192.168.216.82   workernode2
pod/nginx-68d7965994-j5xf5   1/1     Running   0          19m   192.168.216.83   workernode2

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
service/nginx   ClusterIP   10.110.116.36   <none>        80/TCP

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   2/2     2            2           19m

NAME                               DESIRED   CURRENT   READY
replicaset.apps/nginx-68d7965994   2         2         2
```

### 13.2 Deployment verification

```bash
kubectl get deployment nginx -n nginx-demo
```

Observed state:

```text
NAME    READY   UP-TO-DATE   AVAILABLE
nginx   2/2     2            2
```

The Deployment successfully maintains two replicas.

- Desired replicas: 2
- Current replicas: 2
- Ready replicas: 2
- Available replicas: 2

### 13.3 Pod verification

Observed pods:

```text
NAME                     READY   STATUS    RESTARTS   AGE   IP                NODE
nginx-68d7965994-8xdrh   1/1     Running   0          19m   192.168.216.82   workernode2
nginx-68d7965994-j5xf5   1/1     Running   0          19m   192.168.216.83   workernode2
```

Both Nginx pods are:

- Running and Ready (`1/1`).
- Running without container restarts.
- Assigned unique Pod IP addresses.
- Scheduled on the `workernode2` worker node.

### 13.4 Service verification

Observed Service:

```text
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
nginx   ClusterIP   10.110.116.36   <none>        80/TCP
```

Service configuration:

- Service name: `nginx`
- Type: `ClusterIP`
- Cluster IP: `10.110.116.36`
- Service port: `80`
- Selector: `app=nginx`

The Service provides a stable internal endpoint for accessing the Nginx pods.

### 13.5 ReplicaSet verification

Observed ReplicaSet:

```text
NAME                     DESIRED   CURRENT   READY
nginx-68d7965994          2         2         2
```

The ReplicaSet is responsible for maintaining the desired number of Nginx pod replicas.

### 13.6 Understanding the request flow

```text
                 Client / Application
                         |
                         |
                         v
                Nginx ClusterIP Service
                  10.110.116.36:80
                         |
                 Service Selector
                    app=nginx
                         |
                 +-------+-------+
                 |               |
                 v               v
           Nginx Pod 1      Nginx Pod 2
           192.168.216.82   192.168.216.83
                 |               |
                 +-------+-------+
                         |
                     workernode2
```

The Kubernetes Service selects pods using the label `app=nginx` and makes them available through a stable ClusterIP endpoint.

## 14. Additional Kubernetes verification commands

Check the Deployment:

```bash
kubectl get deployment nginx -n nginx-demo
```

Check pods and their node placement:

```bash
kubectl get pods -n nginx-demo -o wide
```

Check the Service:

```bash
kubectl get svc nginx -n nginx-demo
```

Check Service endpoints:

```bash
kubectl get endpoints nginx -n nginx-demo
```

Check EndpointSlices:

```bash
kubectl get endpointslices -n nginx-demo
```

Check rollout status:

```bash
kubectl rollout status deployment/nginx -n nginx-demo
```

## 15. Understanding the GitOps workflow

The complete workflow in this lab:

```text
Developer
    |
    | Changes Kubernetes YAML
    v
GitHub Repository
    |
    | New commit
    v
Argo CD Repo Server
    |
    | Fetches Git repository
    | Reads Kubernetes manifests
    v
Argo CD Application Controller
    |
    | Compares desired state with actual state
    | Detects differences
    v
Kubernetes API Server
    |
    | Applies desired resources
    v
nginx-demo Namespace
    |
    +---- Nginx Deployment
    |
    +---- Nginx ReplicaSet
    |
    +---- Nginx Pods (2 replicas)
    |
    +---- Nginx ClusterIP Service
```

### Desired state vs actual state

**Desired state:** The configuration stored in Git.

**Actual state:** The resources currently running in Kubernetes.

Argo CD continuously compares these two states and reports whether the application is Synced or OutOfSync.

When automated synchronization is configured, Argo CD can apply changes from Git to bring the cluster back toward the desired state.

## 16. Current Lab Status

| Component | Status |
|---|---|
| Kubernetes cluster | Operational |
| Argo CD installation | Successful |
| Argo CD UI | Accessible |
| GitHub repository connection | Successful |
| CoreDNS external forwarding correction | Applied |
| Argo CD application sync | Synced |
| Application health | Healthy |
| Nginx Deployment | 2/2 Available |
| Nginx pods | 2/2 Running |
| Nginx Service | ClusterIP created |
| Nginx ReplicaSet | 2/2 Ready |

**Result:** Successfully deployed a two-replica Nginx application using Argo CD and a GitHub-hosted Kubernetes manifest repository. Verified the Deployment, ReplicaSet, pods, and Service directly through Kubernetes.

## 17. Next Steps and Experiments

The next experiments will focus on understanding GitOps reconciliation and Kubernetes production behavior.

### 17.1 Git-driven scaling

Change the replica count in Git from two to three:

```yaml
spec:
  replicas: 3
```

Commit and push:

```bash
git add .
git commit -m "Increase nginx replicas to 3"
git push origin main
```

Observe Argo CD detecting the Git change and synchronizing the updated desired state.

Verify:

```bash
kubectl get deployment nginx -n nginx-demo

kubectl get pods -n nginx-demo
```

Expected deployment state:

```text
nginx   3/3
```

### 17.2 Drift detection

Manually change the live Kubernetes state:

```bash
kubectl scale deployment nginx \
  --replicas=2 \
  -n nginx-demo
```

If Git still declares three replicas, observe how Argo CD detects the difference.

This experiment helps distinguish desired state from actual state.

### 17.3 Self-healing

Enable and test automated self-healing through Argo CD.

Observe whether manual changes to the live cluster are automatically reverted to match the Git repository.

### 17.4 Service connectivity

Test Nginx through its ClusterIP from a temporary test pod.

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  --rm -it \
  --command -- sh
```

Inside the pod:

```sh
curl http://nginx.nginx-demo.svc.cluster.local
```

Expected result: Nginx welcome page HTML.

### 17.5 Rolling updates

Change the Nginx image version in Git and observe the Deployment rollout.

```bash
kubectl rollout status deployment/nginx -n nginx-demo
```

Inspect rollout history:

```bash
kubectl rollout history deployment/nginx -n nginx-demo
```

## 18. Useful Troubleshooting Commands

Check Argo CD pods:

```bash
kubectl get pods -n argocd -o wide
```

Check Argo CD services:

```bash
kubectl get svc -n argocd
```

Check Argo CD deployments:

```bash
kubectl get deployments -n argocd
```

Inspect a specific pod:

```bash
kubectl describe pod <POD_NAME> -n argocd
```

View repository server logs:

```bash
kubectl logs deployment/argocd-repo-server -n argocd
```

View application controller logs:

```bash
kubectl logs deployment/argocd-application-controller -n argocd
```

View CoreDNS logs:

```bash
kubectl logs -n kube-system \
  -l k8s-app=kube-dns \
  --tail=100
```

Check applications:

```bash
kubectl get applications -n argocd
```

Check Nginx resources:

```bash
kubectl get all -n nginx-demo -o wide
```

## 19. References

- Official Argo CD Getting Started Guide: https://argo-cd.readthedocs.io/en/latest/getting_started/
- Official Argo CD Installation Documentation: https://argo-cd.readthedocs.io/en/latest/operator-manual/installation/
- Official CoreDNS Documentation: https://coredns.io/manual/toc/
- Kubernetes Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes Services: https://kubernetes.io/docs/concepts/services-networking/service/
