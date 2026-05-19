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

8 - When you first access a new Jenkins controller, you are asked to unlock it using an automatically-generated password. 

    # You need to retrieve the Jenkins container id or name using docker ps; use it as input on the second command.

    docker ps

    sudo docker exec ${CONTAINER_ID or CONTAINER_NAME} cat /var/jenkins_home/secrets/initialAdminPassword

    # Copy the password

9 - On the Unlock Jenkins page (http://{HostIPAddress}:8080), paste this password into the Administrator password field and click Continue. Note: make sure your cloud firewall rules allow inbound TCP 8080. 

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
