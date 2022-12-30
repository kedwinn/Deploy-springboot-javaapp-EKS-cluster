# Deploy-springboot-javaapp-EKS-cluster

# Overview

In this project, set up a CICD to automate the different builds: pipeline, docker image creation, docker image upload to ECR and deployment of our Springboot java-app into EKS clusters using Jenkins, sonarqube, slack, docker, Helm, AWS EKS and for that continuous feedback loop, we use prometheus and grafana using helmchart to visualise the different metrics cluster, nodes, podes memory and cpu usage. Takeaways from the project, trouble shoot server config, cluster errors on the flight like Error: getting availability zones,UnauthorizedOperation, Error establishing SSH connection. configure the the aws cli and once again exposure. Overview: developers commit code to github, jenkins fetches the code and with the help of maven buildcycle, builds the jar file and docker image pushed to sonarqube/checkstyle for code analysis(vulnerability, code smell, syntax via unit, quality gate, integration test) and if successful, pushed to ECR for further code test and the deployed into EKS cluster.

#### Prequisities
## 1. Install Jenkins and Maven
sudo apt update
sudo apt-get install default-jdk -y: installs Java
java -version: verifies the java version
sudo apt install maven -y: installs maven
mvn -v:  shows you the maven version
Add Repository key to the system
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
Append debian package repo address to the system
echo deb http://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list
Update Ubuntu package
sudo apt update
Install Jenkins
sudo apt install jenkins -y

## 2. Installs the CLI
Follow these steps from the command line to install the AWS CLI on Linux.
sudo curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 
sudo apt install unzip
sudo unzip awscliv2.zip  
sudo ./aws/install
aws --version
sudo apt install awscli

## 3. Installs eksctl on Ubuntu
eksctl is a command line tool for working with EKS clusters that automates many individual tasks. The eksctl tool uses CloudFormation under the hood, creating one stack for the EKS master control plane and another stack for the worker nodes.
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
Move the extracted binary to /usr/local/bin. 
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

## 4. Install kubectl on Ubuntu Instance 
Kubernetes uses a command line utility called kubectl for communicating with the cluster API server. It is tool for controlling Kubernetes clusters. kubectl looks for a file named config in the $HOME directory.
sudo curl --silent --location -o /usr/local/bin/kubectl   https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl 
Verify if kubectl got installed
kubectl version --short --client
 
## 5. Switch as the Jenkins-user using 
sudo su - jenkins

## 6. Create the EKS Cluster
 ### Create EKS Cluster with two worker nodes using eksctl

eksctl create cluster --name tomcat-eks --region eu-west-2 --nodegroup-name my-nodes --node-type t3.small --managed --nodes 2 

The above command should create a EKS cluster in AWS, it might take 15 to 20 mins. The eksctl tool uses CloudFormation under the hood, creating one stack for the EKS master control plane and another stack for the worker nodes. 

eksctl get cluster --name demo-eks --region us-east-2 

### This should confirm that EKS cluster is up and running.

### Update Kube config by entering below command:

aws eks update-kubeconfig --name demo-eks --region us-east-2

kubeconfig file be updated under /var/lib/jenkins/.kube folder.

### you can view the kubeconfig file by entering the below command:

cat  /var/lib/jenkins/.kube/config

## 7. Connect to EKS cluster using kubectl commands
### To view the list of worker nodes as part of EKS cluster.

kubectl get nodes

kubectl get ns

### Deploy Nginx on a Kubernetes Cluster
Let us run some apps to make sure they are deployed to Kubernetes cluster. The below command will create deployment:

kubectl create deployment nginx --image=nginx

### View Deployments
kubectl get deployments

### Delete EKS Cluster using eksctl

eksctl delete cluster --name demo-eks --region us-east-2 

the above command should delete the EKS cluster in AWS, it might take a few mins to clean up the cluster.

Errors during Cluster creation, reconfigure your aws credentials using aws configure  and pass your access and scret keys and double the EKS role
If you are having issues when creating a cluster, try to delete the cluster by executing the below command and re-create it.
eksctl delete cluster --name demo-eks --region us-east-2 

## 8. Install docker on the Jenkin-Mavin server

### Docker installation steps using default repository from Ubuntu
### Update local packages by executing below command:
sudo apt-get update
 
