# Devops-Project-1
Real-Time End-to-End Kubernetes CI/CD Pipeline using Jenkins, ArgoCD, SonarQube, Trivy &amp; AWS EKS

This repository contains the complete documentation, configurations, and pipeline code for building a production-ready, fully automated **DevOps CI/CD GitOps Pipeline**. 

The pipeline automates the lifecycle of a Java-based web application—from the initial code commit in GitHub to automated static code analysis, security vulnerability scanning, containerization, and declarative GitOps deployment onto a managed **Amazon Elastic Kubernetes Service (EKS)** cluster.

---

## 🏗️ Architectural Design & System Flow

To implement a clean separation of concerns and robust security boundaries, this project separates the **Application Code** from the **Infrastructure/GitOps Manifests**. This separation prevents pipeline looping, ensures clear ownership, and aligns with modern enterprise GitOps standards.

```mermaid
graph TD
    %% Source Control
    subgraph SCM [GitHub Repositories]
        AppRepo[Java App Repository <br> Contains Source & CI Jenkinsfile]
        GitOpsRepo[GitOps Repository <br> Contains K8s Manifests & CD Jenkinsfile]
    end

    %% CI Pipeline (Jenkins Master/Agent)
    subgraph CI [Continuous Integration - Jenkins VMs]
        Master[Jenkins Master VM <br> t2.micro] -- Orchestrates --> Agent[Jenkins Agent VM <br> t2.micro]
        Agent -- 1. Compiles & Tests --> Maven[Maven Build]
        Agent -- 2. Static Analysis --> Sonar[SonarQube Server <br> t3.medium]
        Sonar -- 3. Webhook Quality Gate --> Agent
        Agent -- 4. Builds Container --> Docker[Docker Engine]
        Agent -- 5. Security Scan --> Trivy[Trivy Vulnerability Scanner]
    end

    %% Artifact Registry
    subgraph Registry [Container Registry]
        DockerHub[Docker Hub]
    end

    %% CD & GitOps
    subgraph CD [Continuous Delivery & GitOps]
        Bootstrap[EKS Bootstrap VM <br> t2.micro]
        EKS[AWS EKS Cluster <br> 3x t2.small Nodes]
        Argo[ArgoCD <br> Deployed on EKS]
    end

    %% Core Pipeline Workflow Connections
    Developer -- Pushes Code --> AppRepo
    AppRepo -- Polls SCM / Webhook --> Master
    Agent -- 6. Pushes Image --> DockerHub
    Agent -- 7. Triggers CD Job with Tag --> Master
    Master -- Executes CD Job --> GitOpsRepo
    GitOpsRepo -- 8. Updates deployment.yaml Tag --> GitOpsRepo
    Argo -- 9. Monitors Manifest Changes --> GitOpsRepo
    Argo -- 10. Pulls & Reconciles State --> EKS
    DockerHub -- 11. Pulls Certified Image --> EKS

    style SCM fill:#f5f5f5,stroke:#333,stroke-width:2px;
    style CI fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    style Registry fill:#efebe9,stroke:#5d4037,stroke-width:2px;
    style CD fill:#e8f5e9,stroke:#388e3c,stroke-width:2px;
```

### Infrastructure Summary
*   **Jenkins Master (t2.micro):** Serves as the central pipeline coordinator. It has zero executors to keep it performant.
*   **Jenkins Agent (t2.micro):** Executes the pipeline execution phases (compiling, testing, containerizing, scanning).
*   **SonarQube Server (t3.medium):** Hosts the code quality database and static analysis engine.
*   **EKS Bootstrap Workstation (t2.micro):** A management machine to orchestrate cluster deployment and configure ArgoCD CLI.
*   **Amazon EKS Cluster (3x t2.small Nodes):** High-availability Kubernetes cluster running our production pods.

---

## 📁 Repository Strategy & Structures

To reproduce this project, you must create **two separate** repositories on your GitHub account:

