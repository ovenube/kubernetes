# Instalación infraestructura


# Instalación Manual

Desplegar el archivo "vpc-template.yaml" ubicado en la carpeta aws/cf. No es necesario la primera vez intervenir en los parametros del template, por lo que se puede instalar con los valores por defecto. Si se necesita duplicar el ambiente en la misma cuenta entonces, se deben especificar los parámetros.

```bash
aws cloudformation create-stack --stack-name ovenube-erpnext \
	--template-body file://aws/cf/vpc-template.yaml \
    --capabilities "CAPABILITY_NAMED_IAM" \
    --tags '[{"Key":"Project","Value":"Ovenube ERPNext","Key":"Author","Value":"Juan Espinoza", "Key":"Environment","Value":"Prod"}]' \
    --profile ovenube
```

Cuando se requiera agregar o eliminar recursos desde el template, usar el comando update-stack con los mismos parametros.

```bash
aws cloudformation update-stack \
	--stack-name ovenube-erpnext \
	--template-body file://aws/cf/vpc-template.yaml \
    --capabilities "CAPABILITY_NAMED_IAM" \
    --tags '[{"Key":"Project","Value":"Ovenube ERPNext","Key":"Author","Value":"Juan Espinoza", "Key":"Environment","Value":"Dev"}]'

```

### Parametros:

Agregar los siguientes parametros en la argumento de la opción "--parameter" del comando anteriormente descrito:

* VpcCIDR   Valor por defecto 172.17.0.0/24
* PublicSubnet1CIDR
* PublicSubnet2CIDR
* HibridSubnet1CIDR
* HibridSubnet2CIDR
* PrivateSubnet1CIDR
* PrivateSubnet2CIDR
* Proyecto:  valor por defecto "ovenube erpnext"
* Environment: Valor por defecto 

```bash
aws cloudformation update-stack \
....
--parameters "ParameterKey=Bucket,ParameterValue=${BUCKET}" \
```

# Despliegue de RDS

Es importante actualizar el archivo "parameters.json" ubicado en la carpeta CF con el id del security group y las subnets donde se desplegará la instancia.

```bash
aws cloudformation create-stack \
	--stack-name ovenube-erpnext-rds-dev \
	--template-body file://aws/cf/rds-mariadb-template.yaml \
	--parameters file://aws/cf/parameters.json \
    --capabilities "CAPABILITY_NAMED_IAM" \
    --tags '[{"Key":"Project","Value":"Ovenube ERPNext","Key":"Author","Value":"Juan Espinoza", "Key":"Environment","Value":"Dev"}]'
```
## Parámetros del template

Los únicos parámetros que no tienen valor por defecto son **DatabaseSecurityGroup** y **Subnet**.

* Project:
* Proveedor:
* Environment:
      - dev
      - qa
      - preprod
      - prod
* DatabaseInstanceType:
      - db.t3.medium
      - db.t3.large
      - db.r5.large
      - db.r5.xlarge
      - db.r5.2xlarge
      - db.r5.4xlarge
      - db.r5.8xlarge
      - db.r5.12xlarge
      - db.r5.16xlarge
* DatabaseMasterUsername: Largo maximo 16
* DatabaseName:  Nombre de base de datos, formato:   [a-zA-Z0-9]*
* DatabaseSecurityGroup: Idientificador del grupo de seguridad para base de datos.
* NumberOfSubnets:  entre 1 y 3
* Subnet:  Identificadores separados por comas.

## Instalación Cluster EKS

Requisitos:

* [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
* [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)
* Tener configurado las credenciales con permisos para desplegar EKS
* [kubctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html#eksctl-gs-install-kubectl)

### Permisos para eksctl

```yaml
{

}
```

### Instalar el cluster


Los archivos o templates para la instalación de EKS están contenidos dentro del directorio "eks". ubicarse dentro de ese directorio para comenzar con la instalación.

El template con la confuguración base del cluster se llama:  "cluster-defintion.yaml"

Modificar el id de la VPC y las subredes.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: OVENUBE-ERPNEXT-CLUSTER01
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
```

NOTA: Las subredes privadas en esta configuración se refieren a las redes hibridas.  De otra manera las redes privadas que se registren ahí deben tener acceso a NAT de otra manera la instalación fallará

Para crear el cluster, ejecutar el siguiente comando:

```bash
eksctl create cluster -f cluster-definition.yaml
```

Esto tomará varios minutos.

EL rsultado debe ser como el que sigue:

```bash
ℹ]  eksctl version 0.26.0
[ℹ]  using region us-east-1
[✔]  using existing VPC (vpc-040d11141283a178e) and subnets (private:[subnet-0677cf3d4af0c0a01 subnet-066bfd91afea7e685] public:[subnet-030a0184c0474bb2e subnet-06ae28a1511093957])
[!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
[ℹ]  using Kubernetes version 1.17
[ℹ]  creating EKS cluster "OVENUBE-ERPNEXT-CLUSTER01" in "us-east-1" region with 
[ℹ]  will create a CloudFormation stack for cluster itself and 0 nodegroup stack(s)
[ℹ]  will create a CloudFormation stack for cluster itself and 0 managed nodegroup stack(s)
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-east-1 --cluster=OVENUBE-ERPNEXT-CLUSTER01'
[ℹ]  CloudWatch logging will not be enabled for cluster "OVENUBE-ERPNEXT-CLUSTER01" in "us-east-1"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=us-east-1 --cluster=OVENUBE-ERPNEXT-CLUSTER01'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "OVENUBE-ERPNEXT-CLUSTER01" in "us-east-1"
[ℹ]  2 sequential tasks: { create cluster control plane "OVENUBE-ERPNEXT-CLUSTER01", no tasks }
[ℹ]  building cluster stack "eksctl-OVENUBE-ERPNEXT-CLUSTER01-cluster"
[ℹ]  deploying stack "eksctl-OVENUBE-ERPNEXT-CLUSTER01-cluster"
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/Users/msilva/.kube/config"
[ℹ]  2 sequential sub-tasks: { associate IAM OIDC provider, no tasks }
[✔]  all EKS cluster resources for "OVENUBE-ERPNEXT-CLUSTER01" have been created
[ℹ]  kubectl command should work with "/Users/msilva/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "OVENUBE-ERPNEXT-CLUSTER01" in "us-east-1" region is ready
```

## Crear node Group

Una vez creado el cluster EKS, se necesita agregar nodos al cluster que cumplen el rol de workers. Existen 2 tipos de crear grupo de nodos: 

* Instancias on demand
* Instancias Spot

### Instancias SPOT

Para crear un fleet de instancias spot, ejecutar el siguiente comando:

```bash
eksctl create nodegroup -f spot-instances.yaml
```

El resultado de exito debería ser similar a la siguiente salida:

```bash
[ℹ]  eksctl version 0.26.0
[ℹ]  using region us-east-1
[ℹ]  will use version 1.17 for new nodegroup(s) based on control plane version
[ℹ]  nodegroup "spot-node-group-2vcpu-8gb" present in the given config, but missing in the cluster
[ℹ]  nodegroup "spot-node-group-2vcpu-8gb" will use "ami-07250434f8a7bc5f1" [AmazonLinux2/1.17]
[ℹ]  1 nodegroup (spot-node-group-2vcpu-8gb) was included (based on the include/exclude rules)
[ℹ]  will create a CloudFormation stack for each of 1 nodegroups in cluster "OVENUBE-ERPNEXT-CLUSTER01"
[ℹ]  2 sequential tasks: { fix cluster compatibility, 1 task: { 1 task: { create nodegroup "spot-node-group-2vcpu-8gb" } } }
[ℹ]  checking cluster stack for missing resources
[ℹ]  cluster stack has all required resources
[ℹ]  building nodegroup stack "eksctl-OVENUBE-ERPNEXT-CLUSTER01-nodegroup-spot-node-group-2vcpu-8gb"
[ℹ]  deploying stack "eksctl-OVENUBE-ERPNEXT-CLUSTER01-nodegroup-spot-node-group-2vcpu-8gb"
[ℹ]  no tasks
[ℹ]  adding identity "arn:aws:iam::705735230819:role/eksctl-OVENUBE-ERPNEXT-CLU-NodeInstanceRole-378DGLM0GWDD" to auth ConfigMap
[ℹ]  nodegroup "spot-node-group-2vcpu-8gb" has 0 node(s)
[ℹ]  waiting for at least 2 node(s) to become ready in "spot-node-group-2vcpu-8gb"
[ℹ]  nodegroup "spot-node-group-2vcpu-8gb" has 2 node(s)
[ℹ]  node "ip-172-16-8-122.ec2.internal" is ready
[ℹ]  node "ip-172-16-9-200.ec2.internal" is ready
[✔]  created 1 nodegroup(s) in cluster "OVENUBE-ERPNEXT-CLUSTER01"
[✔]  created 0 managed nodegroup(s) in cluster "OVENUBE-ERPNEXT-CLUSTER01"
[ℹ]  checking security group configuration for all nodegroups
[ℹ]  all nodegroups have up-to-date configuration
```


