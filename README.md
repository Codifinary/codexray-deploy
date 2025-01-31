# Github ssh key add


# Ssh key generation 
ssh-keygen -t ed25519 -C "your_email@example.com"

# Start the SSH agent:
eval "$(ssh-agent -s)"

# Add your SSH private key to the agent:
ssh-add ~/.ssh/id_ed25519

# copy ssh key 
cat ~/.ssh/id_ed25519.pub
