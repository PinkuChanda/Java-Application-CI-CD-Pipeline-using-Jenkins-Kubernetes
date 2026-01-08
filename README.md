# DevOps CI/CD Pipeline for Java Application on Kubernetes

This repository demonstrates a **full DevOps CI/CD pipeline** to build, test, scan, analyze, package, and deploy a simple **Java application** using modern DevOps tools and practices.

The pipeline integrates **Jenkins, SonarQube, Nexus Repository, Docker, and Kubernetes**, and is hosted on **AWS EC2** instances in a production-like setup.

---

## 🚀 Project Summary

This project showcases a real-world DevOps pipeline that takes code from GitHub all the way to deployment on a Kubernetes cluster.

Key features include:

- Automated build and test using **Maven**
- Static code quality analysis with **SonarQube**
- Secret Scanning with **GitLeaks**
- Artifact storage in **Nexus Repository**
- Container creation with **Docker**
- File and Container Security with **Trivy**
- Deployment of containers on a **Kubernetes multi-node cluster**
- Full automation using **Jenkins Pipeline**

---

## 🧱 Architecture & Components

The infrastructure consists of **six AWS EC2 instances**, each responsible for a core service:

| Instance | Service |
|----------|---------|
| EC2-1 | Jenkins Master |
| EC2-2 | SonarQube |
| EC2-3 | Nexus Repository |
| EC2-4 | Kubernetes Master |
| EC2-5 | Kubernetes Worker Node |
| EC2-6 | Kubernetes Worker Node |

This setup simulates a production-like environment with proper separation of responsibilities and high availability.

---

## 📊 Tools & Technologies

| Purpose | Tool |
|---------|------|
| CI/CD Orchestration | Jenkins |
| Code Quality & Security | SonarQube |
| Artifact Management | Nexus Repository |
| Build & Dependency Manager | Maven |
| Containerization | Docker |
| Orchestration | Kubernetes |
| Version Control | Git & GitHub |
| Infrastructure | AWS EC2 |

---

## 🗂️ Jenkins Pipeline (Declarative)


```groovy
pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/PinkuChanda/boardgame-test-project.git'
            }
        }
        
        stage('GitLeaks') {
            steps {
                sh 'gitleaks detect --source . -r gitleaks-report.txt --redact'
            }
        }
        
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        
        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Java-CICD \
                    -Dsonar.projectKey=java-cicd \
                    -Dsonar.java.binaries=target'''
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
        
        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'gmavensettings', jdk: 'jdk17', maven: 'maven3') {
                    sh 'mvn deploy'
                }
            }
        }
        
        stage('Docker Build & Tag') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred') {
                    sh 'docker build -t pinkuchanda/java-cicd:latest .'
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html pinkuchanda/java-cicd:latest'
            }
        }
        
        stage('Docker Push Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred') {
                    sh 'docker push pinkuchanda/java-cicd:latest'
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8s-token', serverUrl: 'https://<k8s-master-ip>:6443') {
                    sh 'kubectl apply -f deployment-service.yaml'
                    sh 'kubectl get all'
                }
            }
        }
    }
}