### 1. Application Repository (`register-app`)
Contains the application source code, its Dockerfile, and the Continuous Integration configuration:
```text
register-app/
├── src/                    # Java Web App source code
├── pom.xml                 # Maven Project Object Model configuration
├── Dockerfile              # Instructions to containerize the compiled WAR
└── Jenkinsfile             # Declarative CI pipeline configuration
```

### 2. GitOps Configuration Repository (`gitops-register-app`)
Contains the declarative Kubernetes state files and the Continuous Delivery configuration:
```text
gitops-register-app/
├── deployment.yaml         # Kubernetes Deployment configuration
├── service.yaml            # Kubernetes LoadBalancer Service configuration
└── Jenkinsfile             # Declarative CD pipeline configuration (updates image tags)
```

---

## 🛠️ Step-by-Step Implementation Guide

### Phase 1: Virtual Machine Provisioning on AWS

Deploy 4 distinct EC2 instances in your default AWS VPC.

| Server Name | AMI | Instance Type | Storage | Security Group Open Ports |
| :--- | :--- | :--- | :--- | :--- |
| **Jenkins Master** | Ubuntu 22.04 LTS | `t2.micro` | 15 GB | SSH (22), Custom TCP (8080) |
| **Jenkins Agent** | Ubuntu 22.04 LTS | `t2.micro` | 15 GB | SSH (22) |
| **SonarQube Server** | Ubuntu 22.04 LTS | `t3.medium` | 15 GB | SSH (22), Custom TCP (9000) |
| **EKS Bootstrap Server** | Ubuntu 22.04 LTS | `t2.micro` | 15 GB | SSH (22) |

---

### Phase 2: Configuring the CI Environment (Jenkins Master & Agent)

#### 1. Setup Jenkins Master
SSH into the Jenkins Master VM:
```bash
# Update local repository indexes
sudo apt-get update -y && sudo apt-get upgrade -y

# Configure Server Identity
sudo hostnamectl set-hostname "Jenkins-Master"
sudo reboot
```
After rebooting, reconnect and install Java and Jenkins:
```bash
# Install OpenJDK 17
sudo apt-get install openjdk-17-jdk -y

# Import Jenkins Repository Key and Package List
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt-get update -y
sudo apt-get install jenkins -y

# Start and enable daemon
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

#### 2. Setup Jenkins Agent
SSH into the Jenkins Agent VM:
```bash
sudo apt-get update -y && sudo apt-get upgrade -y
sudo hostnamectl set-hostname "Jenkins-Agent"
sudo reboot
```
After rebooting, reconnect and install build tools and container runtime:
```bash
# Install Java runtime (required to communicate with Jenkins Master)
sudo apt-get install openjdk-17-jdk -y

# Install Docker Engine
sudo apt-get install docker.io -y

# Add user 'ubuntu' to 'docker' group to run commands without sudo
sudo usermod -aG docker ubuntu
sudo reboot
```

#### 3. Establish SSH Passwordless Trust
To allow the Master to securely manage the Agent, establish SSH key authorization.

**On Jenkins Master VM:**
```bash
# Generate high-entropy RSA Keypair
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa

# Display the public key (Copy this output to clipboard)
cat ~/.ssh/id_rsa.pub
```

**On Jenkins Agent VM:**
```bash
# Open authorization database
nano ~/.ssh/authorized_keys
# Paste the Master public key at the end of this file, save and exit

