# ESMOS PM3 — Complete Deployment Guide
# Team G8-T05 | IS214 Enterprise Solution Management | March 2026

## Overview

**Project:** ESMOS PM3 Healthcare Meal Subscription Platform
**Environments:** Staging (Docker Compose on VM) + Production (AKS)
**Stack:** Odoo 18 + OCA Helpdesk | Moodle LMS | PostgreSQL 16 | MariaDB 10.11 | Nginx

## Resource Naming

| Resource | Name |
|----------|------|
| Azure Subscription | `f2b072c6-2ba0-4e46-9857-4ef884bcdff3` |
| Resource Group | `esmos-prod-rg` |
| Container Registry | `esmosprodacr` |
| AKS Cluster | `esmos-aks-cluster` |
| Staging VM | `esmos-stg-vm` |
| AKS Namespace | `production` |
| Staging Admin | `esmos` |
| Odoo Image | `esmosprodacr.azurecr.io/esmos-odoo:v1` |

## Credentials

| Service | User | Password |
|---------|------|----------|
| Odoo DB (PostgreSQL) | `odoo` | `odoo_esmos_2026` |
| Odoo Master Password | — | `214Odoo` |
| Moodle DB (MariaDB) | `moodle` | `moodle_esmos_2026` |
| Moodle DB Root | `root` | `root_esmos_2026` |
| Moodle Admin | `admin` | `Admin123!` |

## Images

| Service | Image |
|---------|-------|
| Odoo | `esmosprodacr.azurecr.io/esmos-odoo:v1` (custom) |
| Moodle | `ellakcy/moodle:mysql_maria_apache_latest` |
| PostgreSQL | `postgres:16-alpine` |
| MariaDB | `mariadb:10.11` |
| Nginx | `nginx:1.27-alpine` |

## Known Fixes (baked into all files from the start)

1. **Dockerfile**: use `--break-system-packages` for pip install (PEP 668 on Python 3.12)
2. **Odoo K8s**: initContainer with busybox running `chown -R 101:101 /var/lib/odoo` + `fsGroup: 101`
3. **Moodle image**: always `ellakcy/moodle:mysql_maria_apache_latest` (not `mulitbase`)
4. **Moodle on AKS**: LoadBalancer service (not ClusterIP behind Ingress) to avoid redirect loops
5. **Moodle DB port**: explicitly set `MOODLE_DB_PORT=3306` to override K8s service env injection
6. **Staging VM**: Standard_B2s (not B1s — 1GB RAM insufficient for 5 containers)
7. **Ingress**: only routes Odoo; Moodle gets its own LoadBalancer IP
8. **Pipeline YAML**: `pool: name: 'Default'` (self-hosted agent) + `failOnStdErr: false`
9. **WAF**: PCRE limits raised, OWASP CRS disabled for Moodle hostname, admin IP whitelist for `/web/database`

---

## PHASE 0: Prerequisites

### 0.1 Login (manual — browser auth required)
```bash
az login
az account set --subscription "f2b072c6-2ba0-4e46-9857-4ef884bcdff3"
```

### 0.2 Set environment variables (run in every terminal session)
```bash
export RESOURCE_GROUP="esmos-prod-rg"
export LOCATION="southeastasia"
export ACR_NAME="esmosprodacr"
export AKS_CLUSTER="esmos-aks-cluster"
export STAGING_VM="esmos-stg-vm"
export STAGING_ADMIN="esmos"
```

### 0.3 Create resource group
```bash
az group create --name $RESOURCE_GROUP --location $LOCATION
```

> **PAUSE — TICKETING CHECKPOINT**
> Create Jira task for Joash (System Configurator): **"Set up Azure resource group and prerequisites"**
> Move to In Progress in Jira. Screenshot.

---

## PHASE 1: Azure Container Registry

```bash
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Basic \
  --location $LOCATION \
  --admin-enabled true

# Get credentials — save these for Phase 3
export ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
export ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username -o tsv)
export ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

echo "Login Server: $ACR_LOGIN_SERVER"
echo "Username: $ACR_USERNAME"
echo "Password: $ACR_PASSWORD"
```

