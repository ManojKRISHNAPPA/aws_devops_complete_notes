# Jenkins CI/CD — Complete Guide

## Table of Contents
1. [What is Jenkins?](#1-what-is-jenkins)
2. [Jenkins Architecture](#2-jenkins-architecture)
3. [Installation and Setup](#3-installation-and-setup)
4. [Pipeline Concepts](#4-pipeline-concepts)
5. [Declarative Pipeline](#5-declarative-pipeline)
6. [Scripted Pipeline](#6-scripted-pipeline)
7. [Shared Libraries](#7-shared-libraries)
8. [Jenkins Agents](#8-jenkins-agents)
9. [Plugins Ecosystem](#9-plugins-ecosystem)
10. [Security Configuration](#10-security-configuration)
11. [Jenkins with Docker](#11-jenkins-with-docker)
12. [Real-World CI/CD Pipelines](#12-real-world-cicd-pipelines)
13. [Jenkins Best Practices](#13-jenkins-best-practices)
14. [Common Interview Questions](#14-common-interview-questions)

---

## 1. What is Jenkins?

**Jenkins** is an open-source automation server used to implement CI/CD pipelines. It automates building, testing, and deploying software.

```
Without Jenkins (Manual Process):
Developer commits → manually build → manually test → manually deploy → hope for no bugs

With Jenkins (Automated CI/CD):
Developer commits ──► Jenkins detects ──► Build ──► Test ──► Code Quality ──► Deploy ──► Notify
       │                                                         │
       └── Immediate feedback loop ◄──────────────────────────┘
           (fail fast, fix fast)
```

**CI/CD Concepts:**
- **Continuous Integration (CI)**: Automatically build and test on every commit
- **Continuous Delivery (CD)**: Every passing build is deployable (human gate before production)
- **Continuous Deployment**: Every passing build auto-deploys to production

---

## 2. Jenkins Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Jenkins Controller                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │ Web UI   │  │  REST    │  │  Job Configuration   │  │
│  │ (port 80)│  │  API     │  │  & Scheduling        │  │
│  └──────────┘  └──────────┘  └──────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐ │
│  │           Build Queue & Executor Pool               │ │
│  └────────────────────────────────────────────────────┘ │
└────────────────────────┬────────────────────────────────┘
                         │ (SSH / JNLP / Docker)
         ┌───────────────┼───────────────┐
         │               │               │
  ┌──────▼──────┐ ┌──────▼──────┐ ┌─────▼───────┐
  │   Agent 1   │ │   Agent 2   │ │   Agent 3   │
  │   (Linux)   │ │  (Windows)  │ │   (Docker)  │
  │  Java Build │ │ .NET Build  │ │ Node Build  │
  └─────────────┘ └─────────────┘ └─────────────┘
         │
  ┌──────▼──────────────────────────────────────────┐
  │         External Services                        │
  │  Git/GitHub  Slack  SonarQube  Nexus  AWS  K8s  │
  └─────────────────────────────────────────────────┘
```

**Key Jenkins Terminology:**
| Term | Description |
|------|-------------|
| **Job/Project** | A configured task (build, test, deploy) |
| **Pipeline** | A series of automated stages defined as code |
| **Build** | Single execution of a job |
| **Executor** | Slot for running builds on an agent |
| **Workspace** | Directory where Jenkins runs builds |
| **Artifact** | File produced by a build (jar, docker image) |
| **SCM** | Source Control Management (Git, SVN) |

---

## 3. Installation and Setup

### Docker Installation (Recommended for Dev)

```bash
# Run Jenkins in Docker
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Production Installation (Ubuntu)

```bash
# Add Jenkins repo
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
    sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | \
    sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install
sudo apt-get update
sudo apt-get install -y jenkins openjdk-17-jre

# Start and enable
sudo systemctl enable --now jenkins

# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Jenkins listens on port 8080
# Initial Setup Wizard:
# 1. Enter admin password
# 2. Install suggested plugins
# 3. Create admin user
# 4. Set Jenkins URL
```

### Directory Structure

```
/var/lib/jenkins/
├── jobs/                    # Job configurations and build history
│   └── my-pipeline/
│       ├── config.xml       # Job configuration
│       └── builds/
│           └── 42/          # Build #42
│               ├── log      # Console output
│               └── archive/ # Archived artifacts
├── plugins/                 # Installed plugins
├── workspace/               # Build workspaces
│   └── my-pipeline/         # Checkout + build files
├── secrets/                 # Encrypted credentials
├── users/                   # User accounts
└── nodes/                   # Agent configurations
```

---

## 4. Pipeline Concepts

### Pipeline as Code Benefits

```
Without Pipeline as Code:          With Pipeline as Code (Jenkinsfile):
───────────────────────────────    ──────────────────────────────────────
Config in Jenkins UI               Config in source control (Jenkinsfile)
Hard to review changes             PR review for pipeline changes
Configuration drift                Version-controlled, reproducible
Not reproducible                   Runs the same everywhere
Lost if Jenkins breaks             Backed up in Git
Manual documentation               Self-documenting
```

### Pipeline Stages

```
Checkout → Build → Test → Code Quality → Security Scan → Package → Deploy Staging → Integration Tests → Deploy Prod
   │         │       │         │               │             │            │                  │               │
  Git       Maven  JUnit   SonarQube         Trivy         Docker       ECS/K8s         Selenium          ECS/K8s
  clone     build  tests    analysis        image scan     push         deploy            tests            deploy
```

---

## 5. Declarative Pipeline

Declarative pipeline is the modern, structured syntax (recommended):

```groovy
// Jenkinsfile (place in root of your repository)
pipeline {
    // Where to run
    agent any   // Run on any available agent
    // Or:
    // agent { label 'linux' }
    // agent { docker 'maven:3.9-eclipse-temurin-17' }
    // agent none  // Defined per stage

    // Global options
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timestamps()
        ansiColor('xterm')
    }

    // Parameters (triggered manually)
    parameters {
        string(name: 'VERSION', defaultValue: '', description: 'Release version')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'])
        booleanParam(name: 'SKIP_TESTS', defaultValue: false)
    }

    // Environment variables
    environment {
        APP_NAME = 'myapp'
        DOCKER_REGISTRY = 'registry.example.com'
        // Credentials from Jenkins store
        SONAR_TOKEN = credentials('sonarqube-token')
        AWS_CREDENTIALS = credentials('aws-credentials')  // Sets AWS_CREDENTIALS_USR and AWS_CREDENTIALS_PSW
        DOCKER_CREDS = credentials('docker-registry')
    }

    // Trigger conditions
    triggers {
        pollSCM('H/5 * * * *')   // Poll Git every 5 min
        cron('H 2 * * *')        // Nightly build at 2AM
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.GIT_BRANCH = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
                }
                echo "Branch: ${env.GIT_BRANCH}, Commit: ${env.GIT_COMMIT}"
            }
        }

        stage('Build') {
            agent { docker 'maven:3.9-eclipse-temurin-17' }
            steps {
                sh 'mvn clean package -DskipTests'
                stash name: 'build-output', includes: 'target/*.jar'
            }
        }

        stage('Test') {
            when {
                not { expression { params.SKIP_TESTS } }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test'
                        junit 'target/surefire-reports/*.xml'
                    }
                }
                stage('Code Coverage') {
                    steps {
                        sh 'mvn verify -P coverage'
                        publishHTML target: [
                            reportName: 'Coverage Report',
                            reportDir: 'target/site/jacoco',
                            reportFiles: 'index.html'
                        ]
                    }
                }
            }
        }

        stage('Code Quality') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                unstash 'build-output'
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${env.GIT_COMMIT}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh "trivy image ${DOCKER_REGISTRY}/${APP_NAME}:${env.GIT_COMMIT} --exit-code 1 --severity CRITICAL"
            }
        }

        stage('Deploy to Staging') {
            environment {
                ENV = 'staging'
            }
            steps {
                sh """
                    aws ecs update-service \
                        --cluster ${ENV}-cluster \
                        --service ${APP_NAME} \
                        --force-new-deployment \
                        --region us-east-1
                """
                sh "./scripts/wait-for-deployment.sh ${ENV} ${APP_NAME}"
            }
        }

        stage('Integration Tests') {
            steps {
                sh 'mvn verify -P integration-tests -Denv=staging'
                junit 'target/failsafe-reports/*.xml'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                submitter "admin,devops-lead"
            }
            steps {
                sh """
                    aws ecs update-service \
                        --cluster prod-cluster \
                        --service ${APP_NAME} \
                        --force-new-deployment \
                        --region us-east-1
                """
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean workspace
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ ${APP_NAME} deployed successfully\nBranch: ${env.GIT_BRANCH}\nCommit: ${env.GIT_COMMIT}"
            )
        }
        failure {
            slackSend(
                channel: '#alerts',
                color: 'danger',
                message: "❌ Pipeline FAILED: ${APP_NAME}\nBranch: ${env.GIT_BRANCH}\nBuild: ${env.BUILD_URL}"
            )
            emailext(
                to: 'team@company.com',
                subject: "Build Failed: ${APP_NAME} #${BUILD_NUMBER}",
                body: "Check build: ${BUILD_URL}"
            )
        }
        unstable {
            slackSend(channel: '#alerts', color: 'warning', message: "⚠️ Build UNSTABLE: ${APP_NAME}")
        }
    }
}
```

---

## 6. Scripted Pipeline

Groovy-based, more flexible but less structured:

```groovy
node('linux') {
    def app
    def version = params.VERSION ?: "1.0.${BUILD_NUMBER}"
    
    try {
        stage('Checkout') {
            checkout scm
        }
        
        stage('Build') {
            sh "mvn clean package"
        }
        
        stage('Docker') {
            app = docker.build("myapp:${version}")
        }
        
        stage('Push') {
            docker.withRegistry('https://registry.example.com', 'docker-creds') {
                app.push(version)
                app.push("latest")
            }
        }
        
        currentBuild.result = 'SUCCESS'
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // Always runs
        stage('Cleanup') {
            cleanWs()
        }
    }
}
```

---

## 7. Shared Libraries

Shared libraries allow reusing pipeline code across multiple Jenkinsfiles:

```
jenkins-shared-library/ (separate Git repo)
├── vars/
│   ├── buildDockerImage.groovy    # Global variable/function
│   ├── deployToK8s.groovy
│   └── sendSlackNotification.groovy
├── src/
│   └── org/company/
│       ├── Build.groovy           # Helper classes
│       └── Deploy.groovy
└── resources/
    └── config.yaml
```

```groovy
// vars/buildDockerImage.groovy
def call(String imageName, String version) {
    docker.build("${imageName}:${version}")
    docker.withRegistry('https://registry.example.com', 'docker-creds') {
        docker.image("${imageName}:${version}").push()
    }
}
```

```groovy
// Jenkinsfile using shared library
@Library('jenkins-shared-library@main') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                buildDockerImage('myapp', '1.2.3')
                deployToK8s('production', 'myapp', '1.2.3')
                sendSlackNotification('#deployments', 'success')
            }
        }
    }
}
```

---

## 8. Jenkins Agents

```bash
# Method 1: SSH Agent
# Jenkins Controller connects via SSH to agent

# On Agent machine:
# 1. Install Java
apt install -y openjdk-17-jre

# 2. Create jenkins user
useradd -m -d /home/jenkins jenkins

# 3. Configure SSH (add Jenkins controller's public key)
mkdir -p /home/jenkins/.ssh
echo "jenkins-controller-public-key" >> /home/jenkins/.ssh/authorized_keys
chmod 700 /home/jenkins/.ssh
chmod 600 /home/jenkins/.ssh/authorized_keys

# In Jenkins UI:
# Manage Jenkins → Nodes → New Node
# - Type: Permanent Agent
# - # of executors: 4
# - Remote root directory: /home/jenkins/workspace
# - Launch method: SSH
# - Host: agent-ip
# - Credentials: jenkins SSH key

# Method 2: Docker Agent (dynamic)
# Each build gets a fresh Docker container
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '-v $HOME/.m2:/root/.m2'  // Cache Maven deps
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
    }
}

# Method 3: Kubernetes Agent (dynamic scaling)
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ['sleep', '99d']
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
"""
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn package'
                }
            }
        }
        stage('Docker') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp .'
                }
            }
        }
    }
}
```

---

## 9. Plugins Ecosystem

**Essential Jenkins Plugins:**

| Plugin | Purpose |
|--------|---------|
| Pipeline | Core pipeline support |
| Git | Git SCM integration |
| GitHub Branch Source | Multi-branch pipelines from GitHub |
| Docker Pipeline | Docker operations in pipeline |
| Kubernetes | Dynamic K8s agents |
| Credentials Binding | Inject secrets into pipeline |
| SSH Agent | SSH key management |
| JUnit | Test result publishing |
| HTML Publisher | HTML report publishing |
| SonarQube Scanner | Code quality |
| Slack Notification | Slack alerts |
| Email Extension | Rich email notifications |
| Blue Ocean | Modern Jenkins UI |
| Role-based Authorization | RBAC |
| AWS Steps | AWS SDK integration |
| Nexus Artifact Uploader | Upload artifacts to Nexus |
| OWASP Dependency Check | Security scanning |
| Timestamper | Add timestamps to console output |

---

## 10. Security Configuration

```groovy
// Credentials management
// Never hardcode secrets in Jenkinsfile!

// Use withCredentials block
withCredentials([
    string(credentialsId: 'api-token', variable: 'API_TOKEN'),
    usernamePassword(credentialsId: 'db-creds', 
                     usernameVariable: 'DB_USER', 
                     passwordVariable: 'DB_PASS'),
    sshUserPrivateKey(credentialsId: 'ssh-key',
                      keyFileVariable: 'SSH_KEY_FILE')
]) {
    sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com'
    sh "mysql -u$DB_USER -p$DB_PASS mydb < schema.sql"
    sh "ssh -i $SSH_KEY_FILE user@server 'deploy.sh'"
}

// AWS credentials (auto-sets environment)
withAWS(credentials: 'aws-creds', region: 'us-east-1') {
    sh 'aws s3 ls'
}
```

```
Jenkins Security Settings:
1. Enable Security: Manage Jenkins → Configure Global Security
2. Authorization: Matrix-based or Role-Based (RBAC plugin)
3. Credentials: Never use plain text, use Jenkins Credentials Store
4. CSRF Protection: Enable
5. Agent Protocols: Use only JNLP4 (encrypted)
6. Plugins: Keep updated, remove unused
7. Jenkins URL: Use HTTPS
8. Audit Logs: Install Audit Trail plugin
```

---

## 11. Jenkins with Docker

```groovy
// Jenkinsfile for containerized application
pipeline {
    agent any
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE = "${REGISTRY}/myapp"
        TAG = "${BUILD_NUMBER}-${GIT_COMMIT.take(7)}"
    }
    
    stages {
        stage('Build Image') {
            steps {
                script {
                    // Multi-stage build for smaller images
                    docker.build("${IMAGE}:${TAG}", 
                        "--build-arg BUILD_DATE=${new Date().format('yyyy-MM-dd')} .")
                }
            }
        }
        
        stage('Scan Image') {
            steps {
                // Trivy security scan
                sh """
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --exit-code 1 \
                        --severity CRITICAL,HIGH \
                        ${IMAGE}:${TAG}
                """
            }
        }
        
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'registry-creds',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS'
                )]) {
                    sh "echo $REGISTRY_PASS | docker login ${REGISTRY} -u $REGISTRY_USER --password-stdin"
                    sh "docker push ${IMAGE}:${TAG}"
                    sh "docker tag ${IMAGE}:${TAG} ${IMAGE}:latest"
                    sh "docker push ${IMAGE}:latest"
                }
            }
        }
        
        stage('Clean Up') {
            steps {
                sh "docker rmi ${IMAGE}:${TAG} || true"
            }
        }
    }
}
```

---

## 12. Real-World CI/CD Pipelines

### Complete Java Microservice Pipeline

```groovy
pipeline {
    agent none
    
    environment {
        SERVICE_NAME = 'payment-service'
        REPO = "mycompany/${SERVICE_NAME}"
        STAGING_CLUSTER = 'arn:aws:ecs:us-east-1:123456789:cluster/staging'
        PROD_CLUSTER    = 'arn:aws:ecs:us-east-1:123456789:cluster/production'
    }
    
    stages {
        stage('Build & Test') {
            agent { docker 'maven:3.9-eclipse-temurin-17' }
            steps {
                sh 'mvn clean verify'
                junit 'target/surefire-reports/*.xml'
                stash name: 'jar', includes: 'target/*.jar'
                recordCoverage qualityGates: [
                    [threshold: 80.0, metric: 'LINE', baseline: 'PROJECT']
                ]
            }
        }
        
        stage('Quality Gate') {
            agent any
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build & Push Docker') {
            agent any
            steps {
                unstash 'jar'
                withAWS(credentials: 'aws-ecr', region: 'us-east-1') {
                    sh """
                        aws ecr get-login-password | \\
                        docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
                    """
                    sh "docker build -t ${REPO}:${BUILD_NUMBER} ."
                    sh "docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/${REPO}:${BUILD_NUMBER}"
                }
            }
        }
        
        stage('Deploy Staging') {
            agent any
            steps {
                withAWS(credentials: 'aws-deploy', region: 'us-east-1') {
                    sh """
                        aws ecs update-service \\
                            --cluster ${STAGING_CLUSTER} \\
                            --service ${SERVICE_NAME} \\
                            --force-new-deployment
                    """
                    // Wait for stable
                    sh """
                        aws ecs wait services-stable \\
                            --cluster ${STAGING_CLUSTER} \\
                            --services ${SERVICE_NAME}
                    """
                }
            }
        }
        
        stage('Approval') {
            when { branch 'main' }
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input message: "Deploy ${SERVICE_NAME} to production?",
                          ok: "Deploy",
                          submitter: "admin,devops-lead"
                }
            }
        }
        
        stage('Deploy Production') {
            when { branch 'main' }
            agent any
            steps {
                withAWS(credentials: 'aws-deploy', region: 'us-east-1') {
                    sh """
                        aws ecs update-service \\
                            --cluster ${PROD_CLUSTER} \\
                            --service ${SERVICE_NAME} \\
                            --force-new-deployment
                    """
                    sh """
                        aws ecs wait services-stable \\
                            --cluster ${PROD_CLUSTER} \\
                            --services ${SERVICE_NAME}
                    """
                }
            }
        }
    }
    
    post {
        success {
            slackSend color: 'good', message: "✅ ${SERVICE_NAME} deployed to production (Build #${BUILD_NUMBER})"
        }
        failure {
            slackSend color: 'danger', message: "❌ ${SERVICE_NAME} pipeline FAILED (Build #${BUILD_NUMBER})\n${BUILD_URL}"
        }
    }
}
```

---

## 13. Jenkins Best Practices

```
✅ DO:
  • Store Jenkinsfiles in source control (Pipeline as Code)
  • Use declarative pipeline syntax (readable, validated)
  • Use credentials plugin — never hardcode secrets
  • Clean workspace after builds (cleanWs())
  • Use parallel stages where possible
  • Set timeouts to prevent hung builds
  • Keep build history limited (buildDiscarder)
  • Use shared libraries for common patterns
  • Tag Docker images with build number + git SHA
  • Always wait for ECS/K8s deployment stability

❌ DON'T:
  • Run builds on the Jenkins controller
  • Store secrets in environment variables in plain text
  • Use scripted pipeline when declarative works
  • Leave unused plugins installed (security risk)
  • Skip tests to "save time"
  • Ignore failing quality gates
  • Deploy to production without human approval
  • Store large binaries in Jenkins artifacts (use Nexus/S3)
```

---

## 14. Common Interview Questions

**Q1: What is the difference between Declarative and Scripted Pipeline?**
> Declarative: structured, predefined syntax with `pipeline {}` block, validated by Jenkins, easier to read, recommended. Scripted: Groovy DSL with `node {}` block, more flexible/powerful but harder to read. Both support complex logic, but declarative is preferred for new pipelines.

**Q2: How do you handle secrets in Jenkins?**
> Jenkins Credentials Store (never in Jenkinsfile). Use `withCredentials` block to inject. Types: Secret text, Username/Password, SSH Key, Secret File, AWS credentials. Credentials are masked in console output. For Kubernetes: use Vault plugin or K8s secrets injection.

**Q3: What is a Jenkins agent and why use multiple agents?**
> An agent is a machine running Jenkins build steps. Multiple agents: run builds in parallel, use different OS/environments (Linux for Java, Windows for .NET), isolate builds, scale horizontally. Controller should only orchestrate, not run builds.

**Q4: How do you trigger a Jenkins pipeline automatically?**
> (1) Webhook: GitHub/GitLab sends POST to Jenkins on push/PR. (2) Poll SCM: `H/5 * * * *` checks for changes every 5 min. (3) Cron: `cron('H 2 * * *')` scheduled builds. (4) API trigger: `curl -X POST http://jenkins/job/name/build`. Webhooks are preferred — instant, no polling.

**Q5: How do you implement blue-green deployment in Jenkins?**
> Deploy new version to "green" environment → run smoke tests → if passing, switch load balancer from "blue" to "green" → blue becomes standby. Rollback: switch LB back to blue. In AWS ECS/ALB: swap target groups. Implement via shell scripts in pipeline stages with approval gate.

**Q6: What is the purpose of `stash` and `unstash`?**
> `stash` saves files from one stage's workspace; `unstash` restores them in another stage (possibly on different agent). Used when stages run on different agents and need to share artifacts (e.g., build jar on Maven agent, then push Docker image on another agent).

**Q7: How do you run stages in parallel in Jenkins?**
> ```groovy
> stage('Parallel Tests') {
>   parallel {
>     stage('Unit') { steps { sh 'mvn test' } }
>     stage('Integration') { steps { sh 'mvn verify -Pintegration' } }
>   }
> }
> ```
> Parallel stages run simultaneously. Use `failFast: true` to abort all if one fails.

**Q8: How do you implement rollback in Jenkins?**
> Keep previous image tags (Docker) or artifact versions. Rollback pipeline: take version as parameter, update ECS task definition to previous image, redeploy. Or use ECS update-service with previous task definition revision. Add as separate job or pipeline parameter.

---

*Next: [Jenkins Assignments](02-jenkins-assignments.md)*
