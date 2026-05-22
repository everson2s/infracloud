# PostgreSQL Backup on AKS using Azure Blob Storage

This README provides a reusable and generic approach to execute PostgreSQL backups from AKS and upload them directly to Azure Blob Storage using stream mode, avoiding local dump storage inside the pod.

The solution uses:

- Kubernetes CronJob
- PostgreSQL `pg_dump`
- Azure Blob Storage
- `azcopy`
- Azure Workload Identity
- Azure Container Registry (ACR)

---

## Architecture

```text
AKS
 └── Namespace
      └── CronJob
           └── Pod
                ├── pg_dump
                ├── azcopy
                └── Stream directly to Azure Blob

Azure Blob Storage
 └── Container
      └── postgres/
           └── database-name/
                └── YYYY-MM-DD-HHMM.dump
```

---

## Advantages

- No local storage for large dumps
- Lower AKS disk usage
- Reusable for multiple environments
- Azure native authentication
- Supports large databases
- Simple restore process
- Public repository friendly

---

## Requirements

- Azure CLI
- kubectl
- Docker
- AKS cluster
- Azure Container Registry
- Azure Storage Account
- PostgreSQL accessible from AKS

---

## Environment Variables

```bash
export SUBSCRIPTION_ID="<SUBSCRIPTION_ID>"

export AKS_RESOURCE_GROUP="<AKS_RESOURCE_GROUP>"
export AKS_CLUSTER_NAME="<AKS_CLUSTER_NAME>"

export BACKUP_RESOURCE_GROUP="<BACKUP_RESOURCE_GROUP>"

export STORAGE_ACCOUNT_NAME="<STORAGE_ACCOUNT_NAME>"
export STORAGE_CONTAINER_NAME="postgres-dumps"

export CONTAINER_REGISTRY_NAME="<CONTAINER_REGISTRY_NAME>"

export LOCATION="<AZURE_REGION>"

export BACKUP_NAMESPACE="backup"

export SERVICE_ACCOUNT_NAME="sa-postgres-backup"

export MANAGED_IDENTITY_NAME="mi-postgres-backup"

export POSTGRES_HOSTNAME="<POSTGRES_HOSTNAME>"
export POSTGRES_PORT="5432"

export POSTGRES_DATABASE_NAME="<POSTGRES_DATABASE_NAME>"

export POSTGRES_BACKUP_USERNAME="backup_user"

export BACKUP_SCHEDULE="0 2 * * *"
```

---

## Azure Login

```bash
az login
```

Select subscription:

```bash
az account set --subscription "$SUBSCRIPTION_ID"
```

---

## Connect to AKS

```bash
az aks get-credentials \
  --resource-group "$AKS_RESOURCE_GROUP" \
  --name "$AKS_CLUSTER_NAME" \
  --overwrite-existing
```

---

## Create Storage Account

```bash
az group create \
  --name "$BACKUP_RESOURCE_GROUP" \
  --location "$LOCATION"
```

```bash
az storage account create \
  --name "$STORAGE_ACCOUNT_NAME" \
  --resource-group "$BACKUP_RESOURCE_GROUP" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --https-only true \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2
```

Create container:

```bash
az storage container create \
  --name "$STORAGE_CONTAINER_NAME" \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --auth-mode login
```

---

## Create Docker Image

Create local folder:

```bash
mkdir postgres-backup-image
cd postgres-backup-image
```

Create `Dockerfile`:

```dockerfile
FROM postgres:16

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && curl -L https://aka.ms/downloadazcopy-v10-linux -o /tmp/azcopy.tar.gz \
    && tar -xf /tmp/azcopy.tar.gz -C /tmp \
    && cp /tmp/azcopy_linux_amd64_*/azcopy /usr/local/bin/azcopy \
    && chmod +x /usr/local/bin/azcopy \
    && rm -rf /tmp/azcopy* \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER postgres

CMD ["bash"]
```

---

## Login to ACR

```bash
az acr login --name "$CONTAINER_REGISTRY_NAME"
```

Build image:

```bash
docker build -t ${CONTAINER_REGISTRY_NAME}.azurecr.io/postgres-backup:1.0.0 .
```

Push image:

```bash
docker push ${CONTAINER_REGISTRY_NAME}.azurecr.io/postgres-backup:1.0.0
```

Validate tools inside image:

```bash
docker run --rm ${CONTAINER_REGISTRY_NAME}.azurecr.io/postgres-backup:1.0.0 pg_dump --version
```

```bash
docker run --rm ${CONTAINER_REGISTRY_NAME}.azurecr.io/postgres-backup:1.0.0 azcopy --version
```

