### 🚀 Kubernetes Application Deployment — MongoDB & Mongo Express

Declarative deployment of a two-component database application on a local Kubernetes cluster, demonstrating configuration management, service discovery, browser access, and operational troubleshooting with Minikube.

---

#### 📌 Project Overview

This project deploys **MongoDB** and **Mongo Express** to a local Kubernetes cluster running through **Minikube and Docker Desktop on Windows**. It demonstrates how Kubernetes Deployments, Services, Secrets, and ConfigMaps work together to run interconnected applications.

The project covers:

- A single-replica MongoDB Deployment and internal Service
- A single-replica Mongo Express Deployment and LoadBalancer Service
- Root database credentials injected from a Kubernetes Secret
- A database connection endpoint defined in a ConfigMap
- Service-to-service communication through Kubernetes DNS
- Local browser access through a Minikube service tunnel
- Verification using `kubectl`, application logs, and the Mongo Express dashboard
- Troubleshooting Kubernetes API connectivity, stale workloads, service exposure, and browser authentication

**Scope:** This is a reproducible **local learning and portfolio deployment**, not a production-ready database platform. The current manifests do not configure persistent storage, resource limits, or health probes.

---

#### 💼 Business Problem

Manually managing a database and its administration interface can create configuration drift, hardcoded credentials, unreliable service addressing, and difficult troubleshooting. Container orchestration helps make application deployment and connectivity more consistent.

This implementation separates **workloads**, **network endpoints**, **non-sensitive configuration**, and **credentials** into declarative Kubernetes resources. It also demonstrates how to validate a deployment from the command line and from the application's user interface.

---

#### ⚙️ DevOps Engineering Value

- **Declarative deployment:** Manage application resources using version-controlled YAML.
- **Configuration separation:** Reference a ConfigMap and Secret instead of placing connection values directly into every Deployment.
- **Service discovery:** Connect Mongo Express to `mongodb-service:27017` rather than a changing Pod IP.
- **Network exposure:** Compare an internal `ClusterIP` Service with a `LoadBalancer` Service accessed locally using Minikube.
- **Operational verification:** Inspect Pods, Services, logs, and browser output.
- **Troubleshooting:** Restore a stopped local cluster and remove obsolete workloads without rebuilding Minikube.

---

#### 🏗️ Architecture

```text
Windows browser
     |
     | http://127.0.0.1:<allocated-local-port>
     v
Minikube service tunnel (terminal remains open)
     |
     v
mongodb-express-service
  type: LoadBalancer
  service port: 8081
  nodePort: 30000
     |
     v
Mongo Express Deployment (1 Pod)
  container port: 8081
  |               |
  |               +--> mongodb-secret
  |                    (database username/password)
  +--> mongodb-configmap
       database_url: mongodb-service:27017
     |
     | Kubernetes internal service discovery
     v
mongodb-service (ClusterIP)
  service port: 27017
     |
     v
MongoDB Deployment (1 Pod)
  container port: 27017
  credentials from mongodb-secret
```

**Separate exercise:** `dashboard-ingress.yaml` targets the Kubernetes Dashboard (`k8s-dashboard.local`) in the `kubernetes-dashboard` namespace. It does **not** route traffic to Mongo Express and is not required for the browser access verified in this project.

---

#### 🛠️ Tech Stack

| Area | Technology |
|---|---|
| Orchestration | Kubernetes |
| Local cluster | Minikube (Docker driver) |
| Container environment | Docker Desktop |
| Database | MongoDB (`mongo` image) |
| Administration UI | Mongo Express (`mongo-express` image) |
| Application configuration | ConfigMap |
| Credential injection | Kubernetes Secret |
| Internal networking | ClusterIP Service and Kubernetes DNS |
| Browser access | LoadBalancer Service via Minikube tunnel |
| Manifests | YAML |
| Tooling | `kubectl`, `minikube`, Windows PowerShell |
| Source control | Git and GitHub |

