# Projet_UF_B3
Pour pouvoir executer le projets de manière simple, il vous faut 2 Nodes Vagrant, ou deux VMs distinctes
# Installation d'Ansible

Sur votre 1er Node / VM, vous allez executer le script **install_ansible.sh** 
```shell  #!/bin/bash

# Installation de pip3 et d'Ansible
sudo apt update
sudo apt install -y python3-pip
pip3 install ansible

# Installation de la collection Docker de la communauté Ansible
ansible-galaxy collection install community.docker
```shell
ssh-keygen

