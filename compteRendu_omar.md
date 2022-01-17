# Création des Rôles Ansible.
Le but étant de déployer des conteneurs docker, nous avons déterminé 4 roles :
* **docker_role** : qui permet d'installer tous les outils necessaire pour la création de conteneur docker et de docker-compose
* **odoo_role** : qui lancera 2 conteneurs "connectés" celui de la base de donnée postgres et celui de odoo
* **pgadmin_role** : qui lancera un conteneur pgadmin (permettant de visualiser la base de donnée postgres de odoo)
* **ic-webapp_role** : lance notre site vitrine ic-webapp.

Pour des raisons d'unicité, et à cause d'un problème rencontré avec pgadmin (voir partie Troubleshooting), nous avons choisi de déployer tous nos conteneur via docker_compose de ansible.
<br>

Pour des raisons économique et fonctionnelles nous avons décider que :
* Les machines cibles seront des serveur ubuntu 20.04 LTS (sur aws)
* La région sera la Virginie du Nord
* L'utilisateur par défaut du système sera "ubuntu"

## 1. Role Docker
source : https://github.com/lianhuahayu/docker_role.git<br>
* Installe Docker via le script officiel "get-docker.sh"
* Ajoute et lance le service docker au démarrage du serveur
* Ajoute l'utilisateur "ubuntu" au groupe docker
* Installe python3-pip 
* Installe docker-compose (via le script officiel)
* Installe docker-compose via pip (indispensable pour les commandes ansible docker_compose)

#### ***`tasks/main.yml`***
```yml
- name: "pre-requis pour installer docker"
  package:
    name: curl
    state: present

- name: "Recuperation du script d installation de Docker"
  command: 'curl -fsSL https://get.docker.com -o get-docker.sh'

- name: "Executer le script d'installation de Docker"
  command: "sh get-docker.sh"
  when: ansible_docker0 is undefined

- name: "Démarrage et ajout au redémarrage du service Docker"
  service:
    name: docker
    state: started
    enabled: yes

- name: "Ajout de notre utilisateur au groupe docker"
  user:
    name: "{{ ansible_user }}"
    append: yes
    groups:
        - docker

- name: "Installation de  python pip pour ubuntu"
  apt:
    name: python3-pip
    state: present
  when: ansible_distribution == "Ubuntu"

- name: "prerequis dockercompose"
  command: 'curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose'

- name: "install docker cmpose"
  command: 'chmod +x /usr/local/bin/docker-compose'

- name: "Installation du module docker-compose"
  pip:
    name: docker-compose
    state: present
```

## 2. Role Odoo 
source : https://github.com/lianhuahayu/odoo_role.git<br>
* Va déployer 2 conteneurs avec le template docker-compose:
    * conteneur backend_odoo 
        * basé sur une image postres:13 
        * exposant le port 5432 
        * initialisant les variables d'environnement avec 
            * postgres_user (utilisateur de la database)
            * postgres_password (mot de passe de la database)
            * postgres_database (nom de la database)
        * persistant les données avec un volume
    * conteneur frontend_odoo
        * basé sur une image odoo:13
        * exposant le port 8069
        * initialisant les variables d'environnement avec celles de postgres
        * spécifiant aussi l'adresse ip du host de postgres
        * lancant une commande odoo permettant d'initialiser la database pour ne pas le faire manuellement
        * persistant les données avec un volume
* Les deux conteneurs seront sur un network commun
* Toutes les données sont variabilisées donc pourront être surchargée par ansible (-e )
#### ***`defaults/main.yml`***
```yml
---
# Utilisateur par defaut
ansible_user: user
ansible_sudo_pass: user

# Postgres variables
postgres_container: backend_odoo
postgres_image: postgres:10
postgres_port: 5432
postgres_database: odoo
postgres_user: odoo
postgres_password: odoo
postgres_data: backup-postgres-db-data

# Odoo variables
odoo_container: frontend_odoo
odoo_image: odoo:13.0
odoo_port: 8069
odoo_data: backup-odoo-data

# Nom du réseau
network_name: odoo
```

