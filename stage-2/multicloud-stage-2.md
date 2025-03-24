
# MultiCloud, DevOps & AI Challenge - Stage 2

 In this second stage, I achieve the following:
 - Install docker on workstation
 - Create docker image for CloudMart app
 - Deploy frontend and backend on Kubernetes- EKS

## Docker 

## Step 1 Install Docker on EC2
Run the following commands on the EC2 terminal to install docker:

```sh
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo docker run hello-world
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker
```
![docker installed on ec2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/docker%20installed%20on%20ec2.png)

## Step 2: Create Docker image for CloudMart

### Backend
For the backend, we will create a folder and download the source code:

```sh
mkdir -p challenge-day2/backend && cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip
unzip cloudmart-backend.zip
```
 ![download cloudmart](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/download%20cloudmart.png)
 ![unzip cloudmart](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/unzipt%20cloudmart.png)

Using an editor, create the `.env` file and enter the following content:

```sh
PORT=5000
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=<your-bedrock-agent-id>
BEDROCK_AGENT_ALIAS_ID=<your-bedrock-agent-alias-id>
OPENAI_API_KEY=<your-openai-api-key>
OPENAI_ASSISTANT_ID=<your-openai-assistant-id>
```
![backend .env](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/backend%20.env.png)

Also create the Dockerfile and enter the following content:

```sh
FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```


### Frontend
Create folder and download source code:

```sh
cd ..
mkdir frontend && cd frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip
unzip cloudmart-frontend.zip
```
![download cloudmart frontend](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/download%20cloudmart%20frontend.png)

Create the dockerfile `Dockerfile` and enter the following code:

```sh
FROM node:16-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:16-alpine
WORKDIR /app
RUN npm install -g serve
COPY --from=build /app/dist /app
ENV PORT=5001
ENV NODE_ENV=production
EXPOSE 5001
CMD ["serve", "-s", ".", "-l", "5001"]

```
![frontend dockerfile](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/frontend%20dockerfile.png)

## Kubernetes

## Step 3 Cluster Setup on AWS Elastic Kubernetes Services (EKS)
1. First create a user named `eksuser` with admin privileges. Authenticate it using the following command:(Hint: Create access key on the console)

```sh
aws configure
```

2. Install the cli tool `eksctl`

```sh
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo cp /tmp/eksctl /usr/bin
eksctl version
```

3. Install `kubectl`. This installs the version 1.29, which is a step lower than the 1.30 kubernetes version of AWS

```sh
curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version
```
![kubectl installed 1](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/kubectl%20installed%201.png)

4. Create an EKS Cluster named `cloudmart` with 1 node.

```sh
eksctl create cluster \
  --name cloudmart \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 1 \
  --with-oidc \ #  enables secure IAM authentication for workloads via OIDC.
  --managed
```
 
![cluster provisioned](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/cluster%20provisioned.png)

5. Connect to the EKS cluster using the `kubectl` configuration

```sh
aws eks update-kubeconfig --name cloudmart
```

6. Verify cluster connectivity

```
kubectl get svc
kubectl get nodes
```
![svc and nodes running](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/svc%20and%20nodes%20running.png)

7. Create the role and service account to provide pods access to services used by the application (DynamoDB, Bedrock..)

```sh
eksctl create iamserviceaccount \
  --cluster=cloudmart \
  --name=cloudmart-pod-execution-role \
  --role-name CloudMartPodExecutionRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess\
  --region us-east-1 \
  --approve
```

![service account created](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/service%20account%20created.png)

We applied full administrative access in the above command. In a production environment, you should use more restrictive policies following the least privilege principle. You can create a custom policy with the specific permissions the application needs.

## Step 4 Deployment of the Backend on Kubernetes

1. Create an ECR Repository on the AWS Management Console for the Backend and upload the docker image to it: `Repository name: cloudmart-backend`

![ECR create 1](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/ECR%20create%201.png)
![ECR create 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/ECR%20create%202.png)


2. Navigate to the backend folder, then Click on **view push commands** on the ECR console. 


```sh
cd ../..
cd challenge-day2/backend
```

Follow the steps displayed by ECR to build the docker image
When you click on view push commands, you will see a pop up similar to the below:

![view push commands](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/view%20push%20commands.png)

