set up azure

AZURE_BACKUP_RESOURCE_GROUP=velero
AZURE_STORAGE_ACCOUNT_NAME=veleromarcel
BLOB_CONTAINER=mycluster
AZURE_BACKUP_SUBSCRIPTION_ID=

# set subscription
az account set --subscription $AZURE_BACKUP_SUBSCRIPTION_ID
# resource group
az group create -n $AZURE_BACKUP_RESOURCE_GROUP --location WestUS

# storage account
az storage account create \
    --name $AZURE_STORAGE_ACCOUNT_NAME \
    --resource-group $AZURE_BACKUP_RESOURCE_GROUP \
    --sku Standard_GRS

# get key
AZURE_STORAGE_ACCOUNT_ACCESS_KEY=`az storage account keys list --account-name $AZURE_STORAGE_ACCOUNT_NAME --query "[?keyName == 'key1'].value" -o tsv`

# blob container
az storage container create -n $BLOB_CONTAINER \
  --public-access off \
  --account-name $AZURE_STORAGE_ACCOUNT_NAME \
  --account-key $AZURE_STORAGE_ACCOUNT_ACCESS_KEY


_____________________________________________________________________

if u are using kind 

kind create cluster --name velero --image kindest/node:v1.19.1

_____________________________________________________________________


# install curl & kubectl
apk add --no-cache curl nano
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
export KUBE_EDITOR="nano"

_____________________________________________________________________
install velero 

curl -L -o /tmp/velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/v1.5.1/velero-v1.5.1-linux-amd64.tar.gz 
tar -C /tmp -xvf /tmp/velero.tar.gz
mv /tmp/velero-v1.5.1-linux-amd64/velero /usr/local/bin/velero
chmod +x /usr/local/bin/velero

velero --help

_____________________________________________________________________

install velero for azure

# Azure credential file
cat << EOF  > /tmp/credentials-velero
AZURE_STORAGE_ACCOUNT_ACCESS_KEY=${AZURE_STORAGE_ACCOUNT_ACCESS_KEY}
AZURE_CLOUD_NAME=AzurePublicCloud
EOF

velero install \
    --provider azure \
    --plugins velero/velero-plugin-for-microsoft-azure:v1.1.0 \
    --bucket $BLOB_CONTAINER \
    --secret-file /tmp/credentials-velero \
    --backup-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,storageAccount=$AZURE_STORAGE_ACCOUNT_NAME,storageAccountKeyEnvVar=AZURE_STORAGE_ACCOUNT_ACCESS_KEY,subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID \
    --use-volume-snapshots=true


kubectl -n velero get pods
kubectl logs deployment/velero -n velero

___________________________________________________________________

back up

velero backup create default-namespace-backup --include-namespaces default

# describe
velero backup describe default-namespace-backup

# logs
velero backup logs default-namespace-backup

___________________________________________________________________
restore 

velero restore create default-namespace-backup --from-backup default-namespace-backup

# describe
velero restore describe default-namespace-backup

#logs 
velero restore logs default-namespace-backup

___________________________________________________________________
install postgres

helm repo add bitnami https://charts.bitnami.com/bitnami

helm install mypostgres bitnami/postgresql --set volumePermissions.enabled=true

export POSTGRES_PASSWORD=$(kubectl get secret --namespace default mypostgres-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
