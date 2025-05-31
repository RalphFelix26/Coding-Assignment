# Deploy Python Flask App to AWS EKS Using CI/CD

This project demonstrates how to deploy a Python Flask web application to an AWS EKS (Elastic Kubernetes Service) cluster using a complete CI/CD pipeline with Jenkins, Docker, Terraform, and Kubernetes.

---

## ✅ Overview

### Components Used:
- **Flask**: Python web framework
- **Docker**: Containerization
- **Terraform**: Infrastructure as Code to provision EKS, VPC, and ECR
- **EKS**: Kubernetes Cluster on AWS
- **Jenkins**: CI/CD Pipeline Tool
- **Kubernetes YAMLs**: To deploy the Flask app in EKS

---

## 🛠️ Prerequisites
- AWS Account
- Jenkins Server (on EC2 or local)
- AWS CLI configured
- Docker installed
- Terraform installed
- kubectl & eksctl installed

---

## 📁 Project Structure
```
flask-app-eks/
├── app.py
├── Dockerfile
├── requirements.txt
├── deployment.yaml
├── service.yaml
├── Jenkinsfile
└── terraform/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

---

## 🚀 Step-by-Step Setup Guide

### 1. Flask Application Setup
```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return "Hello from Flask on EKS!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
### 2. requirements.txt
```requirements.txt
flask
gunicorn
```

### 3. Dockerize the App
```dockerfile
# Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

Build & test locally:
```bash
docker build -t flask-app .
docker run -p 5000:5000 flask-app
```

---

### 4. Push Docker Image to ECR
```bash
aws ecr create-repository --repository-name flask-app
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com

docker tag flask-app:latest <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/flask-app
docker push <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/flask-app
```

---

### 5. Provision AWS Infrastructure with Terraform
```hcl
# main.tf (inside terraform/)
provider "aws" {
  region = "us-east-1"
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "flask-cluster"
  cluster_version = "1.28"
  subnets         = ["subnet-xxxx", "subnet-yyyy"]
  vpc_id          = "vpc-xxxxxx"

  node_groups = {
    flask_nodes = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1
      instance_type    = "t3.medium"
    }
  }
}
```

Run:
```bash
cd terraform
terraform init
terraform apply
```

---

### 6. Connect to EKS Cluster
```bash
aws eks --region us-east-1 update-kubeconfig --name flask-cluster
kubectl get nodes
```

---

### 7. Kubernetes Deployment
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
        - name: flask
          image: <ecr_image_url>
          ports:
            - containerPort: 5000
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: LoadBalancer
  selector:
    app: flask
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

Apply:
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

### 8. Jenkins Pipeline (Jenkinsfile)
```groovy
pipeline {
    agent any

    environment {
        ECR_REPO = '<aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/flask-app'
        AWS_REGION = 'us-east-1'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your/flask-app-eks.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app .'
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
                docker tag flask-app:latest $ECR_REPO
                docker push $ECR_REPO
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                aws eks --region $AWS_REGION update-kubeconfig --name flask-cluster
                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml
                '''
            }
        }
    }
}
```

---

## ✅ Result
- Access your application via the Load Balancer URL from:
```bash
kubectl get svc flask-service
```

---

## 📝 License
This project is licensed under the MIT License.

---
