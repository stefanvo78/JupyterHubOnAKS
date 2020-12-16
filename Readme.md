# Installing JupyterHub in AKS

## create postgres db - will be used for JupyterHub
```
KUBE_GROUP=<AKS_RESOURCEGROUP_NAME>
POSTGRES_GROUP=<POSTGRES_RESOURCEGROUP_NAME>
KUBE_NAME=<AKS_NAME>
POSTGRES_NAME=<JUPYTERHUB_NAME>
LOCATION=<LOCATION>
STORAGE_ACCOUNT=<JUPYTERHUB_STORAGEACCOUNT_NAME>
PSQL_PASSWORD=$(openssl rand -base64 10)
VNET_NAME=<AKS_VNET_NAME>
SUBNET_NAME=<AKS_SUBNET_NAME>


az postgres server create --resource-group $POSTGRES_GROUP --name $POSTGRES_NAME  --location $LOCATION --admin-user myadmin --admin-password $PSQL_PASSWORD --sku-name GP_Gen5_2 --version 11 --ssl-enforcement disabled
```

## ServiceEndpoint Microsoft.Sql has to be added to Subnet
```
az network vnet subnet update -g $POSTGRES_GROUP -n $SUBNET_NAME  --vnet-name $VNET_NAME --service-endpoints Microsoft.Sql
```


## connect to psql using shell.azure.com and create database for jupyterhub
### Configuration of PostGres Firewall needed!
```
psql --host=$POSTGRES_NAME.postgres.database.azure.com --username=myadmin@$POSTGRES_NAME --dbname=postgres --port=5432


CREATE DATABASE jupyterhub;
```

## configure connection security for postgres
```
az postgres server vnet-rule create -n myRule -g $POSTGRES_GROUP -s $POSTGRES_NAME --vnet-name $VNET_NAME --subnet $SUBNET_NAME

az postgres server update --resource-group $POSTGRES_GROUP --name $POSTGRES_NAME --public-network-access disabled
```



## create storage account

```
az storage account create --resource-group  MC_$(echo $KUBE_GROUP)_$(echo $KUBE_NAME)_$(echo $LOCATION) --name $STORAGE_ACCOUNT --location $LOCATION --sku Standard_LRS --kind StorageV2 --access-tier hot --https-only false
```

## create azure file storage class -  delete existing storageclass azurefile before or edit 
```
kubectl apply -f storageclass.yaml
```

## deploy jupyterhub in cluster with rbac

create cloud provider azure files role
```
kubectl apply -f persistentvolumeclaim.yaml

kubectl apply -f clusterrole.yaml
```

```
openssl rand -hex 32
```

create config.yaml with content
if you want to use existing config.yaml edit password and username
```
cat  <<EOF >config.yaml
proxy:
  secretToken: "774629f880afc0302830c19a9f09be4f59e98b242b65983cea7560e828df2978"
hub:
  uid: 1000
  cookieSecret: "774629f880afc0302830c19a9f09be4f59e98b242b65983cea7560e828df2978"
  db:
    type: postgres
    url: postgres+psycopg2://myadmin@$POSTGRES_NAME:$PSQL_PASSWORD@$POSTGRES_NAME.postgres.database.azure.com:5432/jupyterhub
singleuser:
  storage:
    dynamic:
      storageClass: azurefile
rbac:
   enabled: true
EOF
```

install jhub
```

RELEASE=jhub
NAMESPACE=jhub


helm upgrade --cleanup-on-fail -install jhub jupyterhub/jupyterhub --namespace jhub --create-namespace --version=0.9.0 --values config.yaml
```

cleanup

```
helm delete jhub --purge
kubectl delete ns jhub
```

## deploy jupyterhub without RBAC

```
openssl rand -hex 32
```

create config.yaml with content
if you want to use existing config.yaml edit password and username
```
cat  <<EOF >config.yaml
proxy:
  secretToken: "774629f880afc0302830c19a9f09be4f59e98b242b65983cea7560e828df2978"
hub:
  uid: 1000
  cookieSecret: "774629f880afc0302830c19a9f09be4f59e98b242b65983cea7560e828df2978"
  db:
    type: postgres
    url: postgres+psycopg2://myadmin@$KUBE_NAME:$PSQL_PASSWORD@$KUBE_NAME.postgres.database.azure.com:5432/jupyterhub
singleuser:
  storage:
    dynamic:
      storageClass: azurefile
rbac:
   enabled: false
EOF
```

install jhub
```
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update

RELEASE=jhub
NAMESPACE=jhub

helm upgrade --cleanup-on-fail \
  --install $RELEASE jupyterhub/jupyterhub \
  --namespace $NAMESPACE \
  --create-namespace \
  --version=0.9.0 \
  --values config.yaml

kubectl get service --namespace jhub

```

cleanup

```
helm delete jhub --purge
kubectl delete ns jhub
```