> The manifests currently use unpinned `mongo` and `mongo-express` image tags. Pin explicit, tested versions or digests to improve reproducibility.

---

#### 📂 Repository Structure

```text
kubernetes/
└── kubernetes-mongodb-mongo-express/
    ├── mongo.yaml
    ├── mongo-secret.yaml
    ├── mongo-configmap.yaml
    ├── mongo-express.yaml
    ├── dashboard-ingress.yaml
    ├── assets/
    │   ├── deployment/
    │   │   └── clean-cluster.png
    │   ├── verification/
    │   │   ├── mongo-express-dashboard.png
    │   │   └── mongo-express-logs-and-tunnel.png
    │   └── troubleshooting/
    └── README.md
```

The `assets/` filenames are **suggested**: add your own screenshots using those names before enabling the image links below. Do not commit a Secret containing real credentials.

---

#### 🔄 Deployment Workflow

#### 1️⃣ Verify the Kubernetes environment

Ensure Docker Desktop is running, then start or resume Minikube:

```powershell
minikube start
kubectl config current-context
kubectl get nodes
kubectl get all
```

For this demonstration, the selected context was `minikube`. After removing unrelated earlier practice Deployments, the default namespace contained only the built-in Kubernetes API Service:

```text
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP
```

The cluster itself was **not deleted**.

<!-- Screenshot to add: assets/deployment/clean-cluster.png -->

#### 2️⃣ Align the namespace configuration before deployment

**Important issue in the uploaded manifests:** `mongo-configmap.yaml` specifies `namespace: my-namespace`, whereas the other manifests do not declare a namespace. A Pod cannot reference a ConfigMap or Secret in a different namespace.

For the simple default-namespace walkthrough, remove the following line from `mongo-configmap.yaml`:

```yaml
namespace: my-namespace
```

Then apply **all four application manifests to the same namespace**. Alternatively, create and consistently use a dedicated namespace for every resource. The commands below assume the default namespace and the corrected ConfigMap.

#### 3️⃣ Create the Secret

`mongo-secret.yaml` defines an `Opaque` Secret named `mongodb-secret` with the keys:

```text
mongo-root-username
mongo-root-password
```

MongoDB reads those keys through `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD`. Mongo Express reads the same keys through `ME_CONFIG_MONGODB_ADMINUSERNAME` and `ME_CONFIG_MONGODB_ADMINPASSWORD`.

```powershell
kubectl apply -f .\mongo-secret.yaml
kubectl get secret mongodb-secret
```

**Security:** The supplied manifest contains Base64-encoded demo credentials. Base64 is not encryption. Replace these values for anything beyond an isolated local demonstration, and never commit real credentials to a public repository. Use a sanitized example manifest for GitHub.

#### 4️⃣ Create the ConfigMap

`mongo-configmap.yaml` defines:

```yaml
data:
  database_url: "mongodb-service:27017"
```

Mongo Express consumes this value as `DATABASE_URL` and constructs its MongoDB connection URL using the credentials provided by the Secret.

```powershell
kubectl apply -f .\mongo-configmap.yaml
kubectl get configmap mongodb-configmap
```

#### 5️⃣ Deploy MongoDB

`mongo.yaml` creates:

- Deployment: `mongodb-deployment`
- Replica count: `1`
- Container image: `mongo`
- Container port: `27017`
- Service: `mongodb-service`, reachable internally on port `27017`

```powershell
kubectl apply -f .\mongo.yaml
kubectl get deployment mongodb-deployment
kubectl get pods -l app=mongodb
kubectl get service mongodb-service
```

The MongoDB Service provides a stable in-cluster name:

```text
mongodb-service:27017
```

This Service is internal; MongoDB is not exposed directly to the browser.

#### 6️⃣ Deploy Mongo Express

`mongo-express.yaml` creates:

- Deployment: `mongo-express`
- Replica count: `1`
- Container image: `mongo-express`
- Container port: `8081`
- Secret-backed database username and password
- ConfigMap-backed database hostname and port
- Service: `mongodb-express-service`
- Service type: `LoadBalancer`, port `8081`, NodePort `30000`

