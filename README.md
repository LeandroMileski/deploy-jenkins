# deploy-jenkins
The platform team commits infra changes to git -- the pipeline applies them. No more manual terraform runs.

1 - spin up a server, on any cloud platform.

2 - SSH into the server. Install docker and set permissions:
    
    sudo snap install docker
    sudo addgroup --system docker
    sudo adduser $USER docker
    
    # apply changes to the running system
    newgrp docker
    sudo snap disable docker
    sudo snap enable docker

    # run test command, if ok: "Hello from Docker!"
    docker run hello-world

3 - First create a bridge network:

    docker network create jenkins

4 - In order to execute Docker commands inside Jenkins nodes, download and run the docker:dind Docker image using the following docker run command:

    docker run \
    --name jenkins-docker \
    --rm \
    --detach \
    --privileged \
    --network jenkins \
    --network-alias docker \
    --env DOCKER_TLS_CERTDIR=/certs \
    --volume jenkins-docker-certs:/certs/client \
    --volume jenkins-data:/var/jenkins_home \
    --publish 2376:2376 \
    docker:dind \
    --storage-driver overlay2

5 - Create a Dockerfile (vim Dockerfile) with the following content:

    FROM jenkins/jenkins:2.555.2-jdk21
    USER root
    RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
        install -m 0755 -d /etc/apt/keyrings && \
        curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
        chmod a+r /etc/apt/keyrings/docker.asc && \
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
        https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
        | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
        apt-get update && apt-get install -y docker-ce-cli && \
        apt-get clean && rm -rf /var/lib/apt/lists/*
    USER jenkins
    RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"

6 - Build a new docker image from this Dockerfile, and assign the image a meaningful name, such as "my-jenkins-ocean-blue:22000-1":

    docker build -t my-jenkins-ocean-blue:22000-1 .

7 - Run your own my-jenkins-ocean-blue:22000-1 image as a container in Docker using the following docker run command:

    docker run \
    --name jenkins-blueocean \
    --restart=on-failure \
    --detach \
    --network jenkins \
    --env DOCKER_HOST=tcp://docker:2376 \
    --env DOCKER_CERT_PATH=/certs/client \
    --env DOCKER_TLS_VERIFY=1 \
    --publish 8080:8080 \
    --publish 50000:50000 \
    --volume jenkins-data:/var/jenkins_home \
    --volume jenkins-docker-certs:/certs/client:ro \
    my-jenkins-ocean-blue:22000-1

8 - When you first access a new Jenkins controller, you are asked to unlock it using an automatically-generated password. It's located in /var/jenkins_home/secrets/initialAdminPassword. 

    sudo docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword

    # Copy the password

9 - On the Unlock Jenkins page (http://{HostIPAddress}:8080), paste this password into the Administrator password field and click Continue. Note: Just make sure port 8080 is open in your cloud firewall/security group before hitting the UI. 

<img width="1465" height="613" alt="image" src="https://github.com/user-attachments/assets/792271e2-6810-4aa4-a4d2-fcf6cd02b638" />


10 - Click one of the two options shown:

    Install suggested plugins - to install the recommended set of plugins, which are based on most common use cases.

    Select plugins to install - to choose which set of plugins to initially install. When you first access the plugin selection page, the suggested plugins are selected by default.

    If you are not sure what plugins you need, choose Install suggested plugins. You can install or remove additional Jenkins plugins later via the Manage Jenkins > Plugins page in Jenkins.

11 - Creating the first administrator user:

    Finally, after customizing Jenkins with plugins, Jenkins asks you to create your first administrator user.

    When the Create First Admin User page appears, specify the details for your administrator user in the respective fields and click Save and Finish.

    When the Jenkins is ready page appears, click Start using Jenkins.
    Notes:

    This page may display Jenkins is almost ready! instead and if so, click Restart.

    If the page does not automatically refresh after a minute, use your web browser to refresh the page manually.

    If required, log in to Jenkins using the credentials of the user you just created and you are ready to start using Jenkins!

12 - How to set AWS credentials in Jenkins: (Option 1)

    In Jenkins: Manage Jenkins → Credentials → System → Global → Add Credentials

    (You can get from C:\Users\{yourUSER}\.aws\credentials if Windows)
    Kind: Secret text
    ID: aws-access-key-id → value(Secret): your access key
    ID: aws-secret-access-key → value(Secret): your secret key

Then in your Jenkinsfile:

    pipeline {
      agent any
    
      environment {
        AWS_DEFAULT_REGION = "eu-west-1"
      }
    
      stages {
        stage('Init') {
          steps {
            withCredentials([
              string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
              string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
            ]) {
              sh 'terraform init'
            }
          }
        }
      }
    }

Note: If you're using static keys, wrap each sh block in withCredentials as shown in the previous file.

13 - IAM role on the Jenkins EC2 instance: (Option 2 - Recommended for teams) 

    The rule of thumb: if something runs on AWS infrastructure, give it a role. Only use static keys for things running outside AWS (like a local machine or a third-party CI system).
    
    If Jenkins runs on an EC2 instance, attach an IAM role to that instance with the permissions Terraform needs. No credentials stored anywhere — the AWS SDK picks them up automatically from the instance metadata.
    In AWS:

    Create an IAM role with the necessary policies (AmazonEKSFullAccess, IAMFullAccess, AmazonVPCFullAccess, etc.)
    Attach the role to the Jenkins EC2 instance: EC2 → Actions → Security → Modify IAM role

Your Jenkinsfile then needs no credential handling at all:

    pipeline {
      agent any
    
      environment {
        AWS_DEFAULT_REGION = "eu-west-1"
      }
    
      stages {
        stage('Init') {
          steps {
            sh 'terraform init'
          }
        }
    
        stage('Plan') {
          steps {
            sh 'terraform plan -out=tfplan'
          }
        }
    
        stage('Approve') {
          steps {
            input message: 'Apply this plan?', ok: 'Apply'
          }
        }
    
        stage('Apply') {
          steps {
            sh 'terraform apply tfplan'
          }
        }
      }
    }

If Jenkins runs on EC2 with an IAM role there's no withCredentials block needed anywhere — the pipeline is clean as shown. If you're using static keys, wrap each sh block in withCredentials as shown in the previous answer.


