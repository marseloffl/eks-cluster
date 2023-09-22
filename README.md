# EKS Cluster Setup, Deployment & Load Balancer Service and Autoscaling Pods.

## Prerequisite :
1. AWS Account
2. IAM Role [AmazonEKSClusterPolicy]
3. IAM Role [EC2,VPC,EKS,Cloudformation Full Access]
5. IAM User
6. EC2 Instance with [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) and [eksctl](https://eksctl.io/introduction/#installation) installed

Note: AWS is gonna charge us for this demo/project. So create and clean it everything ASAP.

## Step 1. Creating EKS Cluster
I have taken Ubuntu (t2.micro) as a client for demo purpose to connect to the EKS cluster control plane. Attach second IAM Role to this EC2 Machine. AWS Cli installed and configured. kubectl and eksctl also installed and ready to create a EKS Cluster. Run this below command
```
eksctl create cluster --name demo-cluster --region ap-south-1 \
  --node-ami-family Ubuntu2004 \
  --node-type t2.small \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2 \
  --ssh-access \
  --ssh-public-key demo-keypair
``` 
This will create EKS Cluster control plane and Node Group.

- To [Create Cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) alone, then run this command:
```
eksctl create cluster --name my-cluster --region region-code 
```
- To [Create Node Group](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html) alone, then run this command:
```
eksctl create nodegroup \
  --cluster my-cluster \
  --region region-code \
  --name my-mng \
  --node-ami-family ami-family \
  --node-type t2.small \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 4 \
  --ssh-access \
  --ssh-public-key my-key
```
No need to provide all informations, we can remove which we don't need. The default desired node, min and max node will be 2.

- It's gonna take 10-15 Mins to create New VPC, EKS Cluster and Node Group(EC2 Instance) with auto scaling.

![CloudFormation](https://drive.google.com/uc?export=view&id=1UflallSQ-W93tbJyh5E0zRCSkSYdwhGQ)

![Cluster](https://drive.google.com/uc?export=view&id=1n5K3zNo4IPhGVrzHtbhHmANE0uRgyZhD)

![NodeGroup](https://drive.google.com/uc?export=view&id=1PBXeR86ziWeG_RM8aecEBHWJh1k74D0v)

![AutoScaling](https://drive.google.com/uc?export=view&id=1aVyOne4G27DDH8KsippiQ8DVX39wedUE)

![NodeInstance](https://drive.google.com/uc?export=view&id=13LyJLGoDeunzZm30lIR42bkr6gOLC3eY)

## 1.1 Creating Cluster using YAML File
- First create ```cluster.yaml``` file
```
sudo vi cluster.yaml
```
- Then copy paste this config file and changes it according to your configuration
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: demo-cluster
  region: region-code

nodeGroups:
  - name: my-mng
    instanceType: InstanceType
    desiredCapacity: 2
    volumeSize: 20
    ssh:
      allow: true # will use ~/.ssh/id_rsa.pub as the default ssh key
```
Save and quit
- Execute this config file using this command
```
eksctl create cluster -f cluster.yaml
```

## Step 2. Deployment & Service creation
Now the cluster is created with node group and all configurations are done through cloud formation.
To access the cluster from client machine execute the below command
```
aws eks update-kubeconfig --region region-code --name my-cluster
```
- Now lets deploy a sample app php apache web server through yaml file.
```
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
```
```deployment.apps/php-apache created```\
```service/php-apache created```

Now the pod is created and running. To check the pod running execute this command
```
kubectl get pods
```
![Php-apachePod](https://drive.google.com/uc?export=view&id=1EsdniwZk_nYZpihcSHY6K5KbF6ekYPUA)


## Step 3. Horizontal Pod Autoscaler.
- Pod is created now let's create Horizontal Pod Autoscaler for our pod. Before that we have to install metrics server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
To verify Metrics server is running or not, execute this command
```
kubectl get deployment metrics-server -n kube-system
```
Now execute this command to check the cpu usage of our node 
```
kubectl top nodes
```
- Ok Done, now let's create HPA.
```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```
Let me explain this command, if the CPU Usage increase above 50% it will create new pods maximum till 10.
- Creating HPA Using YAML file:

Create ```HPA.yaml``` file using this command
```
sudo vi HPA.yaml
```
Then copy paste this config file
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```
Execute this file using this command
```
kubectl create -f HPA.yaml
```
- Lets check whether HPA is created or not, using this command
```
kubectl get hpa
```
![HPA](https://drive.google.com/uc?export=view&id=1c15H7_s3ZpGi9mt-_YrfCUeE1plPBQWm)
- Now lets increase the CPU threshold with this command
```
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/bash -c "while sleep 0.01; do wget -q -0- http://php-apache; done"
```
![CPUIncrease](https://drive.google.com/uc?export=view&id=1sTj8N2k13o73KzkpjfWD_6vgu0xenX9R)
It increases the cpu usage and it create new pods automatically, let's check that with this command
```
kubectl get hpa -w
```
![NewPodCreation](https://drive.google.com/uc?export=view&id=13QtY4z1w0zRxg8EunaC_ugVu0y1hJFig)
- If we stop the cpu usage with ```ctrl+c``` then our newly created pods will terminated automatically

![PodsTerminating](https://drive.google.com/uc?export=view&id=1RYYitI07j1NkjdsMf2Xz5_kTHUGRqOb3)

## Step 4. Load Balancer
- Before creating a load balancer service, let's check the running pods, replica set, deployment and services created using this command
```
kubectl get all
```
![GetAll](https://drive.google.com/uc?export=view&id=1XYYrfsT25STYoBbHzCC3Eud0p4la4Yml)

- Lets expose our deployment and create a load balancer
```
kubectl expose deployment php-apache --type=LoadBalancer --name=php-service
```
- Check the newly created particular service using this command
```
kubectl get service php-service
```
![CreateLoadBalancer](https://drive.google.com/uc?export=view&id=1BlULalxD5ZraCsmpES3qiq3QKtkC5Co7)
- To check all services running

![CheckSVC](https://drive.google.com/uc?export=view&id=1-AYoUoMkVtQIs6Bxm_JyDeDmn-0loFlC)

- It creates Load Balancer and external IP to access our deployment by outsider.

![LoadBalancer](https://drive.google.com/uc?export=view&id=1vYpQlY7WPvgVufIJzBTa6wwJ6JoMW0f0)

- Copy and paste the external IP on the browser. Yes it's accessable to anyone with this IP. We can map this IP on route53 with new domain.

![Deployed](https://drive.google.com/uc?export=view&id=1JOnaRhygduuHu1HiBf6_iqxUYZU68Vuu)

## FAQ
### 1. Can we create everything with commands?
Yes. You can also use YAML file for cluster creation, hpa, deployment etc. We can also execute the files with Github link using commands. I also uploaded those files to this repo.

### 2. How to rectify the errors?
The chances for error occuring are due to improper IAM Policy or any network issues. If the cluster or node group didn't create properly and failed, then try once again after checking the IAM role.

### 3. Minimum requirements for node group?
Client machine can be ```t2.micro``` but Node instance must be atleast 2GB of ram. I used ```t2.small``` for demo purpose and i have created only one node to reduce cost. Ofcourse the performance will be very less if you're deploying heavy configured app. So you can try ```t2.medium``` or other better config instances. But be careful and provide instance type before node creation.

### 4. How much does it cost for this project?
AWS gonna charge us for Creating new VPC with NAT gateway and elastic Ip, EKS Cluster, Non Free tier instance everything. I have spend around 1-2$ for this project/demo. So once the practice work is completed you have to delete everything.