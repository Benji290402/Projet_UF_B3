
# Projet B3 UF
Bienvenue dans la documentation d'installation des Services offert par YOméga

## Sommaire

Cliquez-ici pour voir le topic qui vous intéresse directement.

[Installation Ansible](#installation-dansible)<br>
[Utilisation Docker & Playbook](#utilisation-docker--déploiement-des-playbooks)<br>
[Déployer Wordpress](#déployer-wordpress)

## Installation d'Ansible
Pour ce projet, vous allez avoir besoin au choix de 2 Nodes Vagrant ou bien de 2 VMs distinctes
 - [Installation d'Ansible et ses modules](https://github.com/Benji290402/Projet_UF_B3/blob/main/install_ansible.sh)<br>
Deux solutions, soit vous clonez le projet et vous exécutez simplement le script **install_ansible**, soit vous copiez les commandes.
```bash
# Installation de pip3 et d'Ansible
sudo apt update
sudo apt install -y python3-pip
pip3 install ansible

# Installation de la collection Docker de la communauté Ansible
ansible-galaxy collection install community.docker
```
 - Création des Clés SSH <br>
Une fois Ansible installé vous pouvez voir la version qui est installée sur Ansible avec la commande 
```bash
ansible --version
```
Vous devez donc utiliser des Clés SSH pour pouvoir comuniquer avec Ansible de manière sécurisée, on va donc créer une paire de clés SSH
```bash
ssh-keygen
```
**Il est important de mettre une passphrase sécurisée** <br>
![App Screenshot](https://github.com/Benji290402/Projet_UF_B3/blob/main/sc12.PNG)<br>
Ensuite vous pouvez pousser votre clé via SSH dans votre deuxième noeud ou VM
```bash
ssh-copy-id user@ip-de-votre-node
```
 - [Mise en place de l'inventaire](https://github.com/Benji290402/Projet_UF_B3/blob/main/inv.ini)<br>
Une fois les clés SSH exportées, on va mettre en place notre Inventaire, pour se faire nous créeons un fichier **inv.ini**

```bash
[nc] #Nom de votre groupe d'hotes

nc1 ansible_host=IP-CIBLE ansible_ssh_user=USR ansible_become_password=VOTRE-MDP
```

Afin de tester votre connectivité, placez-vous dans le répertoire ou votre fichier d'inventaire existe et exécutez la commande suivante : 

```bash
ansible all -i inv.ini -m ping
```
- [Exécution de votre premier playbook](https://github.com/Benji290402/Projet_UF_B3/blob/main/dockerinstall.yaml)

On va pouvoir désormais s'occuper du deuxième Node / VM qui est vierge. Pour cela on va exécuter le playbook **dockerinstall.yaml**

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
        - docker.io            #Reviens a faire un apt install docker.io
      update_cache: yes

  - name: add user permissions
    shell: "usermod -aG babou {{ ansible_env.SUDO_USER }}"  #Modifier "babou" par votre utilisateur

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
Une fois le playbook dans notre répertoire où se trouve notre inventaire, on va exéctuer la commande : 

```bash
ansible-playbook dockerinstall.yaml -i inv.ini
```

Une fois l'installation terminée, on se connecte à notre deuxième Node, et on effectue la commande suivante :  

```bash
docker --version
Docker version 20.10.21, build 20.10.21-0ubuntu1~20.04.2
```
On va pouvoir passer à l'installation & l'utilisation de Docker

## Utilisation Docker & Déploiement des playbooks
On va maintenant pouvoir exploiter Ansible et notre environement Docker, pour cela on va utiliser deux applications conteneurisées qui vont nous faciliter l'exploitation de notre environement, à savoir **Portainer** et **Watchtower**
- [Déploiement de Portainer et Watchtower](https://github.com/Benji290402/Projet_UF_B3/blob/main/portainerinstall.yml)

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

On va pouvoir exécuter le playbook, toujours avec la même commande, on cible juste notre nouveau fichier YAML,

```bash
ansible-playbook -i inv.ini portainerinstall.yml
```

Ensuite on se rend notre deuxième noeud, et on exécute la commande : 

```bash
sudo docker ps -a
```
Et on devrait obtenir cette sortie : 

```bash
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                      NAMES
1f4e00571380   containrrr/watchtower    "/watchtower --sched…"   14 minutes ago   Up 14 minutes   8080/tcp                                                   watchtower
7a39fdc4d9e6   portainer/portainer-ce   "/portainer"             17 minutes ago   Up 17 minutes   0.0.0.0:8000->8000/tcp, 0.0.0.0:9000->9000/tcp, 9443/tcp   portainer
```
On peut se connecter sur notre navigateur à Portainer et commencer à créer le compte utilisateur admin. Cette application est extrêmement pratique pour visualiser l'infrastructure conteneurisée.

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
On exécute le playbook :

```bash
ansible-playbook -i inv.ini deploynxt.yml
```

Et on devrait pouvoir se connecter en mettant l'adresse ip de notre machine et son port!

## Déployer Wordpress

On va désormais passer au déploiement de notre site wordpress, avec une base de donnée MySQL et le conteneur Wordpress

- [Exécution du playbook](https://github.com/Benji290402/Projet_UF_B3/blob/main/deploywp.yml)

Comme pour le playbook précédent, on va importer le playbook *deploywp.yml*

```yaml
---
- hosts: all
  become: yes
  tasks:

    - name: Create Network
      community.docker.docker_network:
        name: wordpress

    - name: Deploy Wordpress
      community.docker.docker_container:
        name: wordpress
        image: wordpress:latest
        ports:
          - "80:80"
        networks:
          - name: wordpress
        volumes:
          - wordpress:/var/www/html
        env:
          WORDPRESS_DB_HOST: "db"
          WORDPRESS_DB_USER: "exampleuser"
          WORDPRESS_DB_PASSWORD: "examplepass"
          WORDPRESS_DB_NAME: "exampledb"
        restart_policy: always

    - name: Deploy MYSQL
      community.docker.docker_container:
        name: db
        image: mysql:5.7
        ports:
          - "3306:3306"
        networks:
          - name: wordpress
        volumes:
          - db:/var/lib/mysql
        env:
          MYSQL_DATABASE: "exampledb"
          MYSQL_USER: "exampleuser"
          MYSQL_PASSWORD: "examplepass"
          MYSQL_RANDOM_ROOT_PASSWORD: '1'
        restart_policy: always
```

On effectue la commande : 
```bash
ansible-playbook -i ini.inv deploywp.yml
```
Pour finir on essaie d'afficher les conteneurs qui tournent : 
```bash
sudo docker ps
```
Et voici la sortie que l'on devrait obtenir, 
```bash
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS                                                      NAMES
636ed024935f   mysql:5.7                "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   0.0.0.0:3306->3306/tcp, 33060/tcp                          db
b2bb267f7464   wordpress:latest         "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   0.0.0.0:80->80/tcp                                         wordpress
```
Pour conclure on se lance dans l'installation de Wordpress depuis notre navigateur et le tour est joué ! 

**Merci d'avoir suivi ce tutoriel d'installation.** 


