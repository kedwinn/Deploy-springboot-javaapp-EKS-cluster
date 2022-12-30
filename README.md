# Deploy-springboot-javaapp-EKS-cluster

# Overview

In this project, we set up a CICD to automate the different builds: pipeline, docker image creation, docker image upload to ECR and deployment of our Springboot java-app into EKS clusters using Jenkins, sonarqube, slack, docker, Helm, AWS EKS and for that continuous feedback loop, we use prometheus and grafana using helmchart to visualise the different metrics cluster, nodes, podes memory and cpu usage. Takeaways from the project, trouble shoot server config, cluster errors on the flight and once again exposure. Overview: developers commit code to github, jenkins fetches the code and with the help of maven buildcycle, builds the jar file and docker image pushed to sonarqube/checkstyle for code analysis(vulnerability, code smell, syntax via unit, quality gate, integration test) and if successful, pushed to ECR for further code test and the deployed into EKS cluster.