#### ***`tasks/main.yml`***
```yml
---
- name: "Ajout du template docker-compose.yml"
  template:
    src: "docker-compose.yml.j2"
    dest: /home/{{ ansible_user }}/docker-compose.yml

- name: "Lancer le fichier docker-compose.yml"
  docker_compose:
    project_src: /home/{{ ansible_user }}
    files:
    - docker-compose.yml
    
- name: "Relancer le container odoo apres l'init de la db"
  shell: |
    sleep 15
    docker container restart {{ odoo_container }}
    exit 0
```

#### ***`templates/docker-compose.yml.j2`***
```jinja
version: '2'
services:
  {{ postgres_container }}: 
    container_name: {{ postgres_container }}
    image: {{ postgres_image }}
    restart: always
    ports:
      - "{{ postgres_port }}:5432"
    environment:
      - POSTGRES_DB={{ postgres_database }}
      - POSTGRES_PASSWORD={{ postgres_password }}
      - POSTGRES_USER={{ postgres_user }}
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - {{ postgres_data }}:/var/lib/postgresql/data/pgdata
    networks:
      - {{ network_name }}

  {{ odoo_container }}:
    container_name: {{ odoo_container }}
    image: {{ odoo_image }}
    restart: always
    depends_on:
      - {{ postgres_container }}
    ports:
      - "{{ odoo_port }}:8069"
    environment:
      - HOST={{ postgres_container }}
      - PORT={{ postgres_port }}
      - USER={{ postgres_user }}
      - PASSWORD={{ postgres_password }} 
    command: odoo -i base -d {{ postgres_database }} --db_host={{ postgres_container }} -r {{ postgres_user }} -w {{ postgres_password }} 
    volumes:
      - {{ odoo_data }}:/var/lib/odoo
    networks:
      - {{ network_name }} 

networks:
  {{ network_name }}: 
    name: {{ network_name }}
    driver: bridge

volumes:
  {{ odoo_data }}:
  {{ postgres_data }}:
```

