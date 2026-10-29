# OKE Kubernetes Deployments Lab

**Deployments, OCIR, labels, OSOK, load balancers, ingress, Arm and GPU nodes**

This lab assumes that an OKE cluster already exists. It covers application deployment, OCIR image pulls, labels, OSOK, LoadBalancer services, the OCI Native Ingress Controller, Arm nodes and GPU nodes.

## Contents

| Lab | Topic |
|-----|-------|
| 1 | Managing Kubernetes deployments |
| 2 | Multi-tier application with kubectl |
| 3 | Pulling images from OCIR |
| 4 | Supported Kubernetes labels |
| 5 | Installing the OCI Service Operator for Kubernetes (OSOK) |
| 6 | Provisioning an OCI MySQL DB System from Kubernetes |
| 7 | Service of type LoadBalancer |
| 8 | OCI Native Ingress Controller |
| 9 | Applications on Arm-based nodes |
| 10 | Applications on GPU nodes |
| 11 | Validation checklist |
| 12 | Cleanup |

---

## 1. Lab Prerequisites

### Required tools

Install and configure:

- OCI CLI
- kubectl
- Helm
- Docker or Podman
- Access to the existing OKE cluster

Configure the OKE kubeconfig:

```bash
oci ce cluster create-kubeconfig \
  --cluster-id <OKE_CLUSTER_OCID> \
  --file $HOME/.kube/config \
  --region <OCI_REGION> \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT
```

Verify access:

```bash
kubectl get nodes -o wide
kubectl get namespaces
```

Set environment variables:

```bash
export REGION=<OCI_REGION>
export TENANCY_OCID=<TENANCY_OCID>
export COMPARTMENT_OCID=<COMPARTMENT_OCID>
export CLUSTER_OCID=<OKE_CLUSTER_OCID>
export NAMESPACE=demo
```

Create a lab namespace:

```bash
kubectl create namespace $NAMESPACE
```

---

## Lab 1: Managing Kubernetes Deployments

### Objective

Deploy, scale, update and roll back a Kubernetes application.

Create `web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
  labels:
    app.kubernetes.io/name: web
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: web
  template:
    metadata:
      labels:
        app.kubernetes.io/name: web
        app.kubernetes.io/component: frontend
        app.kubernetes.io/part-of: demo-app
    spec:
      containers:
        - name: web
          image: docker.io/library/nginx:1.25
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
```

Deploy the application:

```bash
kubectl apply -f web-deployment.yaml
kubectl rollout status deployment/web -n $NAMESPACE
```

Inspect the deployment:

```bash
kubectl get deployments -n $NAMESPACE
kubectl get pods -n $NAMESPACE -o wide
kubectl describe deployment web -n $NAMESPACE
```

Scale it:

```bash
kubectl scale deployment web \
  --replicas=5 \
  --namespace $NAMESPACE
```

Update the image:

```bash
kubectl set image deployment/web \
  web=docker.io/library/nginx:1.26 \
  --namespace $NAMESPACE
```

Check rollout history:

```bash
kubectl rollout history deployment/web -n $NAMESPACE
```

Roll back:

```bash
kubectl rollout undo deployment/web -n $NAMESPACE
kubectl rollout status deployment/web -n $NAMESPACE
```

---

## Lab 2: Deploying a Multi-Tier Application with kubectl

### Objective

Deploy frontend, backend and database tiers using Kubernetes manifests.

Create `multi-tier.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: demo
  labels:
    app.kubernetes.io/name: backend
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: multi-tier-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: backend
  template:
    metadata:
      labels:
        app.kubernetes.io/name: backend
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: multi-tier-app
    spec:
      containers:
        - name: backend
          image: docker.io/hashicorp/http-echo:1.0
          args:
            - "-text=Response from backend"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: demo
spec:
  selector:
    app.kubernetes.io/name: backend
  ports:
    - port: 8080
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: demo
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: multi-tier-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: frontend
  template:
    metadata:
      labels:
        app.kubernetes.io/name: frontend
        app.kubernetes.io/component: frontend
        app.kubernetes.io/part-of: multi-tier-app
    spec:
      containers:
        - name: frontend
          image: docker.io/library/nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: demo
spec:
  selector:
    app.kubernetes.io/name: frontend
  ports:
    - port: 80
      targetPort: 80
```

Apply the resources:

```bash
kubectl apply -f multi-tier.yaml
kubectl get all -n $NAMESPACE
```

