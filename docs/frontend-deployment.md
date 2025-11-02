# Frontend Deployment on Kubernetes (GitOps)

This document describes how to deploy the Strato frontend (`frontend.gamel.dk`) to Kubernetes
with HTTPS using **NGINX Ingress**, **MetalLB**, and **cert-manager** via GitOps.

---

## 🧩 Overview

| Component | Purpose |
|------------|----------|
| **Frontend Deployment** | Next.js container app |
| **Service (ClusterIP)** | Exposes port 3000 inside cluster |
| **Ingress (NGINX)** | Routes external HTTPS traffic to the frontend |
| **Cert-Manager + Let’s Encrypt** | Issues and renews TLS certificates |
| **MetalLB** | Provides a public IP (LoadBalancer) in bare-metal environments |
| **NGINX Ingress Controller** | Handles ingress rules and TLS termination |

---

## ⚙️ Prerequisites

Before applying manifests, ensure:

- Domain `frontend.gamel.dk` points to your MetalLB public IP (`A` record → `130.225.37.164`)
- Kubernetes cluster running (e.g., k3s or kubeadm)
- `helm`, `kubectl`, and `kubeseal` (optional) installed

---

## 🧱 Infrastructure Components

### 1. MetalLB

`metallb-config.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
    - 130.225.37.164/32  # your public IP
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

Apply:
```bash
kubectl apply -f metallb-config.yaml
```

---

### 2. NGINX Ingress Controller

Install using Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx   --namespace ingress-nginx   --create-namespace   --set controller.ingressClassResource.name=nginx   --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx   --set controller.ingressClassResource.default=true   --set controller.service.type=LoadBalancer   --set controller.service.loadBalancerIP=130.225.37.164
```

Check:
```bash
kubectl get svc -n ingress-nginx
```

Expected:
```
ingress-nginx-controller   LoadBalancer   10.43.x.x   130.225.37.164   80:xxxx/TCP,443:xxxx/TCP
```

---

### 3. Cert-Manager (Let’s Encrypt)

Install cert-manager (if not already installed):

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager   --namespace cert-manager   --create-namespace   --set installCRDs=true
```

Create ClusterIssuer:

`cluster-issuer.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <YOUR_EMAIL_HERE>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply:
```bash
kubectl apply -f cluster-issuer.yaml
```

---

## 🚀 Application Manifests

### Namespace
`namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-deployment
```

---

### Deployment
`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: cgamel/p7-frontend:<IMAGE_TAG>
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: NEXT_PUBLIC_API_BASE_URL
              valueFrom:
                secretKeyRef:
                  name: frontend-secret
                  key: NEXT_PUBLIC_API_BASE_URL
            - name: NEXT_PUBLIC_BASE_URL
              valueFrom:
                secretKeyRef:
                  name: frontend-secret
                  key: NEXT_PUBLIC_BASE_URL
```

---

### Service
`service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: frontend-deployment
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP
```

---

### Ingress
`ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
  namespace: frontend-deployment
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: frontend.gamel.dk
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 3000
  tls:
    - hosts:
        - frontend.gamel.dk
      secretName: frontend-tls
```

---

### HPA (optional)
`hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: frontend-deployment
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 1
  maxReplicas: 2
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## ✅ Apply all manifests

```bash
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml
kubectl apply -f cluster-issuer.yaml
```

---

## 🔍 Verification

1. **Check ingress**
   ```bash
   kubectl get ingress -n frontend-deployment
   ```
   Should show `ADDRESS: 130.225.37.164` and `TLS: frontend-tls`.

2. **Check cert**
   ```bash
   kubectl get certificate -n frontend-deployment
   kubectl describe certificate frontend-tls -n frontend-deployment
   ```
   Should show:
   ```
   Status:
     Conditions:
       Type: Ready
       Status: True
   ```

3. **Access**
   - `https://frontend.gamel.dk` → should show your app with a 🔒 padlock.

4. **Renewal**
   Certificates are automatically renewed by cert-manager before expiry (90 days default).

---

## 🧩 Troubleshooting

| Issue | Command | Fix |
|-------|----------|-----|
| LoadBalancer stuck `<pending>` | `kubectl describe svc -n ingress-nginx` | Ensure MetalLB has IP in range and not used by another LB |
| Cert shows self-signed | `kubectl get challenges -A` | Check cert-manager logs; ensure DNS + ingress are reachable |
| Browser says “unsafe” | Clear cache or test in private window | Happens if old self-signed cert cached |

---

## 🧭 Notes

- For multi-service setups, you can reuse the same NGINX controller and ClusterIssuer.
- MetalLB can advertise multiple IPs if you expand the pool range.
- All manifests can be committed to a GitOps repo (e.g., ArgoCD or FluxCD).
