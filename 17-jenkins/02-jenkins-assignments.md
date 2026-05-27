# Jenkins — Assignments

## Assignment 1: Jenkins Installation and First Pipeline

**Tasks:**
1. Install Jenkins using Docker
2. Complete the initial setup wizard
3. Install recommended plugins
4. Create your first Freestyle job that runs: `echo "Hello Jenkins" && date && uptime`
5. Convert the Freestyle job to a Pipeline job with this Jenkinsfile:

```groovy
pipeline {
    agent any
    stages {
        stage('Hello') {
            steps {
                echo "Hello from Jenkins!"
                sh "echo 'Running on: $(hostname)'"
                sh "date"
            }
        }
        stage('Environment Info') {
            steps {
                sh """
                    echo "User: \$(whoami)"
                    echo "Java: \$(java -version 2>&1 | head -1)"
                    echo "Docker: \$(docker --version)"
                """
            }
        }
    }
    post {
        always {
            echo "Pipeline complete. Status: ${currentBuild.result}"
        }
    }
}
```

---

## Assignment 2: Build a Python Application Pipeline

**Application code (create a simple Python app):**

```python
# app.py
def add(a, b):
    return a + b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

```python
# test_app.py
import pytest
from app import add, divide

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0

def test_divide():
    assert divide(10, 2) == 5.0

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)
```

**Build a Jenkinsfile that:**
1. Checks out code from Git
2. Runs `pip install -r requirements.txt`
3. Runs `pytest --junitxml=test-results.xml`
4. Publishes test results with JUnit plugin
5. Archives the test results artifact
6. Sends build status to Slack (mock it if no Slack)
7. Cleans workspace on completion

---

## Assignment 3: Multi-Branch Pipeline for Feature Development

**Setup:**
1. Create a GitHub repository with two branches: `main` and `feature/new-feature`
2. Create a Multibranch Pipeline in Jenkins
3. Configure branch discovery from GitHub

**Jenkinsfile with branch-specific behavior:**

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'echo "Building..."' }
        }
        stage('Test') {
            steps { sh 'echo "Testing..."' }
        }
        stage('Deploy to Staging') {
            when { branch 'feature/*' }
            steps { echo "Deploying to staging (feature branch)" }
        }
        stage('Deploy to Production') {
            when { branch 'main' }
            input {
                message "Deploy to production?"
                ok "Deploy Now"
            }
            steps { echo "Deploying to production!" }
        }
    }
}
```

**Tasks:**
1. Push Jenkinsfile to both branches
2. Verify Jenkins auto-discovers both branches
3. Trigger builds on both branches
4. Verify staging deploys for feature branch
5. Verify production requires approval on main

---

## Assignment 4: Docker + Jenkins Pipeline

**Build a complete Docker build and push pipeline:**

```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY = 'registry.hub.docker.com'
        IMAGE_NAME = 'your-dockerhub-username/myapp'
        DOCKER_CREDENTIALS = credentials('dockerhub-creds')
    }
    
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    // Build with build args
                    docker.build("${IMAGE_NAME}:${BUILD_NUMBER}", 
                        "--build-arg BUILD_DATE=${new Date().format('yyyy-MM-dd')} .")
                }
            }
        }
        
        stage('Test Container') {
            steps {
                script {
                    // Start container and test
                    sh """
                        docker run -d --name test-${BUILD_NUMBER} \
                            -p 8081:8080 ${IMAGE_NAME}:${BUILD_NUMBER}
                        sleep 5
                        curl -f http://localhost:8081/health || exit 1
                        docker stop test-${BUILD_NUMBER}
                        docker rm test-${BUILD_NUMBER}
                    """
                }
            }
        }
        
        stage('Push to Registry') {
            when { branch 'main' }
            steps {
                sh "echo ${DOCKER_CREDENTIALS_PSW} | docker login -u ${DOCKER_CREDENTIALS_USR} --password-stdin"
                sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }
    }
    
    post {
        always {
            sh "docker rmi ${IMAGE_NAME}:${BUILD_NUMBER} || true"
        }
    }
}
```

**Tasks:**
1. Create a simple Node.js or Python app with Dockerfile
2. Create Docker Hub credentials in Jenkins
3. Build and run the pipeline
4. Verify image appears in Docker Hub

---

## Assignment 5: Jenkins Shared Library

**Create a shared library:**

```
jenkins-library/
├── vars/
│   ├── deployToServer.groovy
│   ├── runTests.groovy
│   └── sendAlert.groovy
└── src/
    └── com/company/
        └── Deploy.groovy
```

