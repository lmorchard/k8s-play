# Deploying Mastodon on Kubernetes

This guide walks through deploying a complete Mastodon instance on a bare-metal Kubernetes cluster.

## Prerequisites

- Kubernetes cluster (tested on v1.31)
- kubectl configured to access your cluster
- NFS or shared storage accessible from all nodes
- Basic understanding of Kubernetes concepts

## Initial Setup

### 1. Create Namespace
```bash
kubectl create namespace mastodon
kubectl config set-context --current --namespace=mastodon
```

### 2. Set Up Shared Storage

#### Mount NFS on all nodes:
```bash
# On each node
sudo mkdir -p /mnt/shared
sudo mount -t nfs <nfs-server-ip>:/path/to/share /mnt/shared
echo "<nfs-server-ip>:/path/to/share /mnt/shared nfs defaults 0 0" | sudo tee -a /etc/fstab
```

#### Create storage resources:
```yaml
# shared-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: shared-storage
  hostPath:
    path: /mnt/shared/mastodon/postgres
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mastodon-media-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: shared-storage
  hostPath:
    path: /mnt/shared/mastodon/media
    type: DirectoryOrCreate
```

### 3. Create PVCs
```yaml
# pvcs.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: mastodon
spec:
  storageClassName: shared-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mastodon-media
  namespace: mastodon
spec:
  storageClassName: shared-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
```

## Deploy Core Services

### 1. PostgreSQL
```yaml
# postgres.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: mastodon
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14-alpine
        env:
        - name: POSTGRES_USER
          value: mastodon
        - name: POSTGRES_PASSWORD
          value: mastodon_db_password
        - name: POSTGRES_DB
          value: mastodon_production
        - name: POSTGRES_HOST_AUTH_METHOD
          value: md5
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
          subPath: postgres
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: mastodon
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
```

### 2. Redis
```yaml
# redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: mastodon
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: mastodon
spec:
  selector:
    app: redis
  ports:
  - port: 6379
```

## Generate Mastodon Secrets

### 1. Run secret generation job:
```yaml
# generate-secrets.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: generate-secrets
  namespace: mastodon
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: generate-secrets
        image: ghcr.io/mastodon/mastodon:v4.3.1
        command: 
        - /bin/bash
        - -c
        - |
          echo "=== Generating encryption keys ==="
          bundle exec rails db:encryption:init
          echo ""
          echo "=== Generating VAPID keys ==="
          bundle exec rails mastodon:webpush:generate_vapid_key
          echo ""
          echo "=== Generating SECRET_KEY_BASE ==="
          echo "SECRET_KEY_BASE=$(bundle exec rails secret)"
          echo ""
          echo "=== Generating OTP_SECRET ==="
          echo "OTP_SECRET=$(bundle exec rails secret)"
```

### 2. Create configuration secret with generated values:
```yaml
# mastodon-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mastodon-config
  namespace: mastodon
type: Opaque
stringData:
  # Database
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: mastodon_production
  DB_USER: mastodon
  DB_PASS: mastodon_db_password
  
  # Redis
  REDIS_HOST: redis
  REDIS_PORT: "6379"
  
  # Federation
  LOCAL_DOMAIN: "192.168.0.200:30080"  # Change to your domain/IP
  RAILS_FORCE_SSL: "false"  # For HTTP access
  
  # Rails encryption (paste from generate-secrets output)
  ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY: "paste-here"
  ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT: "paste-here"
  ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY: "paste-here"
  
  # Core secrets (paste from generate-secrets output)
  SECRET_KEY_BASE: "paste-here"
  OTP_SECRET: "paste-here"
  
  # VAPID keys (paste from generate-secrets output)
  VAPID_PRIVATE_KEY: "paste-here"
  VAPID_PUBLIC_KEY: "paste-here"
  
  # Basic settings
  RAILS_ENV: production
  NODE_ENV: production
  SINGLE_USER_MODE: "false"
```

## Deploy Mastodon Services

### 1. Web Service
```yaml
# mastodon-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mastodon-web
  namespace: mastodon
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mastodon-web
  template:
    metadata:
      labels:
        app: mastodon-web
    spec:
      containers:
      - name: web
        image: ghcr.io/mastodon/mastodon:v4.3.1
        command: ["bundle", "exec", "puma", "-C", "config/puma.rb"]
        envFrom:
        - secretRef:
            name: mastodon-config
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: media
          mountPath: /mastodon/public/system
      volumes:
      - name: media
        persistentVolumeClaim:
          claimName: mastodon-media
---
apiVersion: v1
kind: Service
metadata:
  name: mastodon-web
  namespace: mastodon
spec:
  selector:
    app: mastodon-web
  ports:
  - port: 3000
```

