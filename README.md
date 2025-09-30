# North Deployment on AWS

## Install EKS Cluster
First, launch an EKS cluster by following the steps in the [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html). 
The steps in summary:

---

1. **Create the EKS Cluster**
   - Follow the steps in the AWS documentation to launch an EKS cluster.
   - Choose custom configuration.
   - Disable EKS Auto Mode.
   - Provide a Cluster Name.
   - Create a new recommended IAM role.
   - Provide a VPC and Subnets with Public access.
   - Choose ‘Public and Private’ for cluster endpoint access.
   - Keep the cluster name handy after creation.

2. **Add a Node Group**
   - After the cluster is created, go to the compute tab and add a node group.
   - Provide a name.
   - AMI type: standard.
   - Instance type: m6i.4xl.
   - 500GB EBS storage.
   - Minimum 2, Maximum 5.

## Setup Pre-requisites

Install the following packages from your choice of machine (Cloud VM/local/Windows etc.)
If you are running it via AWS EC2 instance, there will be additional steps to ensure there is connectivity between the EC2 host and the EKS cluster.
The steps below are running on a local MAC laptop. Package install commands and other actions will differ based on machine type (MAC/Linux/Windows etc.)

---

- **Set AWS Credentials**
  - Run command `aws configure`. Enter the access key, secret key, and session token.

- **Install AWS CLI**
  - Follow the instructions at: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

- **Install Helm**
  - Follow the instructions at: https://helm.sh/docs/intro/install/

- **Install KUBECTL/EKSCTL**
  - Follow the instructions at: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
  - Update your Kubeconfig context to the EKS cluster created above.
  - Substitute the region and EKS cluster name before executing the below command:
    ```bash
    aws eks update-kubeconfig --region "$REGION" --name "$CLUSTER_NAME"
    ```

### Setup Default Storage

1. **Check Default Storage Class**
   - Run command `kubectl get sc`.
   - If you do not see a default storage class, run the below command and substitute the storage-class-name from the output:
     ```bash
     kubectl patch storageclass <storage-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
     ```
   - Run `kubectl get sc` again to ensure your storage class has a `(default)` next to it.
   - **Expected Output**
     ```
     gp2 (default)  kubernetes.io/aws-ebs Delete WaitForFirstConsumer false
     ```
     

### Setup Linux Kernel Parameter

1. **Create File `set-vm-max-map-count.yaml`**
   ```
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: set-sysctl
     namespace: kube-system
     labels:
       app: set-sysctl
   spec:
     selector:
       matchLabels:
         app: set-sysctl
     template:
       metadata:
         labels:
           app: set-sysctl
       spec:
         hostPID: true
         containers:
           - name: sysctl
             image: busybox:1.35
             securityContext:
               privileged: true
             command:
               - sh
               - -c
               - |
                 echo "Setting vm.max_map_count=262144 on host..."
                 nsenter --target 1 --mount --uts --ipc --net --pid sysctl -w vm.max_map_count=262144
                 echo "Verifying:"
                 nsenter --target 1 --mount --uts --ipc --net --pid cat /proc/sys/vm/max_map_count
                 sleep 3600
             volumeMounts:
               - name: host-root
                 mountPath: /host
         volumes:
           - name: host-root
             hostPath:
               path: /
    ```

2. **Apply the Configuration**
   - Run command:
     ```bash
     kubectl apply -f set-vm-max-map-count.yaml
     ```

3. **Test the Configuration**
   - Run command:
     ```bash
     kubectl exec -n kube-system -it $(kubectl get pod -n kube-system -l app=set-sysctl -o jsonpath='{.items[0].metadata.name}') -- sysctl vm.max_map_count
     ```
   - **Expected Output**
     ```
     vm.max_map_count = 262144
     ```

### Install EBS CSI Driver

1. **Associate IAM OIDC Provider**
   - Substitute the region and EKS cluster name before executing the below commands:
     ```bash
     eksctl utils associate-iam-oidc-provider --region <<region>> --cluster <<eks-cluster-name>> --approve
     ```

2. **Create IAM Service Account**
   - Run command:
     ```bash
     eksctl create iamserviceaccount --region <<region>> --name ebs-csi-controller-sa --namespace kube-system --cluster <<eks-cluster-name>> --attach-policy-arn arn:aws:iam::aws:policy/AmazonEBSCSIDriverPolicy --approve
     ```

3. **Create EBS CSI Driver Addon**
   - Run command:
     ```bash
     eksctl create addon --name aws-ebs-csi-driver --cluster <<eks-cluster-name>> --region <<region>> --force
     ```

4. **Test the Installation**
   - Run command:
     ```bash
     kubectl get pods -n kube-system | grep ebs
     ```

If this driver is not installed, the postgresql pod does not initialize with the following  error:
```running PreBind plugin "VolumeBinding": binding volumes: context deadline exceeded```

### Create platform value File

- Create the file as is using the link: https://private.docs.cohere.com/docs/install-overview

## PostgreSQL and North Installation
After Pre-requisites are installed, below is the installation process.
Follow steps in https://private.docs.cohere.com/docs/installation-steps. 
To summarize: 

---

#### Authenticate Helm

- You will need the password from a Cohere representative.

#### Create Deployment Namespace

- ‘Cohere’ is the namespace created as per the documentation.

#### Create Secret Key

- Modify the Cohere API key with your production key.

#### Setup PostgreSQL Credentials

- Prepare PostgreSQL Helm.
- Install PostgreSQL.

#### Set up North Helm

1. **Change Default Storage Class**
   - Change the default storage class to the value provided by command `kubectl get sc`.