**SAVE these credentials** — you need them for the staging VM ACR login.

---

## PHASE 2: Build Custom Odoo 18 Image

### 2.1 Setup
```bash
mkdir -p ~/esmos-odoo/custom-addons && cd ~/esmos-odoo
cd custom-addons && git clone --branch 18.0 --depth 1 https://github.com/OCA/helpdesk.git oca-helpdesk && cd ..
```

### 2.2 Dockerfile

Create `~/esmos-odoo/Dockerfile`:

```dockerfile
FROM odoo:18.0
USER root
RUN pip3 install --no-cache-dir --break-system-packages num2words xlwt
RUN mkdir -p /mnt/extra-addons/oca-helpdesk
COPY custom-addons/oca-helpdesk/ /mnt/extra-addons/oca-helpdesk/
RUN echo "[options]" > /etc/odoo/odoo.conf && \
    echo "addons_path = /mnt/extra-addons,/mnt/extra-addons/oca-helpdesk,/usr/lib/python3/dist-packages/odoo/addons" >> /etc/odoo/odoo.conf && \
    echo "data_dir = /var/lib/odoo" >> /etc/odoo/odoo.conf && \
    echo "limit_time_cpu = 600" >> /etc/odoo/odoo.conf && \
    echo "limit_time_real = 1200" >> /etc/odoo/odoo.conf && \
    echo "db_maxconn = 64" >> /etc/odoo/odoo.conf && \
    echo "workers = 2" >> /etc/odoo/odoo.conf && \
    echo "max_cron_threads = 1" >> /etc/odoo/odoo.conf && \
    echo "admin_passwd = 214Odoo" >> /etc/odoo/odoo.conf
RUN chown -R odoo:odoo /mnt/extra-addons /etc/odoo
USER odoo
```

### 2.3 Build and push

**If you have Docker locally:**
```bash
docker build -t $ACR_LOGIN_SERVER/esmos-odoo:v1 .
az acr login --name $ACR_NAME
docker push $ACR_LOGIN_SERVER/esmos-odoo:v1
```

**If using Azure Cloud Shell (no local Docker):**
```bash
az acr build --registry $ACR_NAME --image esmos-odoo:v1 .
```

**Verify:**
```bash
az acr repository list --name $ACR_NAME -o table
```

> **PAUSE — TICKETING CHECKPOINT**
> Create Jira task for Joash: **"Build custom Odoo 18 image with OCA Helpdesk"**
> Move to Done. Screenshot.

---

## PHASE 3: Staging VM (Docker Compose)

### 3.1 Create VM (Standard_B2s — NOT B1s)
```bash
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $STAGING_VM \
  --image Ubuntu2404 \
  --size Standard_B2s \
  --admin-username $STAGING_ADMIN \
  --generate-ssh-keys \
  --location $LOCATION \
  --public-ip-sku Standard

export STAGING_IP=$(az vm show -d -g $RESOURCE_GROUP -n $STAGING_VM --query publicIps -o tsv)
echo "Staging IP: $STAGING_IP"
```

### 3.2 Open ports
```bash
az vm open-port --resource-group $RESOURCE_GROUP --name $STAGING_VM --port 22 --priority 100
az vm open-port --resource-group $RESOURCE_GROUP --name $STAGING_VM --port 80 --priority 110
az vm open-port --resource-group $RESOURCE_GROUP --name $STAGING_VM --port 8069 --priority 120
az vm open-port --resource-group $RESOURCE_GROUP --name $STAGING_VM --port 8080 --priority 130
```

