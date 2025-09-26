# Terraform configuration to deploy a production-ready Kubernetes microservices setup
# Resources created:
# - Namespace `production`
# - ConfigMap (frontend-config)
# - Secret (backend-secret) - expects plaintext variables, will be base64 encoded
# - Deployments: backend, frontend (with probes, resources)
# - Services: backend-service, frontend-service
# - Ingress (nginx) with TLS secret reference
# - HorizontalPodAutoscaler (backend) via kubernetes_manifest
# - PodDisruptionBudget (backend) via kubernetes_manifest

# NOTE:
# - This configuration uses the Kubernetes provider and the kubernetes_manifest resource
#   to create HPA/PDB which may not have first-class support in older provider versions.
# - You must have Terraform >= 1.0 and the Kubernetes provider installed.
# - This assumes you already have a working kubeconfig and an ingress controller (nginx) & metrics-server installed.

terraform {
  required_version = ">= 1.0"
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.11.0"
    }
  }
}

provider "kubernetes" {
  config_path = var.kubeconfig_path
  config_context = var.kubeconfig_context
}

variable "kubeconfig_path" {
  description = "Path to kubeconfig file"
  type        = string
  default     = "~/.kube/config"
}

variable "kubeconfig_context" {
  description = "Kubeconfig context (optional)"
  type        = string
  default     = ""
}

variable "domain" {
  description = "Primary host domain for ingress"
  type        = string
  default     = "myapp.example.com"
}

variable "backend_image" {
  description = "Backend container image"
  type        = string
  default     = "myrepo/backend:1.0.0"
}

variable "frontend_image" {
  description = "Frontend container image"
  type        = string
  default     = "myrepo/frontend:1.0.0"
}

variable "db_user" {
  description = "DB username (plaintext)"
  type        = string
  default     = "myuser"
}

variable "db_pass" {
  description = "DB password (plaintext)"
  type        = string
  default     = "mypass"
  sensitive   = true
}

# Namespace
resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
  }
}

# ConfigMap
resource "kubernetes_config_map" "frontend_config" {
  metadata {
    name      = "frontend-config"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  data = {
    APP_ENV = "production"
  }
}

# Secret (Opaque) - values will be base64 encoded by Kubernetes provider
resource "kubernetes_secret" "backend_secret" {
  metadata {
    name      = "backend-secret"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  data = {
    DB_USER = base64encode(var.db_user)
    DB_PASS = base64encode(var.db_pass)
  }
  type = "Opaque"
}

# Backend Deployment
resource "kubernetes_deployment" "backend" {
  metadata {
    name      = "backend"
    namespace = kubernetes_namespace.production.metadata[0].name
    labels = {
      app = "backend"
    }
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "backend"
      }
    }

    template {
      metadata {
        labels = {
          app = "backend"
        }
      }

      spec {
        container {
          name  = "backend"
          image = var.backend_image

          port {
            container_port = 3000
          }

          env_from {
            secret_ref {
              name = kubernetes_secret.backend_secret.metadata[0].name
            }
          }

          resources {
            limits = {
              cpu    = "500m"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "256Mi"
            }
          }

          liveness_probe {
            http_get {
              path = "/health"
              port = 3000
            }
            initial_delay_seconds = 10
            period_seconds        = 15
          }

          readiness_probe {
            http_get {
              path = "/ready"
              port = 3000
            }
            initial_delay_seconds = 5
            period_seconds        = 10
          }
        }
      }
    }
  }
}

# Backend Service
resource "kubernetes_service" "backend_service" {
  metadata {
    name      = "backend-service"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  spec {
    selector = {
      app = "backend"
    }
    port {
      port        = 80
      target_port = 3000
    }
    type = "ClusterIP"
  }
}

# Frontend Deployment
resource "kubernetes_deployment" "frontend" {
  metadata {
    name      = "frontend"
    namespace = kubernetes_namespace.production.metadata[0].name
    labels = {
      app = "frontend"
    }
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "frontend"
      }
    }

    template {
      metadata {
        labels = {
          app = "frontend"
        }
      }

      spec {
        container {
          name  = "frontend"
          image = var.frontend_image

          port {
            container_port = 80
          }

          env_from {
            config_map_ref {
              name = kubernetes_config_map.frontend_config.metadata[0].name
            }
          }

          resources {
            limits = {
              cpu    = "400m"
              memory = "256Mi"
            }
            requests = {
              cpu    = "200m"
              memory = "128Mi"
            }
          }

          liveness_probe {
            http_get {
              path = "/"
              port = 80
            }
            initial_delay_seconds = 10
            period_seconds        = 20
          }

          readiness_probe {
            http_get {
              path = "/"
              port = 80
            }
            initial_delay_seconds = 5
            period_seconds        = 10
          }
        }
      }
    }
  }
}

# Frontend Service
resource "kubernetes_service" "frontend_service" {
  metadata {
    name      = "frontend-service"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  spec {
    selector = {
      app = "frontend"
    }
    port {
      port        = 80
      target_port = 80
    }
    type = "ClusterIP"
  }
}

# Ingress (networking.k8s.io/v1)
resource "kubernetes_ingress_v1" "prod_ingress" {
  metadata {
    name      = "prod-ingress"
    namespace = kubernetes_namespace.production.metadata[0].name
    annotations = {
      "nginx.ingress.kubernetes.io/rewrite-target" = "/"
    }
  }

  spec {
    ingress_class_name = "nginx"

    tls {
      hosts      = [var.domain]
      secret_name = "tls-secret"
    }

    rule {
      host = var.domain

      http {
        path {
          path     = "/"
          path_type = "Prefix"
          backend {
            service {
              name = kubernetes_service.frontend_service.metadata[0].name
              port {
                number = 80
              }
            }
          }
        }

        path {
          path     = "/api"
          path_type = "Prefix"
          backend {
            service {
              name = kubernetes_service.backend_service.metadata[0].name
              port {
                number = 80
              }
            }
          }
        }
      }
    }
  }
}

# HorizontalPodAutoscaler for backend - using kubernetes_manifest to ensure compatibility
resource "kubernetes_manifest" "backend_hpa" {
  manifest = yamldecode(<<-EOT
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
EOT
  )
}

# PodDisruptionBudget (backend)
resource "kubernetes_manifest" "backend_pdb" {
  manifest = yamldecode(<<-EOT
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
EOT
  )
}

# (Optional) NetworkPolicy: allow frontend->backend only
resource "kubernetes_manifest" "np_frontend_to_backend" {
  manifest = yamldecode(<<-EOT
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 3000
EOT
  )
}

# Outputs
output "namespace" {
  value = kubernetes_namespace.production.metadata[0].name
}

output "frontend_service" {
  value = kubernetes_service.frontend_service.metadata[0].name
}

output "backend_service" {
  value = kubernetes_service.backend_service.metadata[0].name
}