### 2. Streaming Service
```yaml
# mastodon-streaming.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mastodon-streaming
  namespace: mastodon
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mastodon-streaming
  template:
    metadata:
      labels:
        app: mastodon-streaming
    spec:
      containers:
      - name: streaming
        image: ghcr.io/mastodon/mastodon-streaming:v4.3.1
        command: ["node", "./streaming/index.js"]
        envFrom:
        - secretRef:
            name: mastodon-config
        ports:
        - containerPort: 4000
---
apiVersion: v1
kind: Service
metadata:
  name: mastodon-streaming
  namespace: mastodon
spec:
  selector:
    app: mastodon-streaming
  ports:
  - port: 4000
```

### 3. Sidekiq (Background Jobs)
```yaml
# mastodon-sidekiq.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mastodon-sidekiq
  namespace: mastodon
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mastodon-sidekiq
  template:
    metadata:
      labels:
        app: mastodon-sidekiq
    spec:
      containers:
      - name: sidekiq
        image: ghcr.io/mastodon/mastodon:v4.3.1
        command: ["bundle", "exec", "sidekiq"]
        envFrom:
        - secretRef:
            name: mastodon-config
        volumeMounts:
        - name: media
          mountPath: /mastodon/public/system
      volumes:
      - name: media
        persistentVolumeClaim:
          claimName: mastodon-media
```

## Initialize Database

```yaml
# db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mastodon-db-migrate
  namespace: mastodon
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: mastodon-db-migrate
        image: ghcr.io/mastodon/mastodon:v4.3.1
        command: ["bundle", "exec", "rails", "db:setup"]
        envFrom:
        - secretRef:
            name: mastodon-config
```

## External Access

```yaml
# nodeport-services.yaml
apiVersion: v1
kind: Service
metadata:
  name: mastodon-nodeport
  namespace: mastodon
spec:
  type: NodePort
  selector:
    app: mastodon-web
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30080
```

## Deployment Steps

1. **Create directories on NFS share:**
   ```bash
   sudo mkdir -p /mnt/shared/mastodon/postgres
   sudo mkdir -p /mnt/shared/mastodon/media
   sudo chmod 777 /mnt/shared/mastodon/*
   ```

2. **Apply storage configuration:**
   ```bash
   kubectl apply -f shared-storage.yaml
   kubectl apply -f pvcs.yaml
   ```

3. **Deploy core services:**
   ```bash
   kubectl apply -f postgres.yaml
   kubectl apply -f redis.yaml
   ```

4. **Generate secrets:**
   ```bash
   kubectl apply -f generate-secrets.yaml
   kubectl logs -f job/generate-secrets
   # Copy the output values
   ```

5. **Create config secret:**
   ```bash
   # Update mastodon-config.yaml with generated values
   kubectl apply -f mastodon-config.yaml
   ```

6. **Deploy Mastodon services:**
   ```bash
   kubectl apply -f mastodon-web.yaml
   kubectl apply -f mastodon-streaming.yaml
   kubectl apply -f mastodon-sidekiq.yaml
   ```

7. **Initialize database:**
   ```bash
   kubectl apply -f db-migrate.yaml
   kubectl logs -f job/mastodon-db-migrate
   ```

8. **Create NodePort service:**
   ```bash
   kubectl apply -f nodeport-services.yaml
   ```

## Access Options

### NodePort (may redirect to HTTPS):
```bash
http://<any-node-ip>:30080
```

### Port Forward (recommended for testing):
```bash
kubectl port-forward service/mastodon-web 8080:3000
# Access at http://localhost:8080
```

## Create Admin User

```bash
kubectl exec -it deployment/mastodon-web -- \
  bin/tootctl accounts create admin \
  --email admin@example.com \
  --confirmed \
  --role Owner
```

## Troubleshooting

### Check pod status:
```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Database issues:
```bash
# Test database connection
kubectl run pg-test --rm -it --image=postgres:14-alpine --restart=Never -- \
  psql -h postgres -U mastodon -d mastodon_production -c "SELECT version();"
```

### SSL redirect issues:
If Mastodon redirects to HTTPS, ensure `RAILS_FORCE_SSL: "false"` is in your secret and use port-forward instead of NodePort.

### Storage issues:
```bash
kubectl get pv,pvc
kubectl describe pvc <pvc-name>
```

## Clean Up

To completely remove the deployment:
```bash
kubectl delete namespace mastodon
# Remove NFS data if needed
sudo rm -rf /mnt/shared/mastodon
```