### 3.3 SSH in and install Docker
```bash
ssh esmos@$STAGING_IP

# Inside VM:
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

### 3.4 Login to ACR from VM
```bash
docker login esmosprodacr.azurecr.io -u <ACR_USERNAME> -p '<ACR_PASSWORD>'
```

### 3.5 Create project directory
```bash
mkdir -p ~/esmos-staging/nginx && cd ~/esmos-staging
```

### 3.6 Create `.env`

```bash
cat > .env << 'ENV'
ACR_LOGIN_SERVER=esmosprodacr.azurecr.io
STAGING_IP=<YOUR_STAGING_IP>
ODOO_DB_USER=odoo
ODOO_DB_PASSWORD=odoo_esmos_2026
ODOO_MASTER_PASSWORD=214Odoo
MOODLE_DB_USER=moodle
MOODLE_DB_PASSWORD=moodle_esmos_2026
MOODLE_DB_ROOT_PASSWORD=root_esmos_2026
MOODLE_ADMIN_USER=admin
MOODLE_ADMIN_PASSWORD=Admin123!
ENV
```

Replace `<YOUR_STAGING_IP>` with the actual staging VM IP.

### 3.7 Create `docker-compose.yml`

```bash
cat > docker-compose.yml << 'YAML'
services:
  nginx:
    image: nginx:1.27-alpine
    container_name: esmos-nginx
    ports: ["80:80"]
    volumes: ["./nginx/nginx.conf:/etc/nginx/nginx.conf:ro"]
    depends_on: [odoo, moodle]
    restart: unless-stopped
    networks: [esmos-net]

  odoo:
    image: ${ACR_LOGIN_SERVER}/esmos-odoo:v1
    container_name: esmos-odoo
    depends_on: [odoo-db]
    environment:
      - HOST=odoo-db
      - PORT=5432
      - USER=${ODOO_DB_USER}
      - PASSWORD=${ODOO_DB_PASSWORD}
    volumes: [odoo-filestore:/var/lib/odoo]
    restart: unless-stopped
    networks: [esmos-net]

  odoo-db:
    image: postgres:16-alpine
    container_name: esmos-odoo-db
    environment:
      - POSTGRES_USER=${ODOO_DB_USER}
      - POSTGRES_PASSWORD=${ODOO_DB_PASSWORD}
      - POSTGRES_DB=postgres
    volumes: [odoo-db-data:/var/lib/postgresql/data]
    restart: unless-stopped
    networks: [esmos-net]

  moodle:
    image: ellakcy/moodle:mysql_maria_apache_latest
    container_name: esmos-moodle
    depends_on: [moodle-db]
    environment:
      - MOODLE_URL=http://${STAGING_IP}:8080
      - MOODLE_DB_HOST=moodle-db
      - MOODLE_DB_PORT=3306
      - MOODLE_DB_NAME=moodle
      - MOODLE_DB_USER=${MOODLE_DB_USER}
      - MOODLE_DB_PASSWORD=${MOODLE_DB_PASSWORD}
      - MOODLE_DB_TYPE=mariadb
      - MOODLE_ADMIN=${MOODLE_ADMIN_USER}
      - MOODLE_ADMIN_PASSWORD=${MOODLE_ADMIN_PASSWORD}
    volumes: [moodle-data:/var/moodledata]
    ports: ["8080:80"]
    restart: unless-stopped
    networks: [esmos-net]

  moodle-db:
    image: mariadb:10.11
    container_name: esmos-moodle-db
    environment:
      - MYSQL_ROOT_PASSWORD=${MOODLE_DB_ROOT_PASSWORD}
      - MYSQL_DATABASE=moodle
      - MYSQL_USER=${MOODLE_DB_USER}
      - MYSQL_PASSWORD=${MOODLE_DB_PASSWORD}
    volumes: [moodle-db-data:/var/lib/mysql]
    restart: unless-stopped
    networks: [esmos-net]

networks:
  esmos-net:

volumes:
  odoo-filestore:
  odoo-db-data:
  moodle-data:
  moodle-db-data:
YAML
```

### 3.8 Create `nginx/nginx-v1.conf` (V1 baseline — no WAF)

```bash
cat > nginx/nginx.conf << 'CONF'
events { worker_connections 1024; }
http {
    upstream odoo_backend { server odoo:8069; }
    upstream moodle_backend { server moodle:80; }
    server {
        listen 80;
        location / {
            proxy_pass http://odoo_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_read_timeout 720s;
            client_max_body_size 50m;
        }
        location /moodle/ {
            proxy_pass http://moodle_backend/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            client_max_body_size 50m;
        }
    }
}
CONF
```

### 3.9 Start staging
```bash
docker compose pull
docker compose up -d
sleep 30
docker compose ps
```

All 5 containers should show "Up".

> **PAUSE — TICKETING CHECKPOINT**
> Go to Odoo Helpdesk production. Shawmya creates Ticket **[DEPLOY-002] Staging Environment**.
> Follow the ticketing guide steps. Screenshot each stage.

> **PAUSE — TICKETING CHECKPOINT**
> Create Jira task for Joash: **"Deploy staging Docker Compose environment"**
> Move to Done. Screenshot.

---

## PHASE 4: AKS Production Cluster

### 4.1 Register provider (first time only)
```bash
az provider register --namespace Microsoft.ContainerService
# Wait for registration:
az provider show -n Microsoft.ContainerService --query "registrationState" -o tsv
# Should say "Registered" — may take 1-2 minutes
```

### 4.2 Create AKS cluster
```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --location $LOCATION \
  --enable-app-routing \
  --generate-ssh-keys \
  --attach-acr $ACR_NAME \
  --dns-name-prefix esmos-prod
```
This takes 5-10 minutes.

### 4.3 Get kubectl credentials
```bash
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing
kubectl get nodes
```

### 4.4 Create namespace and secrets
```bash
kubectl create namespace production

kubectl create secret generic odoo-db-secret --namespace production \
  --from-literal=POSTGRES_USER=odoo \
  --from-literal=POSTGRES_PASSWORD=odoo_esmos_2026 \
  --from-literal=POSTGRES_DB=postgres

kubectl create secret generic moodle-db-secret --namespace production \
  --from-literal=MYSQL_ROOT_PASSWORD=root_esmos_2026 \
  --from-literal=MYSQL_DATABASE=moodle \
  --from-literal=MYSQL_USER=moodle \
  --from-literal=MYSQL_PASSWORD=moodle_esmos_2026