# Open SSH service configuration
sudo nano /etc/ssh/sshd_config
# Ensure 'PubkeyAuthentication yes' is enabled.
sudo systemctl reload ssh
```

---

### Phase 3: Activating the Jenkins Distributed Cluster

1. Navigate to `http://<Jenkins-Master-Public-IP>:8080`
2. Unlock using the secret key printed via your Master terminal:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Install the **Suggested Plugins**, create your administrator account, and log in.
4. **Disable executors on Master:** Go to **Manage Jenkins** -> **Nodes** -> **Built-in Node** -> **Configure** -> Set **Number of executors** to `0` -> **Save**.
5. **Register Agent Node:** Go to **Manage Jenkins** -> **Nodes** -> **New Node**:
   *   Name: `Jenkins agent` (Select **Permanent Agent**)
   *   Number of executors: `2`
   *   Remote root directory: `/home/ubuntu`
   *   Labels: `Jenkins agent`
   *   Usage: `Use this node as much as possible`
   *   Launch method: `Launch agents via SSH`
   *   Host: `<Jenkins-Agent-Private-IP>`
   *   Credentials: Click **Add** -> **Jenkins**
       *   Kind: `SSH Username with private key`
       *   ID: `jenkins-agent-creds`
       *   Username: `ubuntu`
       *   Private Key: Select **Enter directly** -> paste the entire **Private Key** from your **Jenkins Master** (`cat ~/.ssh/id_rsa`)
       *   Click **Add** and select this credential.
   *   Host Key Verification Strategy: `Non-verifying verification strategy`
   *   Click **Save**.

---

### Phase 4: Build Tool Configurations in Jenkins

Install the necessary Maven plugins and define global toolpaths.

1. Navigate to **Manage Jenkins** -> **Plugins** -> **Available Plugins**. Search for and install:
   *   **Maven Integration**
   *   **Pipeline Maven Integration**
   *   **Eclipse Temurin Installer**
2. Navigate to **Manage Jenkins** -> **Tools**.
3. **Configure Java Compiler:**
   *   Under **JDK installations**, click **Add JDK**.
   *   Name: `Java 17`
   *   Check **Install automatically**.
   *   Add Installer: **Install from adaptium.net** -> Version: `17.0.5+8`.
4. **Configure Build Engine:**
   *   Under **Maven installations**, click **Add Maven**.
   *   Name: `Maven3`
   *   Check **Install automatically** -> Version: Select the latest version available.
5. Click **Apply & Save**.

---

### Phase 5: Code Quality & Security Platform (SonarQube)

#### 1. Setup SonarQube Server
SSH into the SonarQube VM. We use Postgresql as the data storage layer.
```bash
sudo apt-get update -y && sudo apt-get upgrade -y

# Install PostgreSQL DBMS
sudo apt-get install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Create SonarQube DB schema & credentials
sudo -i -u postgres psql
```
Inside the PostgreSQL console:
```sql
CREATE USER sonar WITH ENCRYPTED PASSWORD 'sonar_db_password';
CREATE DATABASE sonar OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonar TO sonar;
\q
```

```bash
# Install Java runtimes
sudo apt-get install openjdk-17-jdk -y

# Configure OS kernel boundaries to support SonarQube's Elasticsearch instance
sudo nano /etc/security/limits.conf
# Append to the file:
sonar   soft   nofile   65536
sonar   hard   nofile   65536

sudo nano /etc/sysctl.conf
# Append to the file:
vm.max_map_count=262144

sudo reboot
```
Reconnect via SSH and install the application:
```bash
# Unzip utility
sudo apt-get install unzip -y

# Download and place SonarQube packages in /opt
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
sudo unzip sonarqube-9.9.0.65466.zip
sudo mv sonarqube-9.9.0.65466 sonarqube

# Create isolated runtime user and assign permissions
sudo useradd sonar
sudo chown -R sonar:sonar /opt/sonarqube

# Integrate database configurations
sudo nano /opt/sonarqube/conf/sonar.properties
# Uncomment and set the following parameters:
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar_db_password
sonar.jdbc.url=jdbc:postgresql://localhost/sonar

# Write Systemd daemon unit file
sudo nano /etc/systemd/system/sonarqube.service
```
Paste this configuration into the file:
```ini
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```
Start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
```

#### 2. Configure SonarQube Integration in Jenkins
1. Open `http://<SonarQube-VM-Public-IP>:9000` (Default user/pass: `admin`/`admin`. Update password on first sign-in).
2. Generate Token: Navigate to **My Account** -> **Security** -> **Generate Token**:
   *   Name: `Jenkins-Token`
   *   Type: `Global Analysis Token`
   *   Click **Generate** and copy the hash value.
