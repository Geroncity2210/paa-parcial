# Integrantes:

- David Esteban Díaz Vargas: AKA Esteban7108
- Diego Norberto Diaz Algarin: AKA 1JuL
- Juan Pablo Moreno Patarroyo: AKA Geroncity2210

##  Instalación manual con Helm

parte de Juanito

## Configuración de valores por ambiente

### `values.yaml` — base compartida

| Parámetro | Valor base |
|---|---|
| `frontend.image` | `1jul/front-taller2:latest` |
| `backend.image` | `1jul/back-taller2:1.1.0` |
| `backend.service.port` | `8080` |
| `frontend.service.port` | `80` |
| `postgresql.auth.database` | `animales-db` |
| `postgresql.auth.existingSecret` | `db-secret` |
| `ingress.host` | `prod.127.0.0.1.sslip.io` |

## Configuración de ArgoCD

### Instalación de ArgoCD (si no está instalado)
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que esté listo
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# Obtener contraseña inicial del admin
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### Aplicar las definiciones de Application
```bash
kubectl apply -f environments/dev/application.yaml
kubectl apply -f environments/prod/application.yaml
```

### Cómo sincroniza ArgoCD automáticamente

Ambas `Application` tienen configurada la política de **sincronización automática**. El flujo es:
```
Push a Git  ──►  ArgoCD detecta el cambio  ──►  Aplica diff en Kubernetes
                     (polling cada ~3 min)          (sin comandos manuales)
```

Las opciones configuradas en ambos `application.yaml`:

| Opción | Efecto |
|---|---|
| `automated.prune: true` | Elimina del clúster recursos que ya no existen en Git |
| `automated.selfHeal: true` | Revierte cambios manuales hechos directamente en Kubernetes |
| `CreateNamespace=true` | Crea el namespace automáticamente si no existe |
| `PrunePropagationPolicy=foreground` | Espera confirmación antes de eliminar recursos dependientes |
| `PruneLast=true` | Elimina recursos obsoletos al final del sync, no al principio |

### Definición `environments/dev/application.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pedido-app-dev
  namespace: argocd
  labels:
    app.kubernetes.io/name: pedido-app
    app.kubernetes.io/instance: dev
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/Geroncity2210/paa-parcial
    targetRevision: HEAD
    path: charts/pedido-app
    helm:
      valueFiles:
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: pedido-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
```

La definición de prod es idéntica cambiando `name: pedido-app-prod`, `instance: prod`, `namespace: pedido-app-prod` y usando `values-prod.yaml`.

---

##  Endpoints de acceso

| Entorno | Frontend | Backend API |
|---|---|---|
| **dev** | `http://dev.127.0.0.1.sslip.io/` | `http://dev.127.0.0.1.sslip.io/api` |
| **prod** | `http://prod.127.0.0.1.sslip.io/` | `http://prod.127.0.0.1.sslip.io/api` |

> Los hosts se resuelven automáticamente a `127.0.0.1` vía `sslip.io`, lo que facilita pruebas locales con Minikube o kind sin modificar `/etc/hosts`.


##  Horizontal Pod Autoscaler (HPA)

El backend escala automáticamente según uso de CPU. Configuración por ambiente:

| Parámetro | dev | prod |
|---|---|---|
| `minReplicas` | `1` | `3` |
| `maxReplicas` | `3` | `6` |
| `cpuUtilization` | `75%` | `75%` |
| `resources.requests.cpu` | `100m` | `100m` |

## Persistencia de datos

PostgreSQL usa un `PersistentVolumeClaim` para que los datos sobrevivan reinicios del pod:
```yaml
postgresql:
  primary:
    persistence:
      enabled: true
      size: 1Gi   # dev: 1Gi  |  prod: 5Gi
