# Instalar Cluster con aws/eksctl

Desplegar el cluster con el siguiente comando:

```bash
aws/eksctl create cluster -f aws/aws/eks/cluster-definition.yaml --profile ovenube
```

## Cluster definition

Cambiar los datos de VPC Id y las Subnets. Los grupos de seguridad son creados automáticamente.

```yaml
apiVersion: aws/eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: OVENUBE-ERPNEXT-DEV-CLUSTER01
  region: us-east-1
  # version: 1.16

iam:
  withOIDC: true

vpc:
  id: vpc-0fe964d707f6c0760
  subnets:
    private:
      us-east-1b: { id: subnet-01088170ef5512eaf}
      us-east-1a: { id: subnet-0cf41598805532191}

    public: 
      us-east-1a: {id: subnet-0a19955363d120f6c}
      us-east-1b: {id: subnet-0a3115a8d01ba3817}

nodeGroups:
  - name: ng-1-workers
    labels: { role: workers }
    instanceType: t3.medium
    desiredCapacity: 2
    privateNetworking: true

```



## Crear Grupo de nodos con EC2

Ejecutar el siguiente comando

```bash

eksctl create nodegroup \
  --cluster ovenube\
  --region us-east-1 \
  --name ovenubepool \
  --node-type c5n.large \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2 \
  --profile ovenube
```

# NGINXG Ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx
```

## Instalar Cert Manager en el cluster

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.3.1 \
  --set installCRDs=true
```

## Crear cluster-issuer

```bash
kubectl create -n cert-manager -f aws/eks/cert-manager/cluster-issuer.yaml
```

# Crear NFS
```bash
helm repo add kvaps https://kvaps.github.io/charts
helm install nfs-server-provisioner kvaps/nfs-server-provisioner
```

# Instalar Base de datos
## Instalar MariaDB
```bash
helm install -n mariadb mariadb bitnami/mariadb -f aws/eks/mariadb/values-production.yaml
```

# Instalar ERPNext
```bash
kubectl create namespace erpnext
helm install frappe-bench-0001 -f erpnext/values.yaml \
    --namespace erpnext frappe/erpnext \
    --set mariadbHost=mariadb.mariadb.svc.cluster.local
```

## Crear secret en Kubernetes
```bash
kubectl create -n erpnext -f erpnext/mariadb-secret.yaml
```

## Get ERPNext pod
```bash
kubectl get pods -n erpnext
kubectl exec -n erpnext frappe-bench-0001-erpnext-worker-d-5c65d855c6-85jfr -it -- bash
```

## Modify common_site_config.json
```bash
cat > common_site_config.json

{
    "db_host": "mariadb.mariadb.svc.cluster.local",
    "db_port": 3306,
    "redis_cache": "redis://frappe-bench-0001-erpnext-redis-cache:13000",
    "redis_queue": "redis://frappe-bench-0001-erpnext-redis-queue:12000",
    "redis_socketio": "redis://frappe-bench-0001-erpnext-redis-socketio:11000",
    "socketio_port": 9000
}
```

## Create new-site job
```bash
kubectl create -n erpnext -f erpnext/add-site-job.yaml
```

## Get job status & logs
```bash
kubectl -n erpnext describe job create-new-site-tokhna-erpnext
kubectl logs -n erpnext create-new-site-tokhna-erpnext-stg-z4xmz
```

## Create new-site ingress
```bash
kubectl create -n erpnext -f erpnext/add-site-ingress.yaml
```

## Actualizar ERPNext
```bash
helm upgrade frappe-bench-0001 -f erpnext/values.yaml \
    --namespace erpnext frappe/erpnext \
    --set mariadbHost=mariadb.mariadb.svc.cluster.local
```