3. Establish Webhook feedback loop: Go to **Administration** -> **Configuration** -> **Webhooks** -> **Create**:
   *   Name: `jenkins-webhook`
   *   URL: `http://<Jenkins-Master-Private-IP>:8080/sonarqube-webhook/`
   *   Click **Create**.
4. In Jenkins UI: Go to **Manage Jenkins** -> **Plugins** and install **SonarQube Scanner** and **Sonar Quality Gates**.
5. Add Secret Credential: Go to **Manage Jenkins** -> **Credentials** -> **Global** -> **Add Credentials**:
   *   Kind: `Secret text`
   *   Secret: paste the SonarQube token
   *   ID: `Jenkins SonarQube token`
6. Register Server Environment: Go to **Manage Jenkins** -> **System** -> **SonarQube servers**:
   *   Check **Enable installation of SonarQube**
   *   Name: `SonarQube server`
   *   Server URL: `http://<SonarQube-VM-Private-IP>:9000`
   *   Server authentication token: Select `Jenkins SonarQube token`
   *   Click **Save**.
7. Register Scanner Toolpath: Go to **Manage Jenkins** -> **Tools** -> **SonarQube scanner installations** -> **Add SonarQube Scanner**:
   *   Name: `SonarQube scanner`
   *   Check **Install automatically** -> Click **Save**.

---

### Phase 6: Core Image Compilation & Vulnerability Scanning (Trivy)

#### 1. Setup Docker Pipeline Plugins and Credentials
1. Install Docker Plugins: Go to **Manage Jenkins** -> **Plugins** -> **Available Plugins**. Install:
   *   `Docker`
   *   `Docker Pipeline`
   *   `CloudBees Docker Build and Push`
   *   Restart Jenkins after installation completes.
2. Store Registry Credentials: Go to **Manage Jenkins** -> **Credentials** -> **Global** -> **Add Credentials**:
   *   Kind: `Username with password`
   *   Username: `<Your-DockerHub-Username>`
   *   Password: `<Your-DockerHub-Access-Token>` (Recommended over plain passwords)
   *   ID: `Docker hub`

#### 2. Setup Trivy Scanner on Jenkins Agent
SSH into the Jenkins Agent VM and install Trivy:
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

---

### Phase 7: Provisioning Kubernetes (EKS Cluster)

#### 1. Setup Bootstrap Workstation
SSH into the EKS Bootstrap Server:
```bash
sudo apt-get update -y && sudo apt-get upgrade -y
sudo hostnamectl set-hostname "EKS-Workstation"
sudo reboot
```
After rebooting, reconnect and install AWS CLI, Kubectl, and Eksctl:
```bash
# AWS CLI
sudo apt-get install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /bin/

# Eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /bin/
```

#### 2. Authorize EKS Workstation via AWS IAM
1. Go to **AWS Console** -> **IAM** -> **Roles** -> **Create Role**.
2. Select **AWS Service** -> **EC2** as the trusted entity.
3. Attach the **AdministratorAccess** policy (for sandbox learning) -> Name it `eksctl_score role`.
4. Go to **EC2 Console** -> Select `EKS-Workstation` instance -> **Actions** -> **Security** -> **Modify IAM Role** -> Select `eksctl_score role` -> Click **Update IAM Role**.

#### 3. Build AWS Managed EKS Cluster
Execute this deployment command on your EKS Bootstrap Server:
```bash
eksctl create cluster   --name virtualtechbox-eks-cluster   --region us-east-1   --node-type t2.small   --nodes 3
```
*Note: Cluster provisioning takes approximately 15 to 20 minutes.*
Once completed, verify your cluster nodes are available:
```bash
kubectl get nodes
```

---

### Phase 8: GitOps Deployment Framework (ArgoCD Setup)

Perform these configurations on your **EKS Bootstrap Workstation**:

```bash
# 1. Build Namespace
kubectl create namespace argocd

# 2. Deploy Manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.4/manifests/install.yaml

# 3. Expose Server over LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# 4. Wait for AWS ELB Allocation & Retrieve Service URL
kubectl get svc -n argocd
# Copy the EXTERNAL-IP address of the argocd-server.

# 5. Extract administrative decryption password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
```