```

### 4.5 Enable ModSecurity globally

```bash
kubectl patch configmap nginx -n app-routing-system --type merge -p '{"data":{"enable-modsecurity":"true","enable-owasp-modsecurity-crs":"true","use-gzip":"true","gzip-types":"text/plain text/css text/javascript application/json application/javascript application/xml application/xml+rss image/svg+xml"}}'
```

### 4.6 Create K8s manifest files

Create all files in the `k8s/` directory of your repo. See [K8s Manifest Files](#k8s-manifest-files) section below.

### 4.7 Apply manifests in order
```bash
kubectl apply -f k8s/storage.yaml && sleep 10
kubectl apply -f k8s/odoo-db.yaml && kubectl apply -f k8s/moodle-db.yaml && sleep 30
kubectl apply -f k8s/odoo.yaml && kubectl apply -f k8s/moodle.yaml && sleep 30
kubectl apply -f k8s/ingress-v1.yaml && sleep 60
```

### 4.8 Get external IPs
```bash
# Odoo (via Ingress)
export ODOO_IP=$(kubectl get ingress esmos-ingress -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Odoo: http://$ODOO_IP"

# Moodle (via LoadBalancer)
export MOODLE_IP=$(kubectl get svc moodle -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Moodle: http://$MOODLE_IP"
```

If Moodle IP shows empty, wait a minute and retry — LoadBalancer IP assignment takes time.

### 4.9 Update Moodle URL
```bash
kubectl set env deployment/moodle -n production MOODLE_URL="http://$MOODLE_IP"
```

### 4.10 Verify
```bash
kubectl get pods -n production
kubectl get ingress -n production
kubectl get svc moodle -n production
```

All 4 pods should be Running.

> **PAUSE — TICKETING CHECKPOINT**
> Go to Odoo Helpdesk production. Shawmya creates Ticket **[DEPLOY-001] AKS Production**.
> Follow the ticketing guide through all stages. Screenshot.

> **PAUSE — MANUAL SETUP**
> Set up Odoo database in browser: `http://<ODOO_IP>/web/database/manager`
> - Master password: `214Odoo`
> - Create database, install Helpdesk, Website, eCommerce modules

> **PAUSE — MANUAL SETUP**
> Set up Moodle in browser: `http://<MOODLE_IP>`
> - Create compliance training course

> **PAUSE — MANUAL SETUP**
> Create all 6 Odoo user accounts + 2 portal users per the ticketing guide.

---

## PHASE 5: CI/CD Pipeline (Azure DevOps)

### 5.1 Create pipeline files

Create `azure-pipelines-staging.yml` and `azure-pipelines-production.yml` in the repo root. See [Pipeline Files](#pipeline-files) section below.

### 5.2 Commit and push
```bash
git checkout staging
git add azure-pipelines-staging.yml azure-pipelines-production.yml
git commit -m "Add CI/CD pipeline configs for staging and production"
git push origin staging
```

> **PAUSE — MANUAL AZURE DEVOPS SETUP**
>
> **On the staging VM (SSH in first):**
> 1. Download and install the self-hosted Azure DevOps agent
> 2. Install Azure CLI: `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`
> 3. Install kubectl: `sudo snap install kubectl --classic`
> 4. Login: `az login --use-device-code`
> 5. Get AKS credentials: `az aks get-credentials --resource-group esmos-prod-rg --name esmos-aks-cluster --overwrite-existing`
>
> **In Azure DevOps portal:**
> 1. Create project → connect to GitHub repo
> 2. Create SSH service connection for staging VM
> 3. Create environments
> 4. Create both pipelines from existing YAML files

> **PAUSE — TICKETING CHECKPOINT**
> Shawmya creates Ticket **[DEPLOY-003] CI/CD Pipeline**.
> Follow ticketing guide. Screenshot.

---

## PHASE 6: WAF Change (RFC-001) — V1 to V2

> **PAUSE BEFORE STARTING**
> Zachary creates Ticket **[RFC-001]** in Odoo Helpdesk.
> Follow ALL 11 steps in the ticketing guide: New → Review → Planning → Authorize → Awaiting Implementation.

> **PAUSE — V1 SCAN**
> Zachary runs OWASP ZAP V1 baseline scan against production Odoo.
> Save report. Attach to ticket. Screenshot.

### 6.1 Apply V2 ingress
```bash
kubectl apply -f k8s/ingress-v2.yaml
```

### 6.2 Verify WAF is working
```bash
# Should return 403 (blocked)
curl -s -o /dev/null -w "XSS: HTTP %{http_code}\n" "http://<ODOO_IP>/?q=<script>alert(1)</script>"
curl -s -o /dev/null -w "SQLi: HTTP %{http_code}\n" "http://<ODOO_IP>/?id=1' OR 1=1--"
curl -s -o /dev/null -w "Scanner: HTTP %{http_code}\n" -H "User-Agent: sqlmap/1.0" "http://<ODOO_IP>/"

# Check gzip
curl -s -I -H "Accept-Encoding: gzip" http://<ODOO_IP>/ | grep -i "content-encoding"

# Check security headers
curl -s -I http://<ODOO_IP>/ | grep -i "x-frame\|x-content-type\|x-xss"

# Check rate limiting (burst 20 requests)
for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code} " "http://<ODOO_IP>/"; done
echo ""
```

> **PAUSE — V2 SCAN**
> Zachary runs OWASP ZAP V2 scan. Compare results with V1. Attach to ticket.

> **PAUSE — TESTING**
> Test manually — SQL injection (expect 403), rate limiting, gzip (check DevTools).
> Sahanya validates user experience. Yichen checks performance. Add notes to ticket.

### 6.3 Rollback drill
```bash
# Rollback to V1
kubectl apply -f k8s/ingress-v1.yaml
# Verify attacks now pass (200 instead of 403)
curl -s -o /dev/null -w "XSS after rollback: HTTP %{http_code}\n" "http://<ODOO_IP>/?q=<script>alert(1)</script>"

# Re-apply V2
kubectl apply -f k8s/ingress-v2.yaml
# Verify attacks blocked again
curl -s -o /dev/null -w "XSS after re-apply: HTTP %{http_code}\n" "http://<ODOO_IP>/?q=<script>alert(1)</script>"
```

> **PAUSE — TICKETING CHECKPOINT**
> Shawmya adds rollback drill note and closes RFC-001 ticket. Screenshot final ticket.

---

## PHASE 7: Service Requests (Onboarding Workflow)

> **PAUSE**
> Login as Jane Doe (portal) and submit Ticket **[SR-001]**.
> Shawmya assigns training. Leave at In Progress. Screenshot.

> **PAUSE**
> Login as Dr. Ahmad (portal) and submit Ticket **[SR-002]**.
> Process through all stages per ticketing guide.
> Joash creates Moodle account. Shawmya closes. Screenshot.

---

## PHASE 8: Bonus — Incident Demo

### 8.1 Kill a pod and watch self-healing
```bash
# Delete the Odoo pod
kubectl delete pod $(kubectl get pods -n production -l app=odoo -o name | head -1) -n production

# Watch it restart automatically
kubectl get pods -n production -w
```

The pod will be replaced within seconds — Kubernetes self-healing in action.

> **PAUSE — TICKETING CHECKPOINT**
> Yichen creates Ticket **[INC-001]**. Process through ticketing guide. Screenshot.

---

## PHASE 9: Domain and TLS (if time permits)

### Set Azure DNS labels on public IPs
```bash
# Find the Odoo ingress public IP resource
ODOO_PIP_NAME=$(az network public-ip list \
  --resource-group MC_esmos-prod-rg_esmos-aks-cluster_southeastasia \
  --query "[?ipAddress=='$ODOO_IP'].name" -o tsv)

az network public-ip update \
  --resource-group MC_esmos-prod-rg_esmos-aks-cluster_southeastasia \
  --name $ODOO_PIP_NAME \
  --dns-name esmos-odoo
# Result: esmos-odoo.southeastasia.cloudapp.azure.com
```

### Or: custom domain + cert-manager + Let's Encrypt
Requires purchasing a domain (~$2 for .xyz). Then install cert-manager on AKS, configure ClusterIssuer for Let's Encrypt, and add TLS to ingress.

---

## PHASE 10: Cleanup / Save Credits

### Stop AKS (saves ~$25/month)
```bash
az aks stop --name $AKS_CLUSTER --resource-group $RESOURCE_GROUP
```

### Start AKS
```bash
az aks start --name $AKS_CLUSTER --resource-group $RESOURCE_GROUP
```

### Stop staging VM (saves ~$15/month)
```bash
az vm deallocate --name $STAGING_VM --resource-group $RESOURCE_GROUP
```

### Start staging VM
```bash
az vm start --name $STAGING_VM --resource-group $RESOURCE_GROUP
```

### Delete everything (irreversible)
```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

---

## K8s Manifest Files

### k8s/storage.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: odoo-db-pvc
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: odoo-filestore-pvc
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: moodle-db-pvc
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: moodle-data-pvc
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
```

### k8s/odoo-db.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: odoo-db
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: odoo-db
  template:
    metadata:
      labels:
        app: odoo-db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: odoo-db-secret
          volumeMounts:
            - name: odoo-db-data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
      volumes:
        - name: odoo-db-data
          persistentVolumeClaim:
            claimName: odoo-db-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: odoo-db
  namespace: production
spec:
  selector:
    app: odoo-db
  ports:
    - port: 5432
      targetPort: 5432
```

### k8s/odoo.yaml (with initContainer fix)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: odoo
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: odoo
  template:
    metadata:
      labels:
        app: odoo
    spec:
      securityContext:
        fsGroup: 101
      initContainers:
        - name: fix-permissions
          image: busybox:1.36
          securityContext:
            runAsUser: 0
          command: ["sh", "-c", "mkdir -p /var/lib/odoo/sessions /var/lib/odoo/filestore && chown -R 101:101 /var/lib/odoo && chmod -R 775 /var/lib/odoo"]
          volumeMounts:
            - name: odoo-filestore
              mountPath: /var/lib/odoo
      containers:
        - name: odoo
          image: esmosprodacr.azurecr.io/esmos-odoo:v1
          ports:
            - containerPort: 8069
          env:
            - name: HOST
              value: odoo-db
            - name: PORT
              value: "5432"
            - name: USER
              valueFrom:
                secretKeyRef:
                  name: odoo-db-secret
                  key: POSTGRES_USER
            - name: PASSWORD
              valueFrom:
                secretKeyRef:
                  name: odoo-db-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: odoo-filestore
              mountPath: /var/lib/odoo
      volumes:
        - name: odoo-filestore
          persistentVolumeClaim:
            claimName: odoo-filestore-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: odoo
  namespace: production
spec:
  selector:
    app: odoo
  ports:
    - port: 8069
      targetPort: 8069
```

### k8s/moodle-db.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: moodle-db
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moodle-db
  template:
    metadata:
      labels:
        app: moodle-db
    spec:
      containers:
        - name: mariadb
          image: mariadb:10.11
          ports:
            - containerPort: 3306
          envFrom:
            - secretRef:
                name: moodle-db-secret
          volumeMounts:
            - name: moodle-db-data
              mountPath: /var/lib/mysql
      volumes:
        - name: moodle-db-data
          persistentVolumeClaim:
            claimName: moodle-db-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: moodle-db
  namespace: production
spec:
  selector:
    app: moodle-db
  ports:
    - port: 3306
      targetPort: 3306
```

### k8s/moodle.yaml (LoadBalancer — NOT behind Ingress)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: moodle
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moodle
  template:
    metadata:
      labels:
        app: moodle
    spec:
      containers:
        - name: moodle
          image: ellakcy/moodle:mysql_maria_apache_latest
          ports:
            - containerPort: 80
          env:
            - name: MOODLE_URL
              value: "http://MOODLE_IP_PLACEHOLDER"
            - name: MOODLE_DB_HOST
              value: moodle-db
            - name: MOODLE_DB_PORT
              value: "3306"
            - name: MOODLE_DB_NAME
              value: moodle
            - name: MOODLE_DB_TYPE
              value: mariadb
            - name: MOODLE_DB_USER
              valueFrom:
                secretKeyRef:
                  name: moodle-db-secret
                  key: MYSQL_USER
            - name: MOODLE_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: moodle-db-secret
                  key: MYSQL_PASSWORD
            - name: MOODLE_ADMIN
              value: admin
            - name: MOODLE_ADMIN_PASSWORD
              value: "Admin123!"
          volumeMounts:
            - name: moodle-data
              mountPath: /var/moodledata
      volumes:
        - name: moodle-data
          persistentVolumeClaim:
            claimName: moodle-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: moodle
  namespace: production
spec:
  type: LoadBalancer
  selector:
    app: moodle
  ports:
    - port: 80
      targetPort: 80
```

### k8s/ingress-v1.yaml (Odoo only — baseline, no WAF)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: esmos-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "720"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: odoo
                port:
                  number: 8069
```

### k8s/ingress-v2.yaml (Odoo only — WAF + rate limiting + gzip + caching)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: esmos-ingress
  namespace: production
  annotations:
    # --- Proxy Settings ---
    nginx.ingress.kubernetes.io/proxy-read-timeout: "720"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-buffering: "on"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"

    # --- ModSecurity WAF + OWASP CRS ---
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-modsecurity-crs: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRequestBodyAccess On
      SecResponseBodyAccess Off
      SecAuditEngine RelevantOnly
      SecPcreMatchLimit 500000
      SecPcreMatchLimitRecursion 500000
      SecRuleRemoveById 200005
      SecRule REQUEST_URI "@beginsWith /web/database" "id:1000,phase:1,allow,nolog"
      SecRule REQUEST_URI "@contains /etc/passwd" "id:1001,phase:1,deny,status:403"
      SecRule REQUEST_URI "@contains ../" "id:1002,phase:1,deny,status:403"
      SecRule ARGS "@detectXSS" "id:1003,phase:2,deny,status:403"
      SecRule ARGS "@detectSQLi" "id:1004,phase:2,deny,status:403"
      SecRule REQUEST_URI "@contains /proc/self" "id:1005,phase:1,deny,status:403"
      SecRule REQUEST_HEADERS:User-Agent "@contains sqlmap" "id:1006,phase:1,deny,status:403"

    # --- Rate Limiting (5 req/sec per IP, burst 10) ---
    nginx.ingress.kubernetes.io/limit-rps: "5"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "2"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # --- Security Headers + Caching ---
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;
      add_header Permissions-Policy "camera=(), microphone=()" always;
      add_header Cache-Control "public, max-age=1800" always;

spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: odoo
                port:
                  number: 8069
```

---

## Pipeline Files

### azure-pipelines-staging.yml

```yaml
trigger:
  branches:
    include:
      - staging
    exclude:
      - production
      - main

pool:
  name: 'Default'

variables:
  stagingVM: '<STAGING_IP>'
  stagingUser: 'esmos'
  sshServiceConnection: 'ssh-esmos-staging'

stages:
  - stage: DeployStaging
    displayName: 'Deploy to Staging VM'
    jobs:
      - job: DeployDockerCompose
        displayName: 'Update and restart Docker Compose'
        steps:
          - task: CopyFilesOverSSH@0
            displayName: 'Copy docker-compose files to staging VM'
            inputs:
              sshEndpoint: '$(sshServiceConnection)'
              sourceFolder: '$(Build.SourcesDirectory)/docker-compose'
              contents: |
                docker-compose.yml
                .env.example
                nginx/**
              targetFolder: '/home/$(stagingUser)/esmos-staging'
              overwrite: true

          - task: SSH@0
            displayName: 'Pull images and restart containers'
            inputs:
              sshEndpoint: '$(sshServiceConnection)'
              runOptions: 'inline'
              failOnStdErr: false
              inline: |
                cd ~/esmos-staging
                docker compose pull
                docker compose down
                docker compose up -d
                sleep 10
                docker compose ps
```

### azure-pipelines-production.yml

```yaml
trigger:
  branches:
    include:
      - production
    exclude:
      - staging
      - main

pool:
  name: 'Default'

stages:
  - stage: DeployProduction
    displayName: 'Deploy to AKS Production'
    jobs:
      - job: DeployKubernetes
        displayName: 'Apply K8s manifests to AKS'
        steps:
          - script: |
              kubectl apply -f k8s/ -n production
              sleep 15
              kubectl get pods -n production
              kubectl get ingress -n production
            displayName: 'Deploy K8s manifests'
```

---

## .gitignore

```
# Environment files with real credentials
docker-compose/.env
.env

# Planning and local files
.planning/
.claude/

# Python
*.pyc
__pycache__/

# OS files
.DS_Store

# Build artifacts
*.tar.gz
```

---

## Quick Reference

### Switch WAF on/off
```bash
kubectl apply -f k8s/ingress-v2.yaml   # V2: WAF on
kubectl apply -f k8s/ingress-v1.yaml   # V1: WAF off (rollback)
```

### Save credits
```bash
az aks stop --name esmos-aks-cluster --resource-group esmos-prod-rg
az vm deallocate --name esmos-stg-vm --resource-group esmos-prod-rg
```

### Start back up
```bash
az aks start --name esmos-aks-cluster --resource-group esmos-prod-rg
az vm start --name esmos-stg-vm --resource-group esmos-prod-rg
```

### Self-healing demo
```bash
kubectl delete pod $(kubectl get pods -n production -l app=odoo -o name | head -1) -n production
kubectl get pods -n production -w
```

### Scale Moodle
```bash
kubectl scale deployment/moodle -n production --replicas=2
kubectl scale deployment/moodle -n production --replicas=1
```