---

## Allow AKS to Pull Image from ACR

```bash
az aks update \
  --resource-group "$AKS_RESOURCE_GROUP" \
  --name "$AKS_CLUSTER_NAME" \
  --attach-acr "$CONTAINER_REGISTRY_NAME"
```

---

## Enable Workload Identity

```bash
az aks update \
  --resource-group "$AKS_RESOURCE_GROUP" \
  --name "$AKS_CLUSTER_NAME" \
  --enable-oidc-issuer \
  --enable-workload-identity
```

Get OIDC issuer:

```bash
export AKS_OIDC_ISSUER="$(az aks show \
  --resource-group "$AKS_RESOURCE_GROUP" \
  --name "$AKS_CLUSTER_NAME" \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv)"
```

---

## Create Managed Identity

```bash
az identity create \
  --name "$MANAGED_IDENTITY_NAME" \
  --resource-group "$BACKUP_RESOURCE_GROUP" \
  --location "$LOCATION"
```

Get identity information:

```bash
export USER_ASSIGNED_CLIENT_ID="$(az identity show \
  --name "$MANAGED_IDENTITY_NAME" \
  --resource-group "$BACKUP_RESOURCE_GROUP" \
  --query 'clientId' \
  -o tsv)"
```

```bash
export USER_ASSIGNED_PRINCIPAL_ID="$(az identity show \
  --name "$MANAGED_IDENTITY_NAME" \
  --resource-group "$BACKUP_RESOURCE_GROUP" \
  --query 'principalId' \
  -o tsv)"
```

---

## Grant Blob Permissions

Get Storage Account ID:

```bash
export STORAGE_ACCOUNT_ID="$(az storage account show \
  --name "$STORAGE_ACCOUNT_NAME" \
  --resource-group "$BACKUP_RESOURCE_GROUP" \
  --query id \
  -o tsv)"
```

Assign role:

```bash
az role assignment create \
  --assignee-object-id "$USER_ASSIGNED_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" \
  --scope "$STORAGE_ACCOUNT_ID"
```

---

## Create Namespace

```bash
kubectl create namespace "$BACKUP_NAMESPACE"
```

---

## Create Federated Credential

```bash
az identity federated-credential create \
  --name "fc-postgres-backup" \
  --identity-name "$MANAGED_IDENTITY_NAME" \
  --resource-group "$BACKUP_RESOURCE_GROUP" \
  --issuer "$AKS_OIDC_ISSUER" \
  --subject "system:serviceaccount:${BACKUP_NAMESPACE}:${SERVICE_ACCOUNT_NAME}" \
  --audience "api://AzureADTokenExchange"
```

---

## Create PostgreSQL Secret

```bash
kubectl create secret generic postgres-backup-secret \
  -n "$BACKUP_NAMESPACE" \
  --from-literal=PGUSER="$POSTGRES_BACKUP_USERNAME" \
  --from-literal=PGPASSWORD='<POSTGRES_PASSWORD>'
```

For production, consider using Azure Key Vault with External Secrets Operator instead of a static Kubernetes Secret.

---

## Create ServiceAccount

Create `serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-postgres-backup
  namespace: backup
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
```

Replace placeholder:

```bash
sed -i "s/<CLIENT_ID>/${USER_ASSIGNED_CLIENT_ID}/g" serviceaccount.yaml
```

Apply:

```bash
kubectl apply -f serviceaccount.yaml
```

---

## Create CronJob

Create `cronjob-postgres-backup.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: backup
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  startingDeadlineSeconds: 1800
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 21600
      template:
        metadata:
          labels:
            azure.workload.identity/use: "true"
            app: postgres-backup
        spec:
          restartPolicy: Never
          serviceAccountName: sa-postgres-backup
          containers:
            - name: postgres-backup
              image: <CONTAINER_REGISTRY_NAME>.azurecr.io/postgres-backup:1.0.0
              imagePullPolicy: IfNotPresent
              resources:
                requests:
                  cpu: "500m"
                  memory: "1Gi"
                limits:
                  cpu: "2"
                  memory: "4Gi"
              env:
                - name: PGHOST
                  value: "<POSTGRES_HOSTNAME>"
                - name: PGPORT
                  value: "5432"
                - name: PGDATABASE
                  value: "<POSTGRES_DATABASE_NAME>"
                - name: PGUSER
                  valueFrom:
                    secretKeyRef:
                      name: postgres-backup-secret
                      key: PGUSER
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-backup-secret
                      key: PGPASSWORD
                - name: AZURE_STORAGE_ACCOUNT
                  value: "<STORAGE_ACCOUNT_NAME>"
                - name: AZURE_STORAGE_CONTAINER
                  value: "postgres-dumps"
              command:
                - /bin/bash
                - -c
                - |
                  set -euo pipefail

                  DATE=$(date +%Y-%m-%d-%H%M)
                  BLOB_NAME="postgres/${PGDATABASE}/${DATE}.dump"
                  DESTINATION="https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/${AZURE_STORAGE_CONTAINER}/${BLOB_NAME}"

                  echo "Azure login using Managed Identity..."
                  azcopy login --identity

                  echo "Starting PostgreSQL backup stream..."

                  pg_dump \
                    -h "$PGHOST" \
                    -p "$PGPORT" \
                    -U "$PGUSER" \
                    -d "$PGDATABASE" \
                    -Fc \
                  | azcopy copy "${DESTINATION}" --from-to PipeBlob

                  echo "Backup completed successfully."
```