Test backend connectivity from a temporary pod:

```bash
kubectl run curl \
  --rm -it \
  --image=docker.io/curlimages/curl \
  --restart=Never \
  -n $NAMESPACE \
  -- curl http://backend:8080
```

View service discovery:

```bash
kubectl get svc -n $NAMESPACE
kubectl get endpoints -n $NAMESPACE
```

---

## Lab 3: Pulling Images from OCIR During Deployment

### Objective

Deploy a private image stored in Oracle Cloud Infrastructure Registry (OCIR).

### 3.1 Create an OCIR repository

```bash
oci artifacts container repository create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name oke-lab \
  --is-public false
```

The OCIR registry endpoint has this format:

```text
<region-key>.ocir.io
```

For example:

```text
bom.ocir.io
iad.ocir.io
```

Obtain the region key for your OCI region from the OCIR documentation or Console.

### 3.2 Log in to OCIR

Create an OCI Auth Token in the Console:

1. Open your user profile.
2. Select **My Profile**.
3. Open **Auth Tokens**.
4. Generate an auth token.
5. Copy it immediately.

Log in:

```bash
docker login <region-key>.ocir.io \
  -u '<tenancy-namespace>/<oci-username>' \
  -p '<auth-token>'
```

### 3.3 Tag and push an image

```bash
export OCIR_REGISTRY=<region-key>.ocir.io
export OCIR_NAMESPACE=<tenancy_namespace>
export IMAGE_NAME=oke-lab/web
export IMAGE_VERSION=1.0

docker pull docker.io/library/nginx:1.25

docker tag docker.io/library/nginx:1.25 \
  $OCIR_REGISTRY/$OCIR_NAMESPACE/$IMAGE_NAME:$IMAGE_VERSION

docker push \
  $OCIR_REGISTRY/$OCIR_NAMESPACE/$IMAGE_NAME:$IMAGE_VERSION
```

### 3.4 Create an image pull secret

```bash
kubectl create secret docker-registry ocir-secret \
  --namespace $NAMESPACE \
  --docker-server=$OCIR_REGISTRY \
  --docker-username="$OCIR_NAMESPACE/<oci-username>" \
  --docker-password="<auth-token>" \
  --docker-email="<email-address>"
```

