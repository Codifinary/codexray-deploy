# git install
    sudo apt install git


# Github ssh key add


### Ssh key generation 
    ssh-keygen -t ed25519 -C "your_email@example.com"

### Start the SSH agent:
    eval "$(ssh-agent -s)"

### Add your SSH private key to the agent:
    ssh-add ~/.ssh/id_ed25519

### copy ssh key 
    cat ~/.ssh/id_ed25519.pub

### Add this ssh public key to Codifinary repo/account

# Docker install 
###  Add Docker's official GPG key:
        sudo apt-get update
        sudo apt-get install ca-certificates curl
        sudo install -m 0755 -d /etc/apt/keyrings
        sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
        sudo chmod a+r /etc/apt/keyrings/docker.asc
        
        # Add the repository to Apt sources:
        echo \
          "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
          $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
          sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        sudo apt-get update
        
### Install the Docker packages.
     sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


### docker download 
    sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

###  Apply executable permissions:
    sudo chmod +x /usr/local/bin/docker-compose

### Setup docker login for Github 
    docker login ghcr.io -u <access username or email> -p <PAT>