#### Connect Repositories to ArgoCD UI
1. Open the external-IP URL in your browser over HTTPS. Log in using `admin` and the decoded secret key.
2. Go to **Settings** (Gear icon) -> **Repositories** -> **Connect Repo Using HTTPS**:
   *   Repository URL: Your GitOps repository URL (`https://github.com/your-username/gitops-register-app.git`)
   *   Username: Your GitHub Username
   *   Password: `<Your-GitHub-PAT>`
   *   Click **Connect**.
3. Create the GitOps Sync App:
   *   Click **New App**:
       *   Application Name: `Register-App`
       *   Project: `default`
       *   Sync Policy: **Automatic** (Check **Prune Resources** and **Self-Heal**)
       *   Source Repository URL: Select your GitOps URL.
       *   Revision: `Main` (or branch name)
       *   Path: `./`
       *   Cluster URL: Select your default cluster (`https://kubernetes.default.svc`)
       *   Namespace: `default`
       *   Click **Create**.

---

### Phase 9: Creating the Trigger Mechanism (Jenkins CD Job)

The GitOps workflow requires a secondary Jenkins pipeline that modifies the manifest files when a new Docker image is successfully pushed.

#### 1. Add GitHub PAT Credential in Jenkins
1. Go to **Manage Jenkins** -> **Credentials** -> **Global** -> **Add Credentials**:
   *   Kind: `Username with password`
   *   Username: Your GitHub Username
   *   Password: `<Your-GitHub-PAT>`
   *   ID: `github`

#### 2. Create Jenkins CD Project
1. Click **New Item** -> Name: `register-app-cd` -> Select **Pipeline** -> Click **OK**.
2. Under configurations:
   *   Check **Discard old builds** (Max to keep: 2)
   *   Check **This project is parameterized** -> Click **Add Parameter** -> **String Parameter**:
       *   Name: `image tag`
   *   Check **Trigger builds remotely**:
       *   Authentication Token: `gitops-token`
3. Under **Pipeline**:
   *   Definition: `Pipeline script from SCM`
   *   SCM: `Git`
   *   Repository URL: `<Your-GitOps-Repo-URL>`
   *   Credentials: Select your `github` credentials.
   *   Branch Specifier: `*/Main` (Ensure this matches your SCM branch)
   *   Script Path: `Jenkinsfile`
   *   Click **Save**.

#### 3. Setup Authenticated Web API Token
To allow our CI job to safely trigger our CD job, generate a Jenkins security token:
1. Click your user profile name (top-right of Jenkins screen) -> **Configure**.
2. Scroll to **API Token** -> **Add new token** -> Name: `Jenkins API token` -> **Generate**.
3. Copy the token hash immediately.
4. Go to **Manage Jenkins** -> **Credentials** -> **Global** -> **Add Credentials**:
   *   Kind: `Secret text`
   *   Secret: Paste the API token hash.
   *   ID: `Jenkins API token`
   *   Click **Save**.

---

## 📝 Configuration Codes & Pipeline Manifests

### 1. Application Repository: Dockerfile
```dockerfile
FROM tomcat:9.0-jre17-alpine
EXPOSE 8080
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/*.war /usr/local/tomcat/webapps/webapp.war
CMD ["catalina.sh", "run"]
```