Replace placeholders:

```bash
sed -i "s|<CONTAINER_REGISTRY_NAME>|${CONTAINER_REGISTRY_NAME}|g" cronjob-postgres-backup.yaml
sed -i "s|<POSTGRES_HOSTNAME>|${POSTGRES_HOSTNAME}|g" cronjob-postgres-backup.yaml
sed -i "s|<POSTGRES_DATABASE_NAME>|${POSTGRES_DATABASE_NAME}|g" cronjob-postgres-backup.yaml
sed -i "s|<STORAGE_ACCOUNT_NAME>|${STORAGE_ACCOUNT_NAME}|g" cronjob-postgres-backup.yaml
```

Apply CronJob:

```bash
kubectl apply -f cronjob-postgres-backup.yaml
```

---

## Run Manual Test

```bash
kubectl create job postgres-backup-manual \
  -n backup \
  --from=cronjob/postgres-backup
```

---

## View Logs

```bash
kubectl logs -n backup -l job-name=postgres-backup-manual -f
```

---

## Verify Blob Upload

```bash
az storage blob list \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --container-name "$STORAGE_CONTAINER_NAME" \
  --prefix "postgres/" \
  --auth-mode login \
  -o table
```

---

## Restore Backup

Download backup:

```bash
az storage blob download \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --container-name "$STORAGE_CONTAINER_NAME" \
  --name "postgres/<DATABASE>/<FILE>.dump" \
  --file "database.dump" \
  --auth-mode login
```

Restore database:

```bash
pg_restore \
  -h "<POSTGRES_HOSTNAME>" \
  -p "5432" \
  -U "postgres" \
  -d "<TARGET_DATABASE>" \
  --clean \
  --if-exists \
  --verbose \
  database.dump
```

Important: a backup without restore validation should not be considered reliable.

---

## Recommended Production Improvements

- Dedicated backup nodepool
- Private Endpoint for Blob Storage
- Private DNS Zone
- Lifecycle Management
- Soft Delete
- Immutable Blob Policy
- Azure Key Vault integration
- External Secrets Operator
- Monitoring and alerts
- Restore validation process
- Backup success/failure notification

---

## Optional: Dedicated Backup Nodepool

```bash
az aks nodepool add \
  --resource-group "$AKS_RESOURCE_GROUP" \
  --cluster-name "$AKS_CLUSTER_NAME" \
  --name npbackup \
  --node-count 1 \
  --node-vm-size Standard_D4s_v5 \
  --labels workload=backup \
  --node-taints workload=backup:NoSchedule
```

Add this to the CronJob if using a dedicated nodepool:

```yaml
nodeSelector:
  workload: backup

tolerations:
  - key: "workload"
    operator: "Equal"
    value: "backup"
    effect: "NoSchedule"
```

---

## Useful Commands

List CronJobs:

```bash
kubectl get cronjob -n backup
```

List Jobs:

```bash
kubectl get jobs -n backup
```

List Pods:

```bash
kubectl get pods -n backup
```

View logs:

```bash
kubectl logs -n backup -l app=postgres-backup --tail=100
```

Suspend CronJob:

```bash
kubectl patch cronjob postgres-backup \
  -n backup \
  -p '{"spec": {"suspend": true}}'
```

Resume CronJob:

```bash
kubectl patch cronjob postgres-backup \
  -n backup \
  -p '{"spec": {"suspend": false}}'
```

Delete manual Job:

```bash
kubectl delete job postgres-backup-manual -n backup
```

---

## Important Notes

- Always validate restore procedures.
- Backup without restore validation is not reliable.
- Large databases may require advanced backup strategies.
- Consider WAL archiving and PITR for enterprise workloads.
- For very large databases, evaluate physical/incremental backup tools such as pgBackRest.
