---
title: "Kubernetes Real Time Scenarios Part 1"
layout: single
comments: true
categories:
  - posts
tags:
  - K8S
  - DevOps
  - Real-Time
  - Scenarios
  - Kubernetes Management
  - Rancher
  - Troubleshooting
---

### Scenario 1 

How do you ensure a zero-downtime deployment for multiple services in a production environment?

**<ins>Resolution:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Achieving zero-downtime deployments requires careful use of rolling updates, readiness probes, and traffic routing. Here’s how you can implement it:

1. **Rolling Update:** Use rolling updates with a small increment in replicas to ensure that only a portion of the pods are updated at a time, reducing the risk of downtime:

```
apiVersion: apps/v1 
kind: Deployment 
metadata: 
    name: multi-service-deployment 
spec: 
  replicas: 4 
  strategy: 
    type: RollingUpdate 
    rollingUpdate: 
      maxUnavailable: 1 
      maxSurge: 1
```

2. **Readiness Probes:** Define readiness probes to ensure that the new pod version is ready to serve traffic before it’s added to the load balancer:

```
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

3. **Traffic Routing:** Use an Ingress or Istio Gateway to route traffic based on header/cookie values for canary testing:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: multi-service-route
spec:
  hosts:
  - multi-service.example.com
  http:
  - route:
    - destination:
        host: multi-service
        subset: v2 
        weight: 10
    - destination:
        host: multi-service
        subset: v1
        weight: 90
```

---

### Scenario 2

How do you implement a blue-green deployment strategy with an easy rollback option?

**<ins>Resolution:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In a blue-green deployment, two identical environments (blue and green) are maintained. Traffic is shifted between them without downtime, allowing easy rollbacks.

1. **Create Two Deployments (blue and green):** Define two separate deployments for the blue and green environments.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: myapp-container
        image: myapp:v1
```

2. **Update the Ingress:** Modify the Ingress (or Istio Gateway) to point to the green deployment when ready:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - backend:
          serviceName: myapp-green
          servicePort: 80
```

3. **Rolling Back:** If the green deployment has issues, switch back to blue by modifying the Ingress to point to myapp-blue again. This ensures that users always hit a stable environment.

---

### Scenario 3

How do you autoscale pods based on custom application metrics, such as requests per second (RPS)?

**<ins>Resolution:**

&nbsp;&nbsp;&nbsp;&nbsp;To autoscale based on custom metrics, integrate a custom metrics API (such as Prometheus Adapter) with the Horizontal Pod Autoscaler (HPA).

1. **Expose Custom Metrics:** Use a monitoring tool like Prometheus to export custom metrics (e.g., RPS). Example metric:
```
http_requests_total{job="myapp"}
```
2. **Create a Custom Metrics API Adapter:** Deploy a custom metrics API adapter (e.g., k8s-prometheus-adapter). This adapter translates Prometheus metrics into a format Kubernetes understands.

3. **Create an HPA for Custom Metrics:** Define an HPA to autoscale based on the custom RPS metric:

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_total
      target:
        type: AverageValue
        averageValue: "100"
```

This setup ensures that Kubernetes scales the number of pods based on application RPS.

---

### Scenario 4

How do you manage workloads in multiple Kubernetes clusters efficiently?

**<ins>Resolution:**

&nbsp;&nbsp;&nbsp;&nbsp;Managing workloads across multiple clusters requires the use of **multi-cluster tools** such as **KubeFed** (Kubernetes Federation) or **Rancher**.

1. **Install KubeFed:** Install KubeFed to manage multiple clusters as a single entity:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts/kubefed/README.md

```
2. **Federated Resources:** With KubeFed, you can create **federated deployments** that are automatically deployed to all member clusters:

```
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: myapp
spec:
  template:
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: myapp
      template:
        metadata:
          labels:
            app: myapp
        spec:
          containers:
          - name: myapp-container
            image: myapp:v1
```

3. **Rancher for Multi-Cluster Management:** Alternatively, use **Rancher** to manage clusters through a centralized dashboard. Rancher integrates with CI/CD tools and provides RBAC controls across multiple clusters.

---

### Scenario 5

What happens when a node fails, and how do you recover pods on that node?

**<ins>Resolution:**

&nbsp;&nbsp;&nbsp;&nbsp;Kubernetes automatically reschedules pods on healthy nodes when a node becomes unavailable.

1. **Pod Eviction:** If a node fails, the **Node Controller** detects the failure after a default timeout of 5 minutes. Pods on the failed node are marked as "Terminating" or "Unknown," and Kubernetes attempts to reschedule them on a healthy node.

2. **Node Termination Grace Period:** You can configure the node termination grace period to a lower value if you want faster eviction:
```
kubectl edit no <node-name>
```
Set grace period in the taint

3. **Pod Disruption Budgets:** Use **PodDisruptionBudget (PDB)** to ensure that critical workloads maintain minimum availability during node failures:

```
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 80%
  selector:
    matchLabels:
    app: myapp
```
---