```groovy
// vars/deployToServer.groovy
def call(Map config = [:]) {
    def server = config.server ?: error("server required")
    def appName = config.appName ?: error("appName required")
    def version = config.version ?: error("version required")
    
    echo "Deploying ${appName}:${version} to ${server}"
    
    sshagent(['ssh-credentials']) {
        sh """
            ssh -o StrictHostKeyChecking=no ubuntu@${server} '
                docker pull myregistry/${appName}:${version}
                docker stop ${appName} || true
                docker rm ${appName} || true
                docker run -d --name ${appName} myregistry/${appName}:${version}
            '
        """
    }
    
    // Health check
    timeout(time: 2, unit: 'MINUTES') {
        waitUntil {
            def result = sh(
                script: "ssh ubuntu@${server} 'curl -sf http://localhost:8080/health'",
                returnStatus: true
            )
            return result == 0
        }
    }
    
    echo "Deployment successful!"
}
```

```groovy
// Jenkinsfile using the library
@Library('jenkins-library@main') _

pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                deployToServer(
                    server: '10.0.1.10',
                    appName: 'myapi',
                    version: "${BUILD_NUMBER}"
                )
            }
        }
    }
}
```

---

## Assignment 6: Jenkins with AWS ECS Deployment

**Full CI/CD pipeline deploying to AWS ECS:**

```groovy
pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '123456789.dkr.ecr.us-east-1.amazonaws.com'
        APP_NAME = 'myapp'
        ECS_CLUSTER = 'production'
        ECS_SERVICE = 'myapp-service'
    }
    
    stages {
        stage('Build') { steps { sh 'mvn clean package -DskipTests' } }
        stage('Test') { steps { sh 'mvn test' } }
        
        stage('Docker Build & Push to ECR') {
            steps {
                withAWS(credentials: 'aws-credentials', region: AWS_REGION) {
                    sh """
                        aws ecr get-login-password | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        
                        docker build -t ${APP_NAME}:${BUILD_NUMBER} .
                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${ECR_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                        docker push ${ECR_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }
        
        stage('Deploy to ECS') {
            steps {
                withAWS(credentials: 'aws-credentials', region: AWS_REGION) {
                    sh """
                        # Update task definition with new image
                        TASK_DEF=\$(aws ecs describe-task-definition \
                            --task-definition ${APP_NAME} \
                            --query taskDefinition)
                        
                        NEW_TASK_DEF=\$(echo \$TASK_DEF | jq \
                            '.containerDefinitions[0].image = "${ECR_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"')
                        
                        aws ecs register-task-definition \
                            --cli-input-json "\$NEW_TASK_DEF"
                        
                        aws ecs update-service \
                            --cluster ${ECS_CLUSTER} \
                            --service ${ECS_SERVICE} \
                            --task-definition ${APP_NAME} \
                            --force-new-deployment
                        
                        # Wait for deployment
                        aws ecs wait services-stable \
                            --cluster ${ECS_CLUSTER} \
                            --services ${ECS_SERVICE}
                    """
                }
            }
        }
    }
}
```

---

## Interview Assignment: Design a Complete CI/CD System

**Scenario:** Your company is building a microservices platform with 10 services.

**Design requirements:**
1. Every service has: unit tests, integration tests, Docker build
2. PRs trigger: test + build (no deploy)
3. Merging to `main`: deploy to staging automatically
4. Deploying to production: requires 2 manual approvals
5. Rollback must be possible within 5 minutes
6. Build status visible on GitHub PRs
7. Failed builds notify on Slack #alerts channel
8. Build artifacts stored in AWS ECR with lifecycle policies

**Deliverables:**
1. Jenkins pipeline architecture diagram
2. Sample Jenkinsfile for one service
3. Shared library design for common patterns
4. Rollback procedure (documented as Runbook)
5. Monitoring plan: what metrics to track in Jenkins

---

## Quick Reference

```bash
# Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 -auth user:token list-jobs
java -jar jenkins-cli.jar -s http://localhost:8080 -auth user:token build JOB_NAME
java -jar jenkins-cli.jar -s http://localhost:8080 -auth user:token build JOB_NAME -p VERSION=1.2.3

# Trigger build via API
curl -X POST "http://user:token@localhost:8080/job/MyJob/build"
curl -X POST "http://user:token@localhost:8080/job/MyJob/buildWithParameters?VERSION=1.2.3"

# Get build status
curl "http://user:token@localhost:8080/job/MyJob/lastBuild/api/json"

# Pipeline syntax helper (in Jenkins UI)
# http://localhost:8080/pipeline-syntax/
```
