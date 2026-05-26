# End-to-End Jenkins CI/CD Pipeline with Dev-to-Prod Promotion

This project demonstrates a complete CI/CD pipeline implementation using Jenkins, Kubernetes, Docker, Kaniko, and AWS ECR.

The main objective of this project was to build a practical DevOps workflow that automates:
- Docker image builds
- Container image publishing
- Kubernetes deployments
- Dev-to-Production promotion
- Approval-based production releases

The application used in this project is a simple Flask app, but the primary focus is the CI/CD pipeline architecture and deployment automation.

---

# Project Architecture

```text
                +----------------------+
                |   Developer Pushes   |
                |      Code to GitHub  |
                +----------+-----------+
                           |
                           v
                +----------------------+
                |       Jenkins        |
                | Kubernetes Build Pod |
                +----------+-----------+
                           |
        +------------------+------------------+
        |                                     |
        v                                     v
+--------------------+           +----------------------+
| Build Docker Image |           | Jenkins Parameters   |
| Using Kaniko       |           | ENV / TAG Handling   |
+----------+---------+           +----------------------+
           |
           v
+-------------------------------+
| Push Image to AWS ECR         |
+---------------+---------------+
                |
                v
+-------------------------------+
| Deploy to Kubernetes DEV      |
| Namespace: test-dev           |
+---------------+---------------+
                |
                v
+-------------------------------+
| Email Approval to Committer   |
+---------------+---------------+
                |
         Manual Approval
                |
                v
+-------------------------------+
| Deploy to Kubernetes PROD     |
| Namespace: test-prod          |
+-------------------------------+
```

---

# Tech Stack

| Tool / Service | Purpose |
|----------------|---------|
| Jenkins | CI/CD automation |
| Kubernetes | Jenkins agents and application deployment |
| Docker | Application containerization |
| Kaniko | Docker image build inside Kubernetes |
| AWS ECR | Container image registry |
| Flask | Sample Python web application |
| SSH Agent | Remote deployment execution |
| Email Extension Plugin | Production approval workflow |

---

# Project Workflow

## 1. Code Push to GitHub

The workflow starts when application code is pushed to the repository.

Jenkins automatically triggers the pipeline.

---

## 2. Jenkins Creates Kubernetes Build Agent

The Jenkins pipeline dynamically creates a Kubernetes pod containing:
- Kaniko container
- Kubectl container
- Jenkins JNLP container

This allows builds to run in isolated and scalable environments.

---

## 3. Docker Image Build Using Kaniko

Kaniko builds the Docker image directly inside Kubernetes without requiring Docker daemon access.

The image is tagged using the Jenkins build number.

Example:

```bash
test-deployment:${BUILD_NUMBER}
```

The image is then pushed to AWS ECR.

---

## 4. Deployment to Development Environment

After the image is successfully built and pushed:
- Jenkins connects to the Kubernetes cluster using SSH
- The deployment image is updated
- The application is deployed to the `test-dev` namespace

Deployment command:

```bash
kubectl set image deployment/test-deployment \
test-deployment=<IMAGE>:${BUILD_NUMBER} \
-n test-dev
```

---

## 5. Approval-Based Production Promotion

Once the application is deployed to DEV:
- Jenkins extracts commit information
- The latest committer receives an approval email
- The email contains:
  - Commit ID
  - Commit message
  - Build number
  - Production deployment approval link

This simulates a real-world release approval process.

---

## 6. Production Deployment

Production deployment only happens when:
- `ENV=prod`
- A valid image tag is provided

The deployment updates the Kubernetes deployment in the `test-prod` namespace.

This prevents accidental production releases.

---

# Jenkins Pipeline Stages

## Build and Publish with Kaniko

This stage:
- Builds the Docker image
- Pushes the image to AWS ECR
- Uses Kaniko for secure container builds inside Kubernetes

Why Kaniko?
- No Docker daemon required
- Safer for Kubernetes environments
- Commonly used in cloud-native CI/CD pipelines

---

## Deploy to Dev

This stage:
- Connects to the Kubernetes environment using SSH
- Updates the deployment image
- Deploys the latest version to the DEV namespace

---

## Email Committer with Approval Link

This stage:
- Reads the latest Git commit information
- Sends approval email to the committer
- Provides a production deployment link

This demonstrates controlled release management.

---

## Deploy to Prod

This stage performs production deployment only after approval.

Safety checks:
- ENV must be `prod`
- Image tag must be provided

This ensures production deployments are intentional and traceable.

---

## Clean Workspace

The workspace is cleaned after pipeline execution to avoid unnecessary file accumulation inside Jenkins agents.

---

# Flask Application

The Flask application is intentionally lightweight because the primary focus of this project is CI/CD pipeline implementation and deployment automation.

## app.py

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Flask inside Docker using jenkins trigger and SNS notification !!!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

# Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt requirements.txt

RUN pip install -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
```

---

# Dockerfile Explanation

## Base Image

```dockerfile
FROM python:3.10-slim
```

Uses a lightweight Python image to reduce container size.

---

## Working Directory

```dockerfile
WORKDIR /app
```

Sets the application working directory inside the container.

---

## Dependency Installation

```dockerfile
COPY requirements.txt requirements.txt

RUN pip install -r requirements.txt
```

Copies and installs Python dependencies.

---

## Application Copy

```dockerfile
COPY app.py .
```

Copies the Flask application into the container.

---

## Application Startup

```dockerfile
CMD ["python", "app.py"]
```

Starts the Flask application.

---

# How to Run Locally

## Clone Repository

```bash
git clone https://github.com/your-username/jenkins-cicd-dev-to-prod-pipeline.git
```

---

## Build Docker Image

```bash
docker build -t flask-cicd-app .
```

---

## Run Container

```bash
docker run -p 5000:5000 flask-cicd-app
```

---

## Access Application

```text
http://localhost:5000
```

---
# Key Learnings

This project helped in understanding:
- Jenkins declarative pipelines
- Kubernetes-based Jenkins agents
- Docker image automation
- Secure image builds using Kaniko
- AWS ECR integration
- Kubernetes deployments
- Environment-based release workflows
- Approval-driven production deployments

---

# Conclusion

This project demonstrates a practical implementation of a cloud-native CI/CD workflow using Jenkins and Kubernetes.

The main focus was not application complexity, but building a reliable DevOps pipeline capable of:
- automated builds,
- container image publishing,
- Kubernetes deployments,
- and controlled production release management.

The workflow closely resembles real-world deployment pipelines used in modern DevOps environments.