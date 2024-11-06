# Starbucks Clone Application Deployment on AWS Using a DevSecOps Approach

Welcome to the comprehensive guide for deploying a Starbucks Clone application on AWS using a DevSecOps approach. This README will guide you through setting up the essential tools and configurations for secure and scalable deployment using Jenkins, Docker, Trivy, and Docker Scout.

## Prerequisites

Before we begin, ensure you have:

- **Ubuntu** as your operating system
- **Sufficient privileges** to install and manage packages
- **AWS account** set up for deployments
- **Docker Hub account** for image storage

## Table of Contents

1. [Install AWS CLI](#install-aws-cli)
2. [Install Jenkins on Ubuntu](#install-jenkins-on-ubuntu)
3. [Install Docker on Ubuntu](#install-docker-on-ubuntu)
4. [Install Trivy for Security Scans](#install-trivy-on-ubuntu)
5. [Install Docker Scout](#install-docker-scout)
6. [Deployment Pipeline Stages](#deployment-stages)
7. [Jenkins Pipeline Script](#jenkins-complete-pipeline)
8. [License](#license)

---

### **Install AWS CLI**

First, set up the AWS Command Line Interface (CLI) to interact with AWS services:

```bash
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

### **Install Jenkins on Ubuntu**

Install Jenkins, which will act as our CI/CD tool:

```bash
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```

---

### **Install Docker on Ubuntu**

Docker is essential for containerizing our application:

```bash
# Update packages and install dependencies
sudo apt-get update
sudo apt-get install ca-certificates curl

# Setup Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker and its components
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Add Docker permissions and verify
sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock
newgrp docker
sudo systemctl status docker
```

---

### **Install Trivy on Ubuntu**

Trivy is a comprehensive security scanner for vulnerabilities:

Reference Doc: [Trivy Installation Guide](https://aquasecurity.github.io/trivy/v0.55/getting-started/installation/)

```bash
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

---

### **Install Docker Scout**

Docker Scout helps identify and mitigate vulnerabilities in your images:

```bash
docker login       # Provide your Docker Hub credentials
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s -- -b /usr/local/bin
```

---

## **Deployment Stages**

Below is a high-level overview of the deployment stages:
![Deployment Stages Diagram](https://github.com/whatDeepak/starbucks-clone/blob/main/public/steps.png)

---

### **Jenkins Complete Pipeline**

This Jenkins pipeline script orchestrates the entire deployment process, ensuring code quality, security scans, Docker image building, and deployment.

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }
        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/yeshwanthlm/starbucks.git'
            }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """ $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=starbucks -Dsonar.projectKey=starbucks """
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        stage("OWASP Dependency Check") {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage("Build Docker Image") {
            steps {
                sh "docker build -t starbucks ."
            }
        }
        stage("Tag & Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag starbucks amonkincloud/starbucks:latest"
                        sh "docker push amonkincloud/starbucks:latest"
                    }
                }
            }
        }
        stage("Docker Scout Analysis") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker-scout quickview amonkincloud/starbucks:latest'
                        sh 'docker-scout cves amonkincloud/starbucks:latest'
                        sh 'docker-scout recommendations amonkincloud/starbucks:latest'
                    }
                }
            }
        }
        stage("Deploy to Container") {
            steps {
                sh "docker run -d --name starbucks -p 3000:3000 amonkincloud/starbucks:latest"
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: """
                <html>
                <body>
                    <div style="background-color: #FFA07A; padding: 10px;">
                        <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                    </div>
                    <div style="background-color: #90EE90; padding: 10px;">
                        <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                    </div>
                    <div style="background-color: #87CEEB; padding: 10px;">
                        <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                    </div>
                </body>
                </html>
                """,
                to: 'provide_your_email_id_here',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy.txt'
        }
    }
}
```

---

## **License**

This project is licensed under the [MIT License](https://github.com/whatDeepak/starbucks-clone/blob/main/LICENSE). See the LICENSE file for more details.

---

Feel free to contribute, report issues, or suggest improvements!