```powershell
kubectl apply -f .\mongo-express.yaml
kubectl get deployment mongo-express
kubectl get pods -l app=mongo-express
kubectl get services
```

The verified Service output included:

```text
NAME                    TYPE           EXTERNAL-IP   PORT(S)
mongodb-express-service LoadBalancer   <pending>     8081:30000/TCP
mongodb-service         ClusterIP      <none>        27017/TCP
```

A pending external IP is expected in this local setup because Minikube is not automatically provisioning a cloud load balancer.

#### 7️⃣ Open Mongo Express in the browser

```powershell
minikube service mongodb-express-service
```

The verified run started a tunnel and opened:

```text
http://127.0.0.1:52069
```

The `52069` port was dynamically allocated for that run; it may differ next time. With Minikube's Docker driver on Windows, keep the terminal running while using the tunnel.

The browser prompted for Mongo Express **HTTP Basic Authentication**. In the observed container logs, the demo credentials were reported as `admin:pass`. These are separate from the MongoDB root credentials supplied by the Kubernetes Secret. Change default HTTP credentials before exposing the interface beyond local testing.

---

#### ✅ Application Verification

#### Kubernetes resource inspection

```powershell
kubectl get pods
kubectl get deployments
kubectl get services
```

The MongoDB Pod was observed in `Running` state, and the Mongo Express logs showed that the application started and listened on port `8081`.

#### Application log inspection

```powershell
kubectl logs deployment/mongo-express
```

Relevant observed log messages:

```text
Waiting for mongodb-service:27017...
Welcome to mongo-express 1.0.2
Mongo Express server listening at http://0.0.0.0:8081
```

The logs also warned that default Basic Auth credentials were enabled, providing an actionable security finding.

<!-- Screenshot to add: assets/verification/mongo-express-logs-and-tunnel.png -->

#### Browser verification

The Mongo Express dashboard loaded successfully and displayed the databases:

```text
Test-db
admin
config
local
```

The dashboard also displayed MongoDB server information, confirming that the administration interface could communicate with the database.

<!-- Screenshot to add: assets/verification/mongo-express-dashboard.png -->

**Result:** The application was accessible from a Windows browser via Minikube, and the Mongo Express interface successfully displayed MongoDB databases.

---

#### 🧪 Operational Troubleshooting

#### ❌ Challenge 1 — Kubernetes API connection refused

**Symptom:**

```text
Unable to connect to the server:
dial tcp 127.0.0.1:<port>:
No connection could be made because the target machine actively refused it.
```

**Investigation:** The kubeconfig referenced a local Minikube API endpoint, but the cluster was not reachable.

**Resolution:** Started the existing Minikube cluster after ensuring Docker Desktop was available:

```powershell
minikube status
minikube start
kubectl get nodes
```

**Engineering insight:** Verify cluster state and context before assuming a YAML or application defect. Restarting the existing cluster can restore access without destroying deployed resources.

#### ❌ Challenge 2 — Old workloads obscured a clean starting point

**Symptom:** `kubectl get all` showed earlier `my-app` and `my-app-two` Deployments with `ImagePullBackOff` statuses.

**Resolution:** After confirming the `minikube` context, removed only the unrelated practice Deployments:

```powershell
kubectl config current-context
kubectl delete deployment my-app my-app-two -n default
kubectl get all
```

**Engineering insight:** Targeted resource cleanup is safer than deleting an entire cluster. The observed `ImagePullBackOff` status was not necessary to resolve for this MongoDB demo.

#### ❌ Challenge 3 — LoadBalancer external IP remained pending

**Symptom:** `mongodb-express-service` showed `EXTERNAL-IP <pending>`.

**Cause:** The local Minikube environment did not provision a cloud load balancer for this Service.

**Resolution:** Used `minikube service mongodb-express-service` to create a local tunnel and open the web interface.

