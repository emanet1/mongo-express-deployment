# mongo-express-deployment

Deployment of MongoDB on kubernetes

## Part 1 - Installing kubectl and eksctl on Amazon Linux 2

### Install git, helm and kubectl

- Launch an AWS EC2 instance of Amazon Linux 2 AMI (type t2.micro) with security group allowing SSH.

- Connect to the instance with SSH.

- Update the installed packages and package cache on your instance.

```bash
sudo yum update -y
```

- Install git.

```bash
sudo yum install git
```

- Install helm.

```bash
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version --short
```

- Download the Amazon EKS vended kubectl binary.

```bash
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl
```

- Apply execute permissions to the binary.

```bash
chmod +x ./kubectl
```

- Copy the binary to a folder in your PATH. If you have already installed a version of kubectl, then we recommend creating a $HOME/bin/kubectl and ensuring that $HOME/bin comes first in your $PATH.

```bash
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
```

- (Optional) Add the $HOME/bin path to your shell initialization file so that it is configured when you open a shell.

```bash
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

- After you install kubectl , you can verify its version with the following command:

```bash
kubectl version --short --client
```

### Install eksctl

- Download and extract the latest release of eksctl with the following command.

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

- Move the extracted binary to /usr/local/bin.

```bash
sudo mv /tmp/eksctl /usr/local/bin
```

- Test that your installation was successful with the following command.

```bash
eksctl version
```

## Part 2 - Creating the Kubernetes Cluster on EKS

- If needed create ssh-key with commnad `ssh-keygen -f ~/.ssh/id_rsa`

- Configure AWS credentials.

```bash
aws configure
```

- Create an EKS cluster via `eksctl`. It will take a while.

```bash


eksctl create cluster --region us-east-1  --zones us-east-1a,us-east-1b,us-east-1c --node-type t2.medium --nodes 1 --nodes-min 1 --nodes-max 2 --name mycluster

```

- Note that the default value for node-type is m5.large.

```bash
$ eksctl create cluster --help
```

- Show the aws `eks service` on aws management console and explain `eksctl-mycluster-cluster` stack on `cloudformation service`.


# Part 3 - Install Nginx Ingress Controller Kubernetes using Helm

## Step 1: Install helm 3 in our workstation

* Install helm 3 in your Workstation

```bash
cd ~/
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

* Let’s query for helm package version to validate working installation:

```bash
helm version
```
# Part 4 -  Deploy Nginx Ingress Controller

* Download latest stable release of Nginx Ingress Controller code:

```bash
controller_tag=$(curl -s https://api.github.com/repos/kubernetes/ingress-nginx/releases/latest | grep tag_name | cut -d '"' -f 4)
```

```bash
wget https://github.com/kubernetes/ingress-nginx/archive/refs/tags/${controller_tag}.tar.gz
```
* Extract the file downloaded:

```bash
tar xvf ${controller_tag}.tar.gz
```

* Switch to the directory created:

```bash
cd ingress-nginx-${controller_tag}
```

* Change your working directory to charts folder:

```bash
cd charts/ingress-nginx/
```

### Label nodes that will run Ingress Controller Pods
The node selector is used when we have to deploy a pod or group of pods on a specific group of nodes that passed the criteria defined in the configuration file.

* List nodes:

```bash
kubectl get nodes
```
* Add label <mark>runingress=nginx</mark>

```bash
kubectl label node ip-192-168-69-2.ec2.internal runingress=nginx
```

* Show labels set

```bash
 kubectl describe node ip-192-168-69-2.ec2.internal
 ```

 ### Update values.yml file to change parameters


 * Edit values file

```bash
vim values.yaml
```
* Add tolerations for Master nodes

```bash
controller:
  tolerations:
    - key: node-role.kubernetes.io/master
      operator: Equal
      value: "true"
      effect: NoSchedule
    - key: node-role.kubernetes.io/master
      operator: Equal
      effect: NoSchedule
```

Setcontroller.service.externalIPs

```bash
controller:
  service:
    externalIPs: ["192.168.42.245","192.168.42.246"]
```


* To set number of replicas of the Ingress controller deployment on controller.replicaCount

```bash
controller:
  replicaCount: 1
```
* If using node selector for pod assignment for the Ingress controller pods set on 

controller.nodeSelector


```bash
controller:
  nodeSelector:
    kubernetes.io/os: linux
    runingress: "nginx"
```

* Create namespace

```bash
kubectl create namespace ingress-nginx
```

* Now deploy Nginx Ingress Controller using the following commands

```bash
helm install -n ingress-nginx ingress-nginx  -f values.yaml .
```


* Check status of all resources iningress-nginx namespace:

```bash
kubectl get all -n ingress-nginx
```

* Pods

```bash
 kubectl get pods -n ingress-ngin
 ```

*  To check logs in the Pods use the commands:

```bash
kubectl -n ingress-nginx  logs deploy/ingress-nginx-controller
```

* To follow logs as they stream run:

```bash
kubectl -n ingress-nginx  logs deploy/ingress-nginx-controller -f
```

## Deploy Application and expose using Nginx Ingress


* The Ingress definition method can also be viewed using explain command option:

```bash
kubectl explain ingress
```


### Deploy sample application and expose using ingress

* Crate a temporary namespace called demo

```bash
kubectl create namespace demo
```

* Create MongoDb-deployment & service object YAML file.

```bash
cd ~/ && 
vim mondb-deployment.yaml

```


```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

* Create MongoDB-Deployment 

```bash
kubectl create -f mondb-deployment.yaml
```
* check if Pods are running and deployed

```bash
kubectl get po -n demo
```

* Create ConfigMap for our MongoDB & Mongo-Express Deployment 

```bash
vi mongo-configmap.yaml
```

apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service


* Apply ConfigMap 

```bash
kubectl create -f mongo-configmap.yaml -n demo
```

* Create Secret 

```bash
vi mongo-secret.yaml
```

apiVersion: v1
kind: Secret
metadata:
    name: mongodb-secret
type: Opaque
data:
    mongo-root-username: dXNlcm5hbWU=
    mongo-root-password: cGFzc3dvcmQ=


* Apply secret Created

```bash
kubectl create -f mongo-secret.yaml -n demo
```

* create mongo-express deployment

```bash
vi mongo-express.yaml
```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom: 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom: 
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer  
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      nodePort: 30000

* Apply mongo-express deployment


```bash
kubectl create -f mongo-express.yaml -n demo
```