Deploy the private image (`private-web.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-web
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: private-web
  template:
    metadata:
      labels:
        app: private-web
    spec:
      imagePullSecrets:
        - name: ocir-secret
      containers:
        - name: web
          image: <region-key>.ocir.io/<tenancy-namespace>/oke-lab/web:1.0
          ports:
            - containerPort: 80
```

Apply and verify:

```bash
kubectl apply -f private-web.yaml
kubectl rollout status deployment/private-web -n $NAMESPACE
kubectl get pods -n $NAMESPACE
```

If the image cannot be pulled:

```bash
kubectl describe pod <POD_NAME> -n $NAMESPACE
```

Check for:

- Incorrect OCIR region key
- Incorrect tenancy namespace
- Expired auth token
- Missing `imagePullSecrets`
- Repository permissions
- Image name or tag mismatch

---

## Lab 4: Supported Kubernetes Labels

Use Kubernetes recommended labels consistently.

```yaml
metadata:
  labels:
    app.kubernetes.io/name: payments
    app.kubernetes.io/instance: payments-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/managed-by: kubectl
    environment: production
    team: platform
```

### Label use cases

**Application selection**

```bash
kubectl get pods -l app.kubernetes.io/name=frontend -n $NAMESPACE
```

**Environment filtering**

```bash
kubectl get all \
  -l environment=production \
  -n $NAMESPACE
```

**Scheduling**

```yaml
spec:
  template:
    metadata:
      labels:
        workload: arm64
```

**Cost and ownership**

```yaml
metadata:
  labels:
    cost-center: engineering
    owner: platform-team
```

**Network policies**

Network policies can select pods using labels:

```yaml
podSelector:
  matchLabels:
    app.kubernetes.io/component: backend
```

---

## Lab 5: Installing the OCI Service Operator for Kubernetes

### Objective

Use Kubernetes resources to provision OCI services.

The OCI Service Operator for Kubernetes (OSOK) requires:

- An OKE cluster
- Kubernetes access
- OCI IAM permissions
- A Kubernetes service account
- Workload identity or another supported OCI authentication method
- Helm

Add the Oracle Helm repository:

```bash
helm repo add oracle https://oracle.github.io/oci-service-operator
helm repo update
```

Check available charts:

```bash
helm search repo oracle
```

Install OSOK using the version and values supported by the current OSOK release:

```bash
helm install osok \
  <OSOK_CHART_NAME> \
  --namespace osok-system \
  --create-namespace
```

Verify the operator:

```bash
kubectl get pods -n osok-system
kubectl get crds | grep -i oracle
```

Inspect the installed CRDs:

```bash
kubectl get crds | grep -E 'mysql|database|oci'
kubectl explain <CRD_NAME>
```

> **Important:** The exact API version and resource schema depend on the installed OSOK release. Use the CRD installed in the cluster rather than copying a manifest from a different release.

---

## Lab 6: Provisioning an OCI MySQL DB System from Kubernetes

### Objective

Create an OCI MySQL HeatWave DB System through a Kubernetes custom resource.

First inspect the installed MySQL CRD:

```bash
kubectl get crds | grep -i mysql
kubectl get crd <MYSQL_CRD_NAME> -o yaml
```

Identify:

- API group
- API version
- Kind
- Required fields
- Compartment OCID
- Availability domain
- Subnet OCID
- Shape
- Admin username
- Secret format

Create a Kubernetes secret for the database administrator:

```bash
kubectl create secret generic mysql-admin \
  --namespace $NAMESPACE \
  --from-literal=username=admin \
  --from-literal=password='<strong-password>'
```

Then create the MySQL custom resource (`mysql-db-system.yaml`) using the schema exposed by the installed CRD. A representative structure is:

```yaml
apiVersion: <OSOK_API_VERSION>
kind: <MYSQL_KIND>
metadata:
  name: lab-mysql
  namespace: demo
spec:
  compartmentId: <COMPARTMENT_OCID>
  displayName: lab-mysql
  availabilityDomain: <AVAILABILITY_DOMAIN>
  subnetId: <SUBNET_OCID>
  shapeName: <MYSQL_SHAPE>
  mysqlVersion: "<MYSQL_VERSION>"
  adminCredentials:
    secretName: mysql-admin
```

Apply it:

```bash
kubectl apply -f mysql-db-system.yaml
kubectl get <MYSQL_RESOURCE> lab-mysql -n $NAMESPACE
kubectl describe <MYSQL_RESOURCE> lab-mysql -n $NAMESPACE
```

Monitor the resource:

```bash
kubectl get events \
  --namespace $NAMESPACE \
  --sort-by=.lastTimestamp
```

The DB System may take several minutes to become available.

> **Cost warning:** Creating an OCI MySQL DB System incurs OCI resource usage and charges. Delete it after completing the lab.

---

## Lab 7: Kubernetes Service of Type LoadBalancer

### Objective

Expose an application using an OCI Load Balancer.

Create `loadbalancer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
  namespace: demo
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "100"
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get service frontend-lb -n $NAMESPACE
```

Wait for the external IP:

```bash
kubectl get service frontend-lb \
  --namespace $NAMESPACE \
  --watch
```

Test it:

```bash
curl http://<EXTERNAL-IP>
```

Inspect the provisioned OCI Load Balancer:

```bash
kubectl describe service frontend-lb -n $NAMESPACE
```

Optional cleanup:

```bash
kubectl delete service frontend-lb -n $NAMESPACE
```

---

## Lab 8: OCI Native Ingress Controller

### Objective

Route multiple HTTP applications through one OCI-native ingress endpoint.

Install the OCI Native Ingress Controller according to the current supported chart and IAM requirements for your OKE cluster. After installation, verify:

```bash
kubectl get pods -A | grep -i ingress
kubectl get ingressclass
```

Create a sample ingress (`ingress.yaml`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo
spec:
  ingressClassName: <OCI_NATIVE_INGRESS_CLASS>
  rules:
    - host: frontend.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    - host: backend.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 8080
```

Apply and inspect:

```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n $NAMESPACE
kubectl describe ingress demo-ingress -n $NAMESPACE
```

> **Note:** The exact IngressClass, annotations and IAM configuration should match the controller version installed in the cluster.

Test with host headers if DNS is not configured:

```bash
curl -H "Host: frontend.example.com" http://<INGRESS_IP>/
curl -H "Host: backend.example.com" http://<INGRESS_IP>/
```

---

## Lab 9: Running Applications on Arm-Based Nodes

### Objective

Schedule an Arm-compatible workload on Arm nodes.

Check node architectures:

```bash
kubectl get nodes \
  -L kubernetes.io/arch \
  -L kubernetes.io/os
```

List Arm nodes:

```bash
kubectl get nodes \
  -l kubernetes.io/arch=arm64
```

If Arm nodes already exist, deploy (`arm-demo.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: arm-demo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: arm-demo
  template:
    metadata:
      labels:
        app: arm-demo
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
      containers:
        - name: arm-demo
          image: docker.io/arm64v8/nginx:latest
          ports:
            - containerPort: 80
```

Apply and verify placement:

```bash
kubectl apply -f arm-demo.yaml
kubectl get pods -n $NAMESPACE -o wide
kubectl describe pod <POD_NAME> -n $NAMESPACE
```

### Important considerations

- The container image must support `linux/arm64`.
- Use multi-architecture images where possible.
- Native binaries compiled only for `amd64` will not run on Arm nodes.
- Some images use a multi-architecture manifest automatically.

---

## Lab 10: Running Applications on GPU Nodes

### Objective

Schedule a CUDA workload on GPU nodes.

Check for GPU nodes:

```bash
kubectl get nodes --show-labels | grep -i gpu
```

Check GPU resources:

```bash
kubectl describe node <GPU_NODE_NAME> | grep -i -A5 capacity
```

A GPU node should advertise a resource such as:

```text
nvidia.com/gpu
```

Create `gpu-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-demo
  namespace: demo
spec:
  nodeSelector:
    accelerator: nvidia
  restartPolicy: Never
  containers:
    - name: cuda
      image: docker.io/nvidia/cuda:12.2.0-base-ubuntu22.04
      command:
        - nvidia-smi
      resources:
        limits:
          nvidia.com/gpu: 1
```

The node selector may differ in your cluster. Discover the correct labels:

```bash
kubectl get nodes --show-labels
```

Apply and inspect:

```bash
kubectl apply -f gpu-demo.yaml
kubectl get pod gpu-demo -n $NAMESPACE -o wide
kubectl logs gpu-demo -n $NAMESPACE
```

Expected output includes NVIDIA GPU information. If the pod remains pending:

```bash
kubectl describe pod gpu-demo -n $NAMESPACE
```

Check:

- GPU node availability
- GPU node taints
- Matching tolerations
- NVIDIA device plugin
- GPU resource advertisement
- Requested GPU capacity
- Container image compatibility

Example toleration for a tainted GPU node:

```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

---

## Lab 11: Validation Checklist

Run the following checks at the end of the lab:

```bash
kubectl get nodes -o wide
kubectl get namespaces
kubectl get deployments -n $NAMESPACE
kubectl get pods -n $NAMESPACE -o wide
kubectl get services -n $NAMESPACE
kubectl get ingress -n $NAMESPACE
kubectl get events -n $NAMESPACE --sort-by=.lastTimestamp
```

Validate application access:

```bash
kubectl port-forward service/frontend 8080:80 -n $NAMESPACE
```

From another terminal:

```bash
curl http://localhost:8080
```

Validate scheduling:

```bash
kubectl get pods -n $NAMESPACE \
  -o custom-columns=\
NAME:.metadata.name,\
NODE:.spec.nodeName,\
STATUS:.status.phase
```

---

## Lab 12: Cleanup

Delete the lab workloads:

```bash
kubectl delete namespace $NAMESPACE
```

Remove OSOK if it was installed only for this lab:

```bash
helm uninstall osok -n osok-system
kubectl delete namespace osok-system
```

Remove the OCI MySQL DB System using the OSOK custom resource or OCI Console. Confirm that the resource is fully deleted before ending the lab. Also remove:

- OCI Load Balancers
- OCI Native Ingress resources
- OCIR repositories
- Unused node pools
- GPU or Arm node pools created specifically for the lab

---

## Suggested Lab Sequence

1. Verify OKE access.
2. Deploy a basic application.
3. Deploy the multi-tier application.
4. Push and pull a private image from OCIR.
5. Apply standard labels.
6. Install and validate OSOK.
7. Provision a MySQL DB System.
8. Expose an application with a LoadBalancer service.
9. Configure OCI Native Ingress.
10. Run an application on Arm nodes.
11. Run a GPU workload.
12. Validate and clean up.