Here is the image uploaded to the public repo we created on ECR:

![image uploaded to ECR](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/image%20uploaded%20to%20ECR.png)

3. Create a file named `cloudmart-backend.yaml`and enter the code for the kubernetes deployment yaml file for the backend

```sh
cd ../..
cd challenge-day2/backend
vi cloudmart-backend.yaml

```

**content of the backend deployment yaml file**

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-backend-app
  template:
    metadata:
      labels:
        app: cloudmart-backend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-backend-app
        image: public.ecr.aws/g2z2j1n0/cloudmart-backend:latest
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxxxx"
        - name: OPENAI_ASSISTANT_ID
          value: "xxxx"
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-backend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-backend-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```
![backend deploy.yaml](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/backend%20deploy.yaml.png)

copy the image uri from the console and replace the image in the deployment file above with it.

![copy image uri](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/copy%20image%20uri.png)


4. Deploy the backend on kubernetes:
To deploy the backend on kubernetes, run the following command:

```
kubectl apply -f cloudmart-backend.yaml
```
![deploy app backend](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/deploy%20app%20backend.png)

Verify the status of the objects that have been created and get the public IP for the api

```sh
kubectl get pods
kubectl get deployment
kubectl get service
```
![verify backend deployment](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/verify%20backend%20deployment.png)

The "ImagePullBackOff" error seen when we run the `kubectl get pods` usually happens when Kubernetes fails to pull an image from a registry. Verify that the image uri has been replaced correctly and apply the deployment again: `kubectl apply -f cloudmart-backend.yaml`

![container creating](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/container%20creating.png)

![pod running](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/pod%20running.png)

## Step 5 Deployment of the Frontend on Kubernetes

1. Edit the Frontend's `.env` file to point to the API URL created within Kubernetes obtained by the `kubernetes get service` commands

```
cd ../challenge-day2/frontend
vi .env
```

Content to of the frontend `.env`

```sh
VITE_API_BASE_URL=http://<your_url_kubernetes_api>:5000/api

```
![vite url](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/vite%20url.png)

2. Create ECR Repository for the frontend and upload the docker image to it

```
Repository name: cloudmart-frontend
```
![frontend repo](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/frontend%20repo.png)

3. Select the **cloudmart-frontend** repo and choose **view push commands**. Follow the steps displayed by ECR to build the Docker image.

![view push commands for frontend](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/view%20push%20commands%20for%20frontend.png)

![docker image built](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/docker%20image%20built.png)

![docker image tag and push](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/docker%20image%20tag%20and%20push.png)

4. Create a kubernetes deployment file for the frontend

```
vi cloudmart-frontend.yaml
```
Copy the image uri of the frontend image from the ECR console and replace it in the following file

**content of the deployment yaml file**
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-frontend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-frontend-app
  template:
    metadata:
      labels:
        app: cloudmart-frontend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-frontend-app
        image: public.ecr.aws/g2z2j1n0/cloudmart-frontend:latest
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-frontend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-frontend-app
  ports:
    - protocol: TCP
      port: 5001
      targetPort: 5001
```

![frontend deploy-file](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/frontend%20deploy-file.png)

Deploy the frontend on Kubernetes with the following command:

```
kubectl apply -f cloudmart-frontend.yaml
```

Verify the status of the objects that have been created and get the public IP for the api

```sh
kubectl get pods
kubectl get deployment
kubectl get service
```
![Verify frontend deployment](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/Verify%20frontend%20deployment.png)

When we enter the external-ip of both the backend and the frontend mapped to their ports on a browser, we observe that they are accessible:
![frontend running 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/frontend%20running%202.png)

![backend accessible](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/backend%20accessed.png)

I am able to access the admin page and add new products:

![admin page](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-2/images/admin%20page.png)

## Step 6 [Optional] Delete Cluster
If you choose to delete the cluster, at this point, run the following commands:

```sh
kubectl delete service cloudmart-frontend-app-service
kubectl delete deployment cloudmart-frontend-app
kubectl delete service cloudmart-backend-app-service
kubectl delete deployment cloudmart-backend-app

eksctl delete cluster --name cloudmart --region us-east-1
```

But note that, you will have to recreate it for the later stages. This marks the completion of stage 2.
