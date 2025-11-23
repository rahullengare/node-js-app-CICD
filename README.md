# node-js-app-CICD
**Jenkins-based CI/CD pipeline for deploying a Node.js app, running all stages sequentially—from cloning the repo to uploading files, installing dependencies, and starting the application.**

## **About the Project:**

This project demonstrates how to build a fully automated **CI/CD pipeline** using **Jenkins Pipeline (Declarative Pipeline + Jenkinsfile)** to deploy a Node.js application to a remote EC2 instance. The pipeline executes all stages in sequence—starting from cloning the source code to installing dependencies, uploading build artifacts, and starting the application using PM2 on a dedicated deployment server.

This system ensures that every code change pushed to the GitHub repository triggers a reliable, repeatable deployment workflow without any manual involvement.

---

## Process:

The goal of this project is to implement a **complete end-to-end CI/CD process** for a Node.js application using **Jenkins Pipeline Jobs**. All stages are defined inside a `Jenkinsfile`, enabling smooth automation, scalability, and version-controlled CI/CD logic.

The pipeline consists of the following automated stages:

1. **Clone Repository** – Fetches the latest code from GitHub
2. **Upload Files to EC2** – Securely transfers updated code to the deployment server
3. **Install Dependencies** – Runs NPM installation on remote server
4. **Start Application** – Launches or restarts the app using PM2
5. **Continuous Delivery** – Ensures that each new commit automatically deploys to the server

This approach reflects modern DevOps practices where Jenkins Pipelines are used to achieve controlled, repeatable and fully automated application delivery.

---

## **Technologies Used**

- **Jenkins** → CI/CD automation system
- **Pipeline (Jenkinsfile)** → Defines the complete workflow
- **SSH Agent Plugin** → Secure authentication for remote EC2 access
- **GitHub** → Source code repository and webhook integration
- **Node.js** → Application runtime
- **NPM** → Dependency management
- **PM2** → Process manager for keeping Node.js app alive
- **AWS EC2** → Deployment server (nodeapp-server)
- **Key Pair Authentication** → Secure remote command execution

---

## **Prerequisites**

- Jenkins installed on a dedicated EC2 instance
- GitHub repository containing a Node.js application and Jenkinsfile
- SSH private key copied and configured in Jenkins Credentials
- AWS EC2 (Ubuntu) instance prepared as a deployment target
- Node.js, PM2, and NPM installed on deployment server
- Proper security group rules (allow port 3000)
- Jenkins Pipeline Plugins installed

---

## **How the CI/CD Pipeline Works**

This Jenkins-based CI/CD pipeline is fully declarative and divided into clear stages:

- Jenkins automatically pulls the latest commit from GitHub
- Files are securely transferred to the deployment EC2 instance using SCP
- Node.js dependencies are installed on the server
- PM2 starts or restarts the app to apply changes
- The application becomes live instantly on the public IP at port **3000**

All automation logic resides in the Jenkinsfile, making the pipeline version-controlled and easy to modify over time.

---

## Step 1: Start Jenkins Server

1. Go to AWS console → EC2 Services
2. Start Jenkins server 
3. You need to already created on instance that is installed jenkins server
4. connect instance via ssh 
5. hostname set - jenkins

![Project Screenshot](/images/ssh-jenkins.png).
---

## Step 2: Launch instance(nodeapp-server)

1. Go to AWS console → EC2 Services
2. Click on create instance then
    - Name: nodeapp-server
    - AMI: Ubuntu
    - Key: pem-server-key
    - Security Group: launch-wizard-2 (allow-3000)
3. connect instance via ssh 
4. hostname set - nodeapp
5. Install all dependency's 

```bash
sudo apt upade 
sudo apt install node -y
node -v
sudo npm -g install pm2
npm -v
pm2 -v
```

6. copy key local to jenkins server

```bash
scp -i pem-server-key.pem pem-server-key.pem ubuntu@54.173.116.190:/home/ubuntu/
```
![Project Screenshot](/images/nodeapp-server.png).
![Project Screenshot](/images/ssh-nodeapp.png).

---

## Step 3: Create Repo

1. create one repo in GitHub
2. Name : **node-js-app-CICD**
3. Desc :  Jenkins-based CI/CD pipeline for deploying a Node.js app, running all stages sequentially—from cloning the repo to uploading files, installing dependencies, and starting the application.
4. add 3 files following

**app.js**

```bash
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('Hello from jenkins, added webhook, this is the third version of node');
});

app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});
```

**package.json**

```bash
{
  "name": "jenkins-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

**jenkinsfile**

  

```bash
pipeline {
    agent any

    environment {
        SERVER_IP      = '98.89.20.143'
        SSH_CREDENTIAL = 'node-app-key'
        REPO_URL       = 'https://github.com/rahullengare/node-js-app-CICD.git'
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
![Project Screenshot](/images/repo-create.png).
---

## Step 4: Install Plugins & Create Job

1. install 3 major plugin you need to this project
    - Pipeline
    - ssh agent
    - github
2. restart your jenkins server 
3. Create JOB 
    - name - **nodeapp-deployment**
    - desc - nodeapp deploy pull code from repository
    - pipeline - select (Pipeline script from SCM)
    - SCM → Git then Repository URL → <your repo url>
    - Script Path → jenkinsfile
4. Click to save 
5. Click to Build Now 

![Project Screenshot](/images/job-create.png).
![Project Screenshot](/images/jobs-scm.png).
![Project Screenshot](/images/build-output.png).

---

## Step 5: Check nodeapp

1. Copy node public ip then paste to search with port number 

```bash
http://98.89.20.143:3000/
```

2. Now your node application run successfully 

![Project Screenshot](/images/nodeapp-done.png).

---

## Step 6: Now all Instances Terminated

1. Go to AWS console → EC2 Services
2. Select Your instance you want to deleted
3. Click on terminate

![Project Screenshot](/images/delete-instance.png).