**Engineering insight:** A Service's Kubernetes type and its externally reachable address depend on the cluster's networking implementation.

#### ❌ Challenge 4 — Browser requested credentials

**Symptom:** A browser authentication dialog appeared when opening Mongo Express.

**Investigation:** `kubectl logs deployment/mongo-express` revealed the image's active HTTP Basic Auth credentials in the observed run.

**Resolution:** Used the demo browser credentials to reach the dashboard and verify database connectivity.

**Engineering insight:** Distinguish **Mongo Express browser authentication** from **MongoDB database authentication**. Default application credentials should be replaced before broader exposure.

#### ⚠️ Configuration review — ConfigMap namespace mismatch

**Finding:** The uploaded ConfigMap explicitly uses `my-namespace`, but the other application manifests omit `metadata.namespace`.

**Required correction:** Place the ConfigMap, Secret, and Deployments in the same namespace. For the default-namespace instructions in this README, remove the ConfigMap's `namespace: my-namespace` field before applying it.

**Status:** This is a configuration issue identified during manifest review, **not an observed troubleshooting incident** from the provided screenshots.

---

#### 🔐 Security and Reliability Limitations

The current implementation is a local demonstration and has several important limitations:

| Area | Current state | Recommended improvement |
|---|---|---|
| Database credentials | Base64-encoded demo values in a Secret manifest | Sanitize GitHub examples; manage real secrets securely |
| Browser authentication | Default Mongo Express credentials observed in logs | Configure unique credentials and restrict access |
| Image versions | Unpinned `mongo` and `mongo-express` tags | Pin tested versions or digests |
| Persistence | No PersistentVolumeClaim in the MongoDB manifest | Add durable database storage |
| Availability | One replica per Deployment | Design availability appropriate to workload and database topology |
| Health checks | No explicit readiness/liveness probes | Add appropriate probes |
| Resource management | No CPU/memory requests or limits | Set and test resource budgets |
| Network isolation | No NetworkPolicies shown | Restrict service-to-service access |
| Ingress | Dashboard Ingress is separate from Mongo Express | Configure a dedicated app Ingress if needed |

**Database persistence warning:** Without a persistent volume, database data may be lost if the MongoDB container/Pod is recreated or replaced. Do not treat this deployment as durable storage.

---

#### 📈 DevOps Concepts Demonstrated

Kubernetes Deployments and Pods · Declarative YAML · Secrets · ConfigMaps · Service discovery · ClusterIP networking · LoadBalancer and NodePort behavior · Minikube tunnelling · Application log inspection · Cluster recovery · Targeted resource cleanup · Authentication troubleshooting · Deployment verification

---

#### 🔮 Future Improvements

1. Correct and standardize namespace declarations across all manifests.
2. Replace demo credentials with securely managed values and provide a sanitized example Secret.
3. Pin MongoDB and Mongo Express image versions.
4. Add a PersistentVolumeClaim and evaluate a StatefulSet for database management.
5. Add health probes, resource requests, and limits.
6. Configure dedicated application Ingress and appropriate access controls.
7. Add automated manifest validation and deployment smoke tests.
8. Package the application as a Helm chart.
9. Introduce monitoring, logging, and CI/CD integration.

---

#### 🔗 Related Projects

- **Java Maven CI/CD Pipeline:** https://github.com/migi-devops/java-maven-app
- **AWS DevOps Deployment Pipeline:** https://github.com/migi-devops/aws-devops-pipeline

These projects complement the Kubernetes work by covering build automation, container image management, and cloud deployment workflows.

---

#### 👨‍💻 Author

**Miguel — DevOps Engineer**

Building practical DevOps projects focused on CI/CD automation, cloud infrastructure, containerization, Kubernetes orchestration, and operational troubleshooting.

---

#### 📚 Learning Source

Built while following the **TechWorld with Nana DevOps Bootcamp**. This portfolio documentation reflects the actual manifests, local deployment verification, and troubleshooting performed during the exercise.
