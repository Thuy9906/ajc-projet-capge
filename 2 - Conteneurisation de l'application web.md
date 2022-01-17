## 2 - Conteneurisation de l'application web

Il s’agit en effet d’une application web python utilisant le module Flask. L'image de base que nnous vous recommandons afin de conteneuriser cette application est : `python:3.6-alpine`

Une fois le Dockerfile crée, Buildez le et lancer un container test permettant d’aller sur les sites web officiels de chacune de ces applications (site web officiels fournis ci-dessus).

**Nom de l’image :** ic-webapp ;*  
**tag :** 1.0*  
**container test_name :** test-ic-webapp*

Une fois le test terminé, supprimez ce container test, testez les vulnérabilités de votre image à l'aide de l'outil `snyk`. Une fois terminé, poussez votre image sur votre registre Docker hub.

### Repertoire de notre application
Récupération du projet puis se rendre dans le répertoire du projet
```shell
sudo apt update -y
sudo apt install git -y

git clone https://github.com/sadofrazer/devops-project1.git
cd devops-project1
```

### Installation de docker
Installation de docker, si l'on veut build et tester notre image il faudra docker.
```shell
#Installation de docker via le script fourni par docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

#Ajout du user ubuntu dans le groupe docker
sudo usermod -aG docker ubuntu

#Demarrer docker et configurer son demarrage auto à chaque reboot
sudo systemctl start docker
sudo systemctl enable docker
```


### Redaction du Dockerfile
Edition du Dockerfile :
```dockerfile
FROM python:3.6-alpine

LABEL maintainer=CAPGEMINI

# Configuration du répertoire de travail
WORKDIR /opt/ic-webapp/

# Copier les fichiers de notre projet dans notre répertoire de travail
COPY . /opt/ic-webapp/

# Set variables env ODOO_URL and PGADMIN_URL (Les variables d'environnement sont spécifié dans l'app.py)
ENV ODOO_URL='https://www.odoo.com/'
ENV PGADMIN_URL='https://www.pgadmin.org/'

# Test variables env ODOO_URL and PGADMIN_URL
# Pendant le build de l'image : 
# Vérifiez que ces variables ont bien pris les valeurs précédentes 
RUN echo ${ODOO_URL}
RUN echo ${PGADMIN_URL}

# Installation Flask avec pip car l'application utilise flask
RUN pip install flask

# Exposition du port 8080 pour l'API (Le port est spécifié dans l'app.py Run Flask Application)
EXPOSE 8080

# Start le serveur et lancer l'application
ENTRYPOINT [ "python", "app.py" ]
```

### Build de notre image et lancement du container test
Commandes de build et de lancement
```shell
# Commande pour build notre image
docker build -t ic-webapp:1.0 .

# Commande pour lancer notre container
docker run -d --name test-ic-webapp -p 80:8080 ic-webapp:1.0
```

La commande docker build le -t ou --tag permet de nommer et ajouter un tag à notre image que l'on va build, le . à la fin de notre commande est obligatoire pour dire que l'on s'appuie sur un Dockerfile se trouvant dans le répertoire courant.

Pour la commande docker run l'option -d ou detach sert à lancer le container en background pour libérer le terminal, le --name sert à donner un nom à notre container, -p pour exposé un port pour que l'on puisse se connecter à l'API, on fait une redirection du 80 externes (pour de l'hote) vers le 8080 internes (port du container), étant donné que c'est une application web, c'est mieux d'utiliser le port http tcp/80. (NB : Vérifiez que vous n'avez pas un service web qui tourne sur votre machine et qui soient déjà binder au port 80 de host). Ensuite à la fin de la commande on ajoute le nom et le tag de notre image builder précédemment.


Pour valider que notre image déployer dans le container `test-ic-webapp` fonctionne bien, on peut tout simplement ouvrir notre navigateur et cliquer sur les différentes applications qui nous redirigeront normalement sur les sites officiels de odoo et de pgadmin.

### Vérification du bon fonctionnement de notre image
Sur la capture ci dessous on constate bien que ces deux images ont bien pris la référence du site officiel.
![[Pasted image 20220112151043.png]]

