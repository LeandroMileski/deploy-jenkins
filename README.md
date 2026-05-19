# deploy-jenkins
The platform team commits infra changes to git -- the pipeline applies them. No more manual terraform runs.

1 - spin up a server, on any cloud platform.

2 - SHH into the server. Install docker and set permissions:
    
    sudo snap install docker
    sudo addgroup --system docker
    sudo adduser $USER docker
    
    # apply changes to the running system
    newgrp docker
    sudo snap disable docker
    sudo snap enable docker

    # run test command, if ok: "Hello from Docker!"
    docker run hello-world

3 - 