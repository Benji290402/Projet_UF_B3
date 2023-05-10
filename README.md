
# Projet B3 UF
Bienvenue dans la documentation d'installation du Service 2 Offert par YOméga


## Installation d'Ansible
Pour ce projet, vous allez nécéssités 2 Nodes Vagrant ou bien 2 VMs distinctes
 - [Installation d'Ansible et ses modules](https://awesomeopensource.com/project/elangosundar/awesome-README-templates)
Deux solutions, soit vous cloner le projets et vous executez simplement le script **install_ansible**, soit vous copiez les commandes.
```bash
# Installation de pip3 et d'Ansible
sudo apt update
sudo apt install -y python3-pip
pip3 install ansible

# Installation de la collection Docker de la communauté Ansible
ansible-galaxy collection install community.docker
```
 - [Création des Clés SSH]
Une fois ansible installer vous pouvez voir la version qui est installer sur ansible avec la commande 
```bash
ansible --version
```
Donc vous devez utiliser des Clés SSH pour pouvoir comuniquer avec Ansible de manière sécurisé, pour ce faire
```bash
ssh-keygen
```
**Il est important de mettre une passphrase sécurisée**
![App Screenshot](https://github.com/Benji290402/Projet_UF_B3/blob/main/sc12.PNG)
Ensuite vous pouvez poussez votre clé via SSH dans votre deuxième noeud, VM
```bash
ssh-copy-id user@ip-de-votre-node
```
 - [Mise en place de l'inventaire](https://bulldogjob.com/news/449-how-to-write-a-good-readme-for-your-github-project)
Une fois les clés SSH exporter, on va mettre en place notre Inventaire