2. **Modify `setVMMaxMapCount`**
   - Change the value from `false` to `true`: `setVMMaxMapCount: true`.

3. **Install North Helm**
   - Install North Helm.

4. **Confirm Pods are Healthy**
   - Use command `kubectl get pods` or check the pod status through the AWS management portal.

5. **Port Forward with Envoy**

6. **Login to the Platform**
   - Access: http://localhost:8080/admin
   - Username: admin@cohere.com
   - Password: Run command:
     ```bash
     kubectl get secret bootstrap-admin-password -n cohere -o jsonpath="{.data.bootstrapAdminPassword}" | base64 -d | pbcopy
     ```
   - The password will be copied to your clipboard; there will be no output.
  
## Networking: Enable external/public access

This section explains how to configure external access to the Envoy Gateway using a Network Load Balancer (NLB) in AWS and update the Helm values file for your deployment.

---

### 1. Edit Helm values file

Open `myenv.platform.values.yaml` and update the service section as follows:

```
service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```
> **Note:** A Network Load Balancer (NLB) will automatically be created with these settings.

### 2. Upgrade Helm release

Run the Helm upgrade to apply changes and configure the public URL/IP of the load balancer:
```
helm upgrade --install north oci://helm.cohere.com/north/stable/cohere-eno --version "<<version>>" -f platform.values.yaml -f myenv.platform.values.yaml -n cohere
```
### 3. Retrieve External IP / URL

Get the external hostname assigned to the Envoy Gateway service:
```
kubectl get svc -n cohere
```
Look for the `External-IP` of the `cohere-envoy-gateway-xxxxxxx` service.  
It will mostly be a DNS name, for example:
```
a408f5dd68e094f3eaeec70db6d85e1d-ad87f2616d68f41b.elb.us-east-1.amazonaws.com
```
### 4. Update Helm values with External IP / URL

Replace the following placeholders in `myenv.platform.values.yaml` with the External-IP / hostname obtained above:
```
global:
  config:
    dex:
      issuer: "http://<<EXTERNAL-IP-value>>/dex"

toolkit:
  config:
    publicBackendURL: "http://<<EXTERNAL-IP-value>>/api"
    publicFrontendURL: "http://<<EXTERNAL-IP-value>>"
    publicAdminURL: "http://<<EXTERNAL-IP-value>>/admin"

ingress:
  customerDomain: "<<EXTERNAL-IP-value>>" #change from localhost to external-ip-value
```

### 5. Re-apply Helm chart

After updating the file, run:
```
helm upgrade --install north oci://helm.cohere.com/north/stable/cohere-eno --version "<<version>>" -f platform.values.yaml -f myenv.platform.values.yaml -n cohere
```
This ensures that Envoy Gateway routes and the public URLs are properly configured for external access via the NLB.

### 6. Test External Access

Once the deployment is complete, you should be able to reach the application via the ELB hostname:
```
http://<<EXTERNAL-IP-value>>/admin  
http://<<EXTERNAL-IP-value>>/login  
```
---
---

## Install separate Postgres databse and Kubernetes Integration

### Install Postgres via AWS Management Console
- Install Postgres using AWS Management Console. <<provide link and specifics>>
- Update the security group to allow traffic from the EKS cluster (recommended for connectivity).

## Create Databases Using a Temporary Pod Inside Kubernetes
- Run a temporary Postgres client pod inside your cluster:

```
kubectl run -i --tty pg-client --image=postgres --rm --env="PGPASSWORD=YourMasterPassword" -- bash
```

- Connect to the RDS instance:

```
psql -h <<rds-endpoint-name>> \
   -U postgres \
   -d postgres
```

- Create the required databases:

```
CREATE DATABASE north;
CREATE DATABASE compass;
CREATE DATABASE dex;
CREATE DATABASE openfga;
CREATE DATABASE north_admin;
CREATE DATABASE north_tables;
CREATE DATABASE atlas;
CREATE DATABASE inngest;
```
- Exit from the temporary pod
  
### Store Credentials as Kubernetes Secrets
- Create a Kubernetes secret for Postgres password. Here the master username and password is used which was created during database creation.

```
kubectl create secret generic north-db-credentials --from-literal=postgresPassword='<<password>>' 
```

- Kill the previous `postgresql-0` pod if it exists (This was part of the earlier postgreq helm install)

```
kubectl scale statefulset postgresql --replicas=0
kubectl delete pod postgresql-0
```

## Update Helm Configuration
- Update your `myenv.platform.values.yaml` file with the new connections:

```
postgres:
  host: &dbHost "northdb.cgythwufggip.us-east-1.rds.amazonaws.com"
  port: 5432
  user: &dbUser "postgres"
  password: &postgresPassword
    secretKeyRef:
      name: north-db-credentials
      key: postgresPassword
```

- Run the Helm command again after updating the configuration.
```
helm upgrade --install north oci://helm.cohere.com/north/stable/cohere-eno --version "<<NORTH VERSION>>" -f platform.values.yaml -f myenv.platform.values.yaml -n cohere --timeout 15m --wait
```

## Debugging
- Run a debug pod inside your cluster:

```
kubectl run --rm -it pg-client-debug --image=alpine -- sh
```

- Install Postgres client inside the pod:

```
apk add --no-cache postgresql-client
```

- Test Postgres connection and list databases and then list users. This is the user list created through the /admin portal

```
\list
psql -h <<rds-endpoint-name>> -U postgres -d <<DATABASE_NAME>> (Example: DATABSE_NAME: 'north')
select * from users;
```