### 2. Application Repository: Jenkinsfile (CI)
```groovy
pipeline {
    agent { label 'Jenkins agent' }
    
    tools {
        jdk 'Java 17'
        maven 'Maven3'
    }
    
    environment {
        APP_NAME = 'register-app-pipeline'
        DOCKER_USER = 'your-dockerhub-username'
        DOCKER_CREDS = credentials('Docker hub')
        JENKINS_API_TOKEN = credentials('Jenkins API token')
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout SCM') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/your-username/register-app.git'
            }
        }
        
        stage('Build Artifact') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Run Tests') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Static Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Quality Gate Evaluator') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        
        stage('Container Compilation') {
            steps {
                sh 'docker build -t ${DOCKER_USER}/${APP_NAME}:${BUILD_NUMBER} .'
                sh 'docker login -u ${DOCKER_USER} -p ${DOCKER_CREDS_PSW}'
                sh 'docker push ${DOCKER_USER}/${APP_NAME}:${BUILD_NUMBER}'
            }
        }
        
        stage('Container Image Scan') {
            steps {
                sh 'trivy image ${DOCKER_USER}/${APP_NAME}:${BUILD_NUMBER}'
            }
        }
        
        stage('Prune Local Images') {
            steps {
                sh 'docker rmi ${DOCKER_USER}/${APP_NAME}:${BUILD_NUMBER}'
            }
        }
        
        stage('Trigger GitOps CD Delivery') {
            steps {
                // Securely invoke the secondary Jenkins CD pipeline, passing the current build number
                sh "curl -v -X POST -u clouduser:${JENKINS_API_TOKEN} 'http://<Jenkins-Master-Private-IP-Address>:8080/job/register-app-cd/buildWithParameters?token=gitops-token&image%20tag=${BUILD_NUMBER}'"
            }
        }
    }
}
```

### 3. GitOps Repository: deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: register-app-deployment
  labels:
    app: register-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: register-app
  template:
    metadata:
      labels:
        app: register-app
    spec:
      containers:
      - name: webapp
        image: your-dockerhub-username/register-app-pipeline:1 # Automatically edited by SCM sed stream
        ports:
        - containerPort: 8080
```

### 4. GitOps Repository: service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: register-app-service
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: register-app
```

### 5. GitOps Repository: Jenkinsfile (CD)
```groovy
pipeline {
    agent { label 'Jenkins agent' }
    
    environment {
        APP_NAME = 'register-app-pipeline'
        DOCKER_USER = 'your-dockerhub-username'
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout Manifest SCM') {
            steps {
                git branch: 'Main', credentialsId: 'github', url: 'https://github.com/your-username/gitops-register-app.git'
            }
        }
        
        stage('Patch Kubernetes Deployment') {
            steps {
                // Dynamically updates the container image tag to match the new CI build version
                sh "sed -i 's|image: .*|image: ${DOCKER_USER}/${APP_NAME}:${params.image_tag}|g' deployment.yaml"
            }
        }
        
        stage('Push Changes to SCM') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh 'git config user.name "clouduser"'
                    sh 'git config user.email "clouduser@example.com"'
                    sh 'git add deployment.yaml'
                    sh 'git commit -m "Updated deployment image tag to version ${params.image_tag} [Skip CI]"'
                    sh "git push https://${GIT_USER}:${GIT_PASS}@github.com/your-username/gitops-register-app.git HEAD:Main"
                }
            }
        }
    }
}
```

---

## 🚀 Verifying the Infrastructure Pipeline

Once you configure your files and SCM environments, verify the setup:

1. **Configure Poll SCM:** In your Jenkins CI Job configuration under **Build Triggers**, check **Poll SCM** and input `* * * * *` (polls GitHub repository for code changes every single minute).
2. **Make a Code Update:** Open your application code (e.g., modifying raw files under `src/main/webapp/index.jsp` to print `"Continuous Delivery Verified!"`), commit and push.
3. **Monitor the Pipeline Stages:**
   *   **CI Pipeline** triggers instantly -> build, testing, quality analysis, containerization, safety scanning complete successfully -> CI calls your CD pipeline endpoint.
   *   **CD Pipeline** triggers -> updates your GitOps `deployment.yaml` with the new Docker image build tag -> commits and pushes to Git.
   *   **ArgoCD** detects the git push -> automates synchronization -> pulls the new image and triggers a rolling update across your cluster pods.
4. **Access the Deployment:** Fetch the LoadBalancer URL:
   ```bash
   kubectl get svc register-app-service
   ```
   Open your browser, navigate to `http://<AWS-ELB-DNS>:8080/webapp/`, and confirm your code changes are live!

