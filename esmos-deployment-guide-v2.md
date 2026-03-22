# ESMOS PM3 — Complete Deployment Guide
# Team G8-T05 | IS214 Enterprise Solution Management | March 2026

## Overview

Two environments: Staging (Docker Compose on VM) + Production (AKS). ACR bridges both.

**Stack:** Odoo 18 + OCA Helpdesk | Moodle (ellakcy) + MariaDB | PostgreSQL 16 | Nginx
**Cost:** ~$45/month (~$25-30 with shutdown policy)

## Naming Convention

| Resource | Name |
|----------|------|
| Resource Group | esmos-pm3-rg |
| Container Registry | esmospm3acr |
| AKS Cluster | esmos-prod-cluster |
| Staging VM | esmos-staging-vm |
| AKS Namespace | production |
| Odoo Image | esmospm3acr.azurecr.io/esmos-odoo:v1 |

## Claude Code Compatibility

**CAN automate:** All az, kubectl, docker commands. All file creation.
**CANNOT automate:** az login (browser auth), Odoo setup wizard, Moodle installation wizard, OWASP ZAP scans.
**Usage:** Run az login yourself first. Then give Claude Code this file and say "execute Phase 0 through Phase 4."

---

## PHASE 0: Prerequisites

### 0.1 Set variables (run in every terminal session)
```bash
export RESOURCE_GROUP="esmos-pm3-rg"
export LOCATION="southeastasia"
export ACR_NAME="esmospm3acr"
export AKS_CLUSTER="esmos-prod-cluster"
export STAGING_VM="esmos-staging-vm"
export STAGING_ADMIN="esmos"
```

### 0.2 Login and create resource group
```bash
az login
az account set --subscription "Azure for Students"
az group create --name $RESOURCE_GROUP --location $LOCATION
```

---

## PHASE 1: Azure Container Registry

```bash
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic --location $LOCATION --admin-enabled true

export ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
export ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username -o tsv)
export ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

echo "Login Server: $ACR_LOGIN_SERVER"
echo "Username: $ACR_USERNAME"
echo "Password: $ACR_PASSWORD"
```

SAVE these credentials — you need them for the staging VM.

---

## PHASE 2: Build Custom Odoo 18 Image

### 2.1 Setup
```bash
mkdir -p ~/esmos-odoo/custom-addons && cd ~/esmos-odoo
cd custom-addons && git clone --branch 18.0 --depth 1 https://github.com/OCA/helpdesk.git oca-helpdesk && cd ..
```

### 2.2 Dockerfile
```bash
cat > Dockerfile << 'EOF'
FROM odoo:18.0
USER root
RUN pip3 install --no-cache-dir num2words xlwt
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
EOF
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

Verify: `az acr repository list --name $ACR_NAME -o table`

---

## PHASE 3: Staging VM (Docker Compose)

### 3.1 Create VM
```bash
az vm create --resource-group $RESOURCE_GROUP --name $STAGING_VM --image Ubuntu2404 --size Standard_B1s --admin-username $STAGING_ADMIN --generate-ssh-keys --location $LOCATION --public-ip-sku Standard

export STAGING_IP=$(az vm show -d -g $RESOURCE_GROUP -n $STAGING_VM --query publicIps -o tsv)
echo "Staging IP: $STAGING_IP"
```

Or use Azure Portal: VM → Create → B1s → Ubuntu 24.04 → Southeast Asia

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
docker login esmospm3acr.azurecr.io -u <ACR_USERNAME> -p '<ACR_PASSWORD>'
```

### 3.5 Create project files
```bash
mkdir -p ~/esmos-staging/nginx && cd ~/esmos-staging
```

Create `.env`:
```bash
cat > .env << 'ENV'
ACR_LOGIN_SERVER=esmospm3acr.azurecr.io
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

Create `docker-compose.yml`:
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
    image: ellakcy/moodle:mulitbase_apache_latest
    container_name: esmos-moodle
    depends_on: [moodle-db]
    environment:
      - MOODLE_URL=http://localhost:8080
      - MOODLE_DB_HOST=moodle-db
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

Create `nginx/nginx.conf` (V1 baseline — no WAF):
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

### 3.6 Start staging
```bash
docker compose pull
docker compose up -d
sleep 30
docker compose ps
```

All 5 containers should show "Up". Then set up Odoo and Moodle in your browser.

---

## PHASE 4: AKS Production Cluster

### 4.1 Create cluster (from local machine or Cloud Shell, NOT staging VM)
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

# Takes 5-10 minutes. Then:
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing
kubectl get nodes
```

### 4.2 Create namespace and secrets
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

### 4.3 Create and apply all K8s manifests

Create the files as shown in the full guide (storage.yaml, odoo-db.yaml, odoo.yaml, moodle-db.yaml, moodle.yaml, ingress-v1.yaml, ingress-v2.yaml).

Deploy in order:
```bash
kubectl apply -f storage.yaml && sleep 10
kubectl apply -f odoo-db.yaml && kubectl apply -f moodle-db.yaml && sleep 30
kubectl apply -f odoo.yaml && kubectl apply -f moodle.yaml && sleep 30
kubectl apply -f ingress-v1.yaml && sleep 60
kubectl get ingress esmos-ingress -n production
kubectl get pods -n production
```

### 4.4 Get production IP and update Moodle URL
```bash
export PROD_IP=$(kubectl get ingress esmos-ingress -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Production: http://$PROD_IP"
kubectl set env deployment/moodle -n production MOODLE_URL="http://$PROD_IP/moodle"
```

Then set up Odoo and Moodle in browser.

---

## PHASE 5: RFC 1 Change (V1 → V2)

### Apply WAF change
```bash
kubectl apply -f ingress-v2.yaml
```

### Rollback
```bash
kubectl apply -f ingress-v1.yaml
```

### Re-apply
```bash
kubectl apply -f ingress-v2.yaml
```

---

## PHASE 6: Daily Operations

### Save credits
```bash
az aks stop --name $AKS_CLUSTER --resource-group $RESOURCE_GROUP
az aks start --name $AKS_CLUSTER --resource-group $RESOURCE_GROUP
az vm deallocate --name $STAGING_VM --resource-group $RESOURCE_GROUP
az vm start --name $STAGING_VM --resource-group $RESOURCE_GROUP
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

### Check logs
```bash
kubectl logs deployment/odoo -n production
kubectl logs deployment/moodle -n production
```

### Delete everything
```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```