On pourra surcharger ces variables d'environnement au lancement du container avec l'option '-e' 

### Nettoyage des containers
On peut maintenant après avoir tester notre image dans un container et vérifier que cela fonctionne bien la supprimer comme suit :

```shell
# Il faut stoper les containers avant de les supprimer
docker container stop test-ic-webapp

# Supprimer notre container qui est à l'arret
docker container rm test-ic-webapp

# Une suppression peut s'être mal déroulé et donc quand on voudra rester notre image on ne pourra pas indiquant que l'image est déjà utilisé dans un container.
# Je vous suggère de lister tous les containers
docker container ls -aq

# Voir faire un nettoyage forcé de ce qu'il y a via
docker container stop $(docker container ls -aq)
docker container rm $(docker container ls -aq)
```

Pour nettoyer nos environnements de tests des containers créés plutard, ce script rapide supprime les container proprement, puis supprime les volumes et supprime le répertoire où est sauvegarder les configurations persistantes ! Nous avons apellé ce script clean.sh toujours utile.
Pour nettoyer l'environnement de test local :
```shell
#!/bin/bash
docker container stop $(docker container ls -aq)
docker container rm $(docker container ls -aq)
docker volume prune --force
docker network prune --force
sudo rm -Rf /backup/*
user@worker:~$ sudo sh clean.sh
```


### Test des vulnérabilités de notre images
Ensuite on récupère snyk pour tester les vulnérabilités de notre image, pour information Docker et Snyk ont un partenaria ce qui fait qu'à l'installation de docker, ce dernier possède l'outil docker scan qui permet de scanner des containers en s'appuyant sur une api Snyk.

```shell
docker scan --version

docker scan --login --token <mon token snyk>
docker scan lianhuahayu/ic-webapp:1.0 --file Dockerfile
```
On peut ajouter beaucoup plus de détail de paramètre pour que nos scans soient efficients. Nous verrons par la suite que dans le pipeline nous pouvons passer par un docker scan ou directement avec une cli snyk que l'on va récupérer avec nodejs sur notre node Master JENKINS en /bin/bash via la commande d'installation `npm install snyk@latest -g`


### Push image sur le registry
Il faut d'abord tag notre image avec le nom de notre repository

```shell
#Listez avec nos images et récupérer l'IMAGE ID de notre image
docker image ls

#Une fois que l'on a notre image ID on peut tag cette dernière avec notre repo sur dockerhub
docker tag 66682c37eec4 lianhuahayu/ic-webapp:1.0


# Ensuite se connecter à dockerhub avec notre compte, cette étape est nécessaire pour push une image
docker login --username lianhuahayu

# Puis push notre image du le registry
docker push lianhuahayu/ic-webapp:1.0

# Une fois que c'est push on peut se logout
docker logout
```

Pour la partie sécurité, on peut pousser notre application sur un registry privé, mais il faudra fournir des accès pour pouvoir push et pull. Alors qu'un registry public ne nécessite pas qu'on soit authentifié pour pull.

Maintenant nous pouvons récupérer notre image pour la déployer où l'on veut :)
```shell
docker pull lianhuahayu/ic-webapp:1.0
```


### Installation de Snyk Security sur Jenkins 
### 3. Configure a Snyk API Token Credentials

J'ai tenté cette partie sans succès, le SNYK utilisé par le plugin sur Jenkins est un snyk test simplement, le snyk container de docker scan est tellement mieux.
-   [Get your Snyk API Token](https://support.snyk.io/hc/en-us/articles/360004037537-Authentication-for-third-party-tools)
-
-   Allez sur "Administrer Jenkins" > "Manage Credentials"
-   Choisir un magasin
-   Choisir un domaine
-   Go to "Add Credentials"
-   Choisir  "Snyk API Token", ce n'est pas un secret text, par contre si vous comptez utiliser le snyk récupérer via un npm sur votre système jenkins ou un node, préféré un "text secret" comme type pour le Token
-   Configurer les credentials
-   Se rapeller de l'ID qu'on a rentré car il sera nécessaire

Cette piste a été explorer mais sans succès. 
