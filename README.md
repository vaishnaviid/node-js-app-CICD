#  Node.js App Deployment using Jenkins Pipeline

This guide delivers a comprehensive, step-by-step methodology for deploying a Node.js application to a remote server using Jenkins. It is designed to ensure a streamlined, reliable, and repeatable deployment workflow, suitable for users at any experience level.

---  
                      
##  Project Overview

Infrastructure Components:

* **Jenkins Server**: Runs the Jenkins job and orchestrates the deployment.
* **App Server**: Remote server where the Node.js app is deployed and run.
* **Deployment Method**: Declarative Jenkins Pipeline with Git, SSH, and PM2.
* **Pipeline Stages**:

  1. Clone source code from GitHub.
  2. Upload files to remote app server.
  3. Install dependencies & start the app using PM2.

**Plugins Used**:

* [Git Plugin](https://plugins.jenkins.io/git/)
* [SSH Agent Plugin](https://plugins.jenkins.io/ssh-agent/)
* [Pipeline Plugin](https://plugins.jenkins.io/workflow-aggregator/)

---

## Prerequisites

### 1. Infrastructure

* **Two servers** (can be AWS EC2, Azure VM, etc.):

  * **Jenkins Server** (to run the pipeline)
  * **App Server** (to host the Node.js application)
* Both servers launched using the **same `.pem` key**.

### 2. Software Requirements

| Component         | Jenkins Server | App Server   |
| ----------------- | -------------- | ------------ |
| Java (OpenJDK 17) | ✅ Required     | ✅ Required   |
| Node.js & NPM     | ❌ Not needed   | ✅ Required   |
| PM2               | ❌ Not needed   | ✅ Required   |
| Git               | ✅ Required     | ✅ Required   |
| Jenkins           | ✅ Required     | ❌ Not needed |

**Java Installation (OpenJDK 17)**:

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```

**Node.js & PM2 Installation (App Server only)**:

```bash
sudo apt install -y nodejs
sudo npm install -g pm2
```

##  Security Group / Firewall Rules

Open these ports in your **cloud security group** (or firewall):

| Port   | Purpose                          | Where to Open                                      |
| ------ | -------------------------------- | -------------------------------------------------- |
| 22     | SSH access                       | Jenkins → App Server, Your PC → Jenkins/App Server |
| 8080   | Jenkins web interface            | Your PC → Jenkins Server                           |
| 3000\* | Node.js app (or your app's port) | Public or specific IPs                             |

> **Tip:** Replace `3000` with the actual port your app listens on.

---

## Setting Up Jenkins Credentials

We are reusing the **same `.pem` key** that was used to launch both the Jenkins server and the App server.
This means Jenkins can SSH into the app server without generating a new key.

### Step 1 — Copy Your PEM Key to Jenkins Server

From your **local machine**:

```bash
scp -i pem-key-server.pem pem-key-server.pem ubuntu@<JENKINS_SERVER_PUBLIC_IP>:/home/ubuntu/
```
### Step 2 — Add PEM Key to Jenkins Credentials

1. Under **Stores scoped to Jenkins**, click **(global)** → **Add Credentials**.
   
2. Fill in:

   * **Kind**: SSH Username with private key
   * **Username**: `ubuntu` (or the SSH username for your app server)
   * **Private Key**: Choose **"Enter directly"** and paste the contents of `pem-key-server.pem`
   * **Host**: Enter the public IP of your app server (e.g., `98.81.210.251`)
   * **ID**: `nodejs-app-credentials`
3. Save.


---

##  Jenkins Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any

    environment {
        SERVER_IP      = '<public-ip of app-server>'
        SSH_CREDENTIAL = 'nodejs-app-credentials'
        REPO_URL       = 'https://github.com/Amogh902/node-js-app-CICD.git'
        BRANCH         = 'main'
        REMOTE_USER    = 'ubuntu'
        REMOTE_PATH    = '/home/ubuntu/node-app'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: "${BRANCH}", url: "${REPO_URL}"
            }
        }

        stage('Upload Files to EC2') {
            steps {
                sshagent([SSH_CREDENTIAL]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} 'mkdir -p ${REMOTE_PATH}'
                        scp -o StrictHostKeyChecking=no -r * ${REMOTE_USER}@${SERVER_IP}:${REMOTE_PATH}/
                    """
                }
            }
        }

        stage('Install Dependencies & Start App') {
            steps {
                sshagent([SSH_CREDENTIAL]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} '
                            cd ${REMOTE_PATH} &&
                            npm install &&
                            pm2 start app.js --name node-app || pm2 restart node-app
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Application deployed successfully!'
        }
        failure {
            echo '❌ Deployment failed.'
        }
    }
}
```

---

1. **Job Configuration** showing "Pipeline script from SCM".
   ![](/nodejs-app-img/job-configuration.png)
2. **Build Console Output** showing successful deployment logs.
   ![Screenshot Placeholder: Console Output](/nodejs-app-img/successful-build.png)
3. **Browser** showing your running Node.js app.
   ![Screenshot Placeholder: App Running in Browser](/nodejs-app-img/final-output-1.png)

  

---

##  How It Works

1. **Clone Repository** – Pulls Node.js app code from GitHub.
2. **Upload Files** – Uses SSH to copy files to app server.
3. **Install & Run** – Installs dependencies and starts app with PM2.
4. **Access App** – Open `http://<APP_SERVER_IP>:<PORT>` in your browser.

---

##  Example Access URL

```
http://<public-ip of app-server>:3000
```