### Install docker
sudo apt install docker.io -y

### Add Ubuntu user to Docker group
sudo usermod -aG docker $USER

### exit and relog for ubuntu to perform some docker commandssudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker

### Add jenkins user to Docker group
sudo usermod -a -G docker jenkins

### Restart Jenkins service
sudo service jenkins restart

### Reload system daemon files
sudo systemctl daemon-reload

### Restart Docker service as well
sudo service docker stop
sudo service docker start

## Modify the  pipeline code using the script below
Make sure you change red highlighted values below as per your settings: Your docker user id should be updated with your registry credentials ID from Jenkins from step # 1 should be copied

pipeline {
   tools {
        maven 'Maven3'
    }
    agent any
    environment {
        registry = "account_id.dkr.ecr.us-east-2.amazonaws.com/my-docker-repo"
    }
   
    stages {
        stage('Cloning Git') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/akannan1087/springboot-app']]])     
            }
        }
      stage ('Build') {
          steps {
            sh 'mvn clean install'           
            }
      }
    // Building Docker images
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry 
        }
      }
    }
   
    // Uploading Docker images into AWS ECR
    stage('Pushing to ECR') {
     steps{  
         script {
                sh 'aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin account_id.dkr.ecr.us-east-2.amazonaws.com'
                sh 'docker push account_id.dkr.ecr.us-east-2.amazonaws.com/my-docker-repo:latest'
         }
        }
      }

       stage('K8S Deploy') {
        steps{   
            script {
                withKubeConfig([credentialsId: 'K8S', serverUrl: '']) {
                sh ('kubectl apply -f  eks-deploy-k8s.yaml')
                }
            }
        }
       }
    }
}

## Prerequisites for Prometheus and Grafana
make sure EKS Cluster is running

### Install Helm on the EC2 instance to access EKS cluster
#### Download scripts 
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

#### Provide permission
sudo chmod 700 get_helm.sh

#### Execute script to install
sudo ./get_helm.sh

#### Verify installation
helm version --client

#### We need to add the Helm Stable Charts for your local client. Execute the below command:

$ helm repo add stable https://charts.helm.sh/stable

#### Add prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm search repo prometheus-community

OR

Prometheus and grafana helm chart moved to kube prometheus stack

### Create Prometheus namespace
kubectl create namespace prometheus

#### Below is helm command to install kube-prometheus-stack. The helm repo kube-stack-prometheus (formerly prometheus-operator) comes with a grafana deployment embedded.

helm install stable prometheus-community/kube-prometheus-stack -n prometheus

#### Lets check if prometheus and grafana pods are running already
kubectl get pods -n prometheus
kubectl get svc -n prometheus

##### This confirms that prometheus and grafana have been installed successfully using Helm.

## In order to make prometheus and grafana available outside the cluster, use load balancer or NodePort.
#### Edit Prometheus Service
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus

#### Edit Grafana Service
kubectl edit svc stable-grafana -n prometheus

#### Verify if service is changed to LoadBalancer and also to get the Load Balancer URL.
kubectl get svc -n prometheus

#### Access Grafana UI in the browser
Get the URL from the above screenshot and put in the browser

UserName: admin 
Password: prom-operator

## Create Dashboard in Grafana

#### In Grafana, we can create various kinds of dashboards as per our needs.
How to Create Kubernetes Monitoring Dashboard?
For creating a dashboard to monitor the cluster:

Click '+' button on left panel and select ‘Import’.

Enter 12740 dashboard id under Grafana.com Dashboard.

Click ‘Load’.

Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.

Click ‘Import’.

#### This will show monitoring dashboard for all cluster nodes
### How to Create Kubernetes Cluster Monitoring Dashboard?

For creating a dashboard to monitor the cluster:

Click '+' button on left panel and select ‘Import’.

Enter 3119 dashboard id under Grafana.com Dashboard.

Click ‘Load’. Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.

Click ‘Import’.

#### This will show monitoring dashboard for all cluster nodes

## Create POD Monitoring Dashboard
For creating a dashboard to monitor the cluster:

Click '+' button on left panel and select ‘Import’.

Enter 6417 dashboard id under Grafana.com Dashboard.

Click ‘Load’.

#### Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.

Click ‘Import’.
This will show monitoring dashboard for all cluster nodes.







