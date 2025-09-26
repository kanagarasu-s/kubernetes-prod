
# Kubernetes Production Microservice Setup

This document describes a **production-ready Kubernetes setup** for a simple microservice application with a **frontend** and a **backend**.  
It includes best practices like **Namespaces, ConfigMaps, Secrets, Probes, HPA, PDB, and TLS Ingress**.

---

## 1. Namespace
We separate production workloads into the `production` namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

---

## 2. ConfigMap & Secret
ConfigMaps store non-sensitive environment variables, and Secrets store sensitive data like DB credentials.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: production
data:
  APP_ENV: "production"
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: production
type: Opaque
data:
  DB_USER: bXl1c2Vy   # base64 for 'myuser'
  DB_PASS: bXlwYXNz   # base64 for 'mypass'
```

---

## 3. Backend Deployment & Service
Includes liveness/readiness probes, resource limits, and secrets.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: production
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myrepo/backend:1.0.0
        ports:
        - containerPort: 3000
        envFrom:
        - secretRef:
            name: backend-secret
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

---

## 4. Frontend Deployment & Service
Includes probes and uses ConfigMap for environment values.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: production
  labels:
    app: frontend
spec:
  replicas: 2
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
        image: myrepo/frontend:1.0.0
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: frontend-config
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "400m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: production
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

---

## 5. Ingress with TLS
Ingress exposes frontend and backend with HTTPS enabled.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

---

## 6. Autoscaling & Availability
HPA scales backend pods automatically, and PDB ensures availability during disruptions.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
  namespace: production
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: backend
```
