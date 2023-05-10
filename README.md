
# Projet B3 UF
Bienvenue dans la documentation d'installation du Service 2 Offert par YOméga

## Sommaire

Cliquez-ici pour voir le topic qui vous interesse directement

[Installation Ansible](https://github.com/Benji290402/Projet_UF_B3/blob/main/README.md#installation-dansible)<br>
[Utilisation Docker & Playbook](https://github.com/Benji290402/Projet_UF_B3/blob/main/README.md#installation-dansible)


## Installation d'Ansible
Pour ce projet, vous allez nécéssités 2 Nodes Vagrant ou bien 2 VMs distinctes
 - [Installation d'Ansible et ses modules](https://github.com/Benji290402/Projet_UF_B3/blob/main/install_ansible.sh)<br>
Deux solutions, soit vous cloner le projets et vous executez simplement le script **install_ansible**, soit vous copiez les commandes.
```bash
# Installation de pip3 et d'Ansible
sudo apt update
sudo apt install -y python3-pip
pip3 install ansible

# Installation de la collection Docker de la communauté Ansible
ansible-galaxy collection install community.docker
```
 - Création des Clés SSH <br>
Une fois ansible installer vous pouvez voir la version qui est installer sur ansible avec la commande 
```bash
ansible --version
```
Donc vous devez utiliser des Clés SSH pour pouvoir comuniquer avec Ansible de manière sécurisé, pour ce faire
```bash
ssh-keygen
```
**Il est important de mettre une passphrase sécurisée** <br>
![App Screenshot](https://github.com/Benji290402/Projet_UF_B3/blob/main/sc12.PNG)<br>
Ensuite vous pouvez poussez votre clé via SSH dans votre deuxième noeud, VM
```bash
ssh-copy-id user@ip-de-votre-node
```
 - [Mise en place de l'inventaire](https://github.com/Benji290402/Projet_UF_B3/blob/main/inv.ini)<br>
Une fois les clés SSH exporter, on va mettre en place notre Inventaire, pour se faire on va créer un fichier **inv.ini**

```bash
[nc] #Nom de votre groupe d'hotes

nc1 ansible_host=IP-CIBLE ansible_ssh_user=USR ansible_become_password=VOTRE-MDP
```

Afin de tester votre connectiviter, placer-vous dans le repertoire ou votre fichier d'inventaires existe et executez la commande : 

```bash
ansible all -i inv.ini -m ping
```
- [Execution de votre premier playbook](https://github.com/Benji290402/Projet_UF_B3/blob/main/dockerinstall.yaml)

On va pouvoir désormais s'occuper du deuxième Node / VM qui est vierge. Pour cela on va executer le playbook **dockerinstall.yaml**

```yaml
---
- hosts: all
  
  become: yes
  tasks:

  # Install Docker
  # --
  # 
  - name: install prerequisites
    apt:
      name:
        - docker.io
      update_cache: yes

  - name: add user permissions
    shell: "usermod -aG babou {{ ansible_env.SUDO_USER }}"

  - name: Reset ssh connection for changes to take effect
    meta: "reset_connection"

  # Installs Docker SDK
  # --
  # 
  - name: install python package manager
    apt:
      name: python3-pip
  
  - name: install python sdk
    become_user: "{{ ansible_env.SUDO_USER }}"
    pip:
      name:
        - docker
        - docker-compose
```
Une fois le playbook dans notre répertoire ou se trouve notre inventaire, on va exectuer la commande : 

```bash
ansible-playbook dockerinstall.yaml -i inv.ini
```

Une fois l'installation terminée, on se connecte à notre deuxième Node, 

```bash
docker --version
Docker version 20.10.21, build 20.10.21-0ubuntu1~20.04.2
```
On va pouvoir passer à l'installation & l'utilisation de Docker

## Utilisation Docker & Déploiement des playbooks
On va maintenant pouvoir exploiter Ansible et notre environement Docker, pour cela on va utiliser deux applications conteneuriser qui vont nous faciliter l'exploitation de notre environement, à savoir **Portainer** et **Watchtower**
- [Execution de votre premier playbook](https://github.com/Benji290402/Projet_UF_B3/blob/main/portainerinstall.yml)

```yaml
- hosts: all
  become: yes
  tasks:

    - name: Deploy Portainer
      community.docker.docker_container:
        name: portainer
        image: portainer/portainer-ce
        ports:
          - "9000:9000"
          - "8000:8000"
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - portainer_data:/data
        restart_policy: always

      name: Deploy Watchtower
      community.docker.docker_container:
        name: watchtower
        image: containrrr/watchtower
        command: --schedule "0 0 4 * * *" --debug
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        restart_policy: always
```

On va pouvoir executer le playbook, toujours avec la meme commande, on cible juste notre nouveau fichier yaml

```bash
ansible-playbook -i inv.ini portainerinstall.yml
```

Ensuite on se rends notre deuxième noeud, et on execute la commande 

```bash
sudo docker ps -a
```
Et on devrait avoir comme sortie

```bash
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                      NAMES
1f4e00571380   containrrr/watchtower    "/watchtower --sched…"   14 minutes ago   Up 14 minutes   8080/tcp                                                   watchtower
7a39fdc4d9e6   portainer/portainer-ce   "/portainer"             17 minutes ago   Up 17 minutes   0.0.0.0:8000->8000/tcp, 0.0.0.0:9000->9000/tcp, 9443/tcp   portainer
```
On peut se connecter sur notre navigateur à Portainer et commencer à créer le compte utilisateur admin. Cette application est extremement pratique pour visualiser l'infrastructure conteneuriser.

- [Déploiement du conteneur Nextcloud](https://github.com/Benji290402/Projet_UF_B3/blob/main/deploynxt.yml)

On va récuperer le fichier *deploynxt.yml*

```yaml
---
- hosts: all
  become: yes
  vars:
    db_volume: mariadb
    nextcloud: nextcloud
  tasks:
    - name: Deploy MariaDB server
      docker_container:
        image: mariadb
        name: mariadb
        volumes:
          - "{{db_volume}}:/var/lib/mysql"
        env:
          MYSQL_ROOT_PASSWORD: somerootpassword #Mettez un mot de passe fort
          MYSQL_PASSWORD: somemysqlpassword #Mettez votre mot de passe
          MYSQL_DATABASE: db
          MYSQL_USER: mysqluser #Changez si vous le souhaitez
    - name: Deploy Nextcloud
      docker_container:
        image: nextcloud
        name: nextcloud
        restart_policy: always
        ports:
          - 80:80
        links:
          - "{{db_volume}}:/var/lib/mysql"
        volumes:
          - "{{nextcloud}}:/var/www/html"
        env:
          MYSQL_PASSWORD: somemysqlpassword #Identique a celui dans la variable Env
          MYSQL_DATABASE: db
          MYSQL_USER: mysqluser #Identique a celui dans la variable Env
          MYSQL_HOST: mariadb
```
On éxécute le playbook :

```bash
ansible-playbook -i inv.ini deploynxt.yml
```

Et on devrait pouvoir se connecter en mettant l'adresse ip de notre machine et son port!

**Merci d'avoir suivis ce tutoriel d'installation.** 