## 3. Role PgAdmin
source : https://github.com/Yellow-carpet/pgadmin_role.git<br>
* Va déployer un conteneur pgAdmin via le template docker-compose
    * basé sur une image de dpage/pgadmin4
    * Initialiser les variables d'environnement avec :
        * pgadmin_email (email et login de l'utilisateur)
        * pgadmin_pass (mot de passe)
        * pgadmin_port (port de l'application par défaut 80)
    * Persistance du fichier /pgadmin4/servers.json permettant à l'initialisation, d'avoir déjà du contenu : celui de la base de donnée odoo
* Va copier le fichier "templatisé" servers.json sur la machine distante contenant toute les informations de la base de donnée odoo
* Toutes les données sont variabilisées donc pourront être surchargée par ansible

#### ***`defaults/main.yml`***
```yml
---
# servers.json variables
host_db: 172.31.82.22
port_db: 5432 
maintenanceDB: postgres 
username_db: odoo

# pgadmin docker-compose variables
pgadmin_email: pgadmin@pgadmin.com
pgadmin_pass: pgadmin
pgadmin_port: 5050
```

#### ***`tasks/main.yml`***
```yml
---
- name: "Ajout du template pgadmin docker-compose.yml"
  template:
    src: "docker-compose.yml.j2"
    dest: /home/{{ ansible_user }}/docker-compose.yml

- name: "Ajout du template servers.json"
  template:
    src: "servers.json.j2"
    dest: /home/{{ ansible_user }}/servers.json


- name: "Lancer le fichier docker-compose.yml pour pgadmin"
  docker_compose:
    project_src: /home/{{ ansible_user }}
    files:
    - docker-compose.yml
```

#### ***`templates/docker-compose.yml.j2`***
```jinja
version: "3.3"
services:
  pgadmin:
     image: dpage/pgadmin4
     restart: always
     environment:
       PGADMIN_DEFAULT_EMAIL: {{ pgadmin_email }} #the username to login to pgadmin
       PGADMIN_DEFAULT_PASSWORD: {{ pgadmin_pass }} # the password to login to pgadmin
     ports:
       - "{{ pgadmin_port }}:80"
     volumes:
       - ./servers.json:/pgadmin4/servers.json # preconfigured servers/connections
```

#### ***`templates/servers.json.j2`***
```jinja
{
  "Servers": {
    "1": {
      "Name": "docker_postgres",  #name of the server in pgadmin
      "Group": "docker_postgres_group", #group of the server in pgadmin
      "Host": "{{ host_db }}", #host of the database (in our case it is equal to the name of the database)
      "Port": {{ port_db }} , #port of the db
      "MaintenanceDB": "{{ maintenanceDB }}", #initial database that we would like to connect to
      "Username": "{{ username_db }}", #username to connect to the db
      "SSLMode": "prefer"
    }
  }
}
```

## 4. Role ic-webapp
source: https://github.com/omarpiotr/ic-webapp_role<br>
* Va déployer un conteneur ic-webapp via le templace docker-compose
    * basé par défaut sur l'image "lianhuahayu/ic-webapp:1.0" variabilisée que l'on devra surcharger par la suite.
    * Initialiser les variables d'environnement avec :
        * odoo_url : l'url de notre site odoo sour la forme http://url-odoo:8069
        * pgadmin_url : l'url de notre site pgAdmin sous la forme http://url-pgadmin:5050
        * ic_webapp_port : port exposé par notre application vers l'extérieur
* Toutes les données sont variabilisées donc pourront être surchargée par ansible

#### ***`defaults/main.yml`***
```yml
# defaults file for ic-webapp_role
ic_webapp_image: "lianhuahayu/ic-webapp:1.0"
odoo_url: "127.0.0.1:8069"
pgadmin_url: "127.0.0.1:5050"
ic_webapp_port: 80
```

#### ***`tasks/main.yml`***
```yml
---
# tasks file for ic-webapp_role

- name: "Ajout du template ic-webapp_role  docker-compose.yml"
  template:
    src: "docker-compose.yml.j2"
    dest: /home/{{ ansible_user }}/docker-compose-ic.yml

- name: "Lancer le fichier docker-compose.yml"
  docker_compose:
    project_src: /home/{{ ansible_user }}
    files:
    - docker-compose-ic.yml
```

#### ***`templates/docker-compose.yml.j2`***
```jinja
version: "3.3"
services:
  ic-webapp:
     image: {{ ic_webapp_image }}
     restart: always
     environment:
       ODOO_URL: {{ odoo_url }}
       PGADMIN_URL: {{ pgadmin_url }}
     ports:
       - "{{ ic_webapp_port }}:8080"
```


## 5. Troubleshooting
Lorsque nous avont déployé pgAdmin à l'aide de docker_container, nous nous sommes rendu compte que le fichier "**`servers.json`**", bien que présent, n'est pas pris en charge par pgAdmin.<br>
Selon la documentation officielle ci-dessous, le fichier servers.json n'est lu qu'une seule et unique fois, c'est au moment du premier lancement. Etant donné que ce fichier était partagé à l'aide de la propriété volume, et suite à de nombreux divers tests manuels que nous avont effectué nous en avont conclu que docker_container:<br>
* 1- lancait d'abord le conteneur
* 2- montait les volume ensuite
De ce fait le fichier n'était pas pris en compte.<br>
Pour résoudre ce problème nous avons utilisé docker_compose.
```
/pgadmin4/servers.json

If this file is mapped, server definitions found in it will be loaded at launch time. 
This allows connection information to be pre-loaded into the instance of pgAdmin in the container. 
Note that server definitions are only loaded on first launch, i.e. when the configuration database is created, and not on subsequent launches using the same configuration database.
```

# Les Playbooks Ansible
Afin de consommer et tester nos roles, nous avons créé 3 playbooks
* playbook_odoo :
    * docker_role
    * odoo_role
* playbook_pgadmin
    * docker_role
    * pgadmin_role
* playbook_ic-webapp
    * docker_role
    * ic-webapp_role

#### ***`roles/requirements.yml`***
```yml
- src: https://github.com/lianhuahayu/docker_role.git
- src: https://github.com/lianhuahayu/odoo_role.git
- src: https://github.com/Yellow-carpet/pgadmin_role.git
- src: https://github.com/omarpiotr/ic-webapp_role.git
```
#### ***`playbook_odoo.yml`***
```yml
---
- name: "deploy Odoo with a role"
  hosts: ansible
  become: true
  roles:
    - docker_role
    - odoo_role 
```
#### ***`playbook_pgadmin.yml`***
```yml
---
- name: "deploy pgadmin with a role"
  hosts: ansible
  become: true
  roles:
    - docker_role
    - pgadmin_role
```
#### ***`playbook_ic-webapp.yml`***
```yml
---
- name: "deploy pgadmin with a role"
  hosts: ansible
  become: true
  roles:
    - docker_role
    - ic-webapp_role
```
#### ***`hosts_.yml`***
```yml
all:
  children:
    ansible:
      hosts:
        localhost:
          ansible_connection: local
          ansible_user: "ubuntu"
          hostname: AnsibleMaster
          ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
```


Afin de tester nos role nous avons créer les deux instances suivantes :
* instance ec2 odoo, sur laquelle on souhaite avoir odoo (frontend + backend)
    * t2.micro
    * ip : 44.201.245.76
    * dns : ec2-44-201-245-76.compute-1.amazonaws.com
    * sécurity Group : 22 / 5432 / 8069
* instance ec2 server, sur laquelle on souhaite avoir pgAdmin et ic-webapp
    * t2.micro
    * ip : 52.87.202.243
    * dns : ec2-52-87-202-243.compute-1.amazonaws.com
    * sécurity Group : 22 / 80 / 5050 
* Nous dispons aussi d'un clé privé permettant de se connecter aux deux instances ec2 ci-dessus:
    * /home/ubuntu/.ssh/capge_projet_kp.pem


* Pour lancer les playbooks, nous devons surcharger les variables d'environnement suivantes
    * **ansible_connection** : spécifier qu'il s'agit d'une connexion ssh
    * **ansible_host** : spécifier l'adresse ip de l'instance ec2 hote sur laquelle on execute le playbook
    * **host_db** : l'adresse ip de la machine sur laquelle se trouve la base de donnée postgres
    * **odoo_url** : l'adresse dns publique ou ip publique de l'instance ec2 odoo 
    * **pgadmin_url** : l'adresse dns publique ou ip publique de l'instance ec2 server
    * **ic_webapp_image** : image docker de ic-webapp sur docker-hub
    * **ic_webapp_port** *(optionnel)* : port exposé (externe) de l'application ic-webapp
    * **postgres_image** : image docker de postgres sur docker-hub 
    * **odoo_image** : image docker de odoo sur docker-hub

## Commandes ansible à exécuter (sur une machine disposant ansible)
```bash
# install requirements
ansible-galaxy install -r roles/requirements.yml
```
!["Capture_Capge_01.JPG"](./assets/Capture_Capge_01.JPG)
```bash
# deploy odoo container on odoo ec2
ansible-playbook -i hosts.yml playbook_odoo.yml \
    -e ansible_connection='ssh' \
    -e ansible_host='44.201.245.76' \
    -e postgres_image='postgres:10' \
    -e odoo_image='odoo:13.0' \
    --private-key '/home/ubuntu/.ssh/capge_projet_kp.pem'
```
#### ***`console`***
!["Capture_Capge_02.JPG"](./assets/Capture_Capge_02.JPG)<br>
#### ***`site odoo`***
!["Capture_Capge_03.JPG"](./assets/Capture_Capge_03.JPG)<br>

```bash
# deploy pgamdin on server ec2
ansible-playbook -i hosts.yml playbook_pgadmin.yml \
    -e ansible_connection='ssh' \
    -e ansible_host='52.87.202.243' \
    -e host_db='44.201.245.76' \
    --private-key '/home/ubuntu/.ssh/capge_projet_kp.pem'
```
#### ***`console`***
!["Capture_Capge_05.JPG"](./assets/Capture_Capge_05.JPG)<br>
#### ***`site pgamdin`***
!["Capture_Capge_06.JPG"](./assets/Capture_Capge_06.JPG)<br>
```bash
# deploy ic-webapp on server ec2
ansible-playbook -i hosts.yml playbook_ic-webapp.yml \
    -e ansible_connection='ssh' \
    -e ansible_host='52.87.202.243' \
    -e odoo_url='http://ec2-44-201-245-76.compute-1.amazonaws.com:8069' \
    -e pgadmin_url='http://ec2-52-87-202-243.compute-1.amazonaws.com:5050' \
    -e ic_webapp_image='lianhuahayu/ic-webapp:1.0' \
    -e ic_webapp_port='80' \
    --private-key '/home/ubuntu/.ssh/capge_projet_kp.pem'
```
#### ***`console`***
!["Capture_Capge_08.JPG"](./assets/Capture_Capge_08.JPG)<br>
#### ***`site ic-webapp`***
!["Capture_Capge_09.JPG"](./assets/Capture_Capge_09.JPG)<br>

# Déploiement de l'environnement dev avec TERRAFORM
## Réflexion 
* Les instances ec2 qui sur lesquelles seront déployés les conteneurs sont de type t2.micro (imposé)
* Ansible requière au minimum une machine de type t2.medium
* Nous avons choisi de ne pas surcharger le serveur Jenkins avec Ansible afin qu'il reste uniquement dédié a son role de CI/CD.
* Nous aurons donc besoin de créer 3 instances :
    * x1 instance ec2 Master pour Ansible de type t3.medium qui lancera les playbooks
    * x2 instances ec2 Worker de type t2.micro 
        * Serveur Admin (ic-webapp et pgadmin)
        * Serveur Odoo (odoo frontend et odoo backend)

## Les modules
* sg : module permettant de mettre un place un group de sécurité aws
* ec2_master : module permettant de créer une instance t3.medium et qui lancera les commandes ansible
* ec2_worker : module permettant des instance t2.micro

## Les crédentials
Pour des raison de sécurité, le fichier ***`.aws/credential`*** est simplement un template.<br>
Dans notre JenkinsFile, nous allons remplacer donc remplacer chaine de caractères suivantes par les valeurs de nos crédentials, stockés de façon sécurisé dans Jenkins :
* YOUR_KEY_ID
* YOUR_ACCESS_KEY

## Les variables à surcharger
* Obligatoires
    * key_path : chemin ou se trouve la clé privé que va utilier ec2 MASTER pour exectuer les script en ssh
    * key_name : le nom de la clé privé du coté de AWS
    * ic-webapp_image : le nom de l'image ic-webapp sur dockerhub
    * odoo_image : nom de l'image odoo sur dockerhub
    * postgres_image : nom de l'image postgres sur dockerhub
* Optionnelles
    * odoo_port : port de l'interface web de odoo
    * pgadmin_port : port de l'interface web de pgAdmin

#### ***`Exemple`***
```bash
terraform init
terraform plan 
terraform apply -var='key_path=~/.ssh/key.pem' \
                -var='ic-webapp_image=lianhuahayu/ic-webapp:1.0' \
                -var='odoo_image=odoo:13.0' \
                -var='postgres_image=postgres:10'
```