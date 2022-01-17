## Kubernetes - applications ic-webapp
### Namespace
Il est demandé de travailler dans le namespace icgroup, il faut donc créer ce namespace avant. Puis ajouter un label env=prod pour la partie k8s

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  labels:
    env: prod
    company : icgroup
  name: icgroup 
```

Appliquer le namespace par defaut en configurant le context (a noter que namespace et context sont deux choses différentes même si ici la commande laisser penser que c'est pareille)

Pour rappel : Les contextes sont utilisés pour cibler plusieurs clusters Kubernetes différents. Et il est possible de basculer entre les clusters à l'aide de la commande kubectl config use-context très rapidement comme on peut le voir par la suite.

Les  namespaces  quant à eux sont un moyen de répartir les ressources du cluster entre plusieurs utilisateurs (via un quota de ressources). Les namespaces sont destinés à être utilisés dans des environnements avec de nombreux utilisateurs répartis sur plusieurs équipes ou projets.

```shell
#Changer de namespaces
kubectl config set-context --current --namespace=icgroup

#Vérifier son namespace par defaut
kubectl config view --minify | grep namespace
```

On peut lister toutes les ressources de ce namespace en faisant
```shell
kubectl get all -n <icgroup>
```
 
Mais on peut également vide toutes les ressources de ce namespace le supprimant puis on le recrée, le namespace sera à ce moment là vierge.

### Ingress - Service A

Le service Ingress est une ressource de kubernetes qui gère l'accès externe aux services d'un cluster, généralement du traffic http comme pour notre ic-webapp ici (bien que l'app.py est codé pour écouter sur le port 8080).

Mais avant de l'utiliser, il faut activer ingress au niveau de notre minikube avec la commande suivante :

```shell
minikube addons enable ingress
```

![[Pasted image 20220113100915.png]]

On aura a appliqué des manifestes lors du pipeline, il faut savoir que pour créer et détruire nos ressources, on peut utiliser les manifestes comme ceci :
```shell
#Pour créer les ressources
kubectl apply -f manifest.yaml

#Pour détruire les ressources créées précédemment
kubectl delete -f manifest.yaml
```


Et pour finir on peut vérifier notre environnement en regardant les ressources de notre namespace avec cette commande. 
```shell
kubectl get all -n icgroup
```
Dans nos manifestes on peut ajouter dans les metadatas de nos ressources le `namespaces: icgroup` comme ceci, nous allons avoir nos ressources dans le bon namespace.

### Déploiement de notre vitrine ic-webapp
Dans le manifeste suivant on déploit deux replicaset de notre application vitrine ic-webapp avec une stratégie de rolling update, pour qu'il y a ait le moins d'interruption de service possible.
 
On s'appuie sur un selector qui match les labels app=ic-webapp pour ce déploiement.

Le template que l'on va prendre est un template simple qui consite simplement a récupéré notre image build précédemment et pusher sur dockerhub avec les bonnes variables d'environnement et aura le label app=ic-wepapp.

A terme ces variables d'environnements prendront les URLs de nos deux autres applications le frontend d'odoo et pgadmin. Il sera simple d'accès car ce qui changera sera le node port que l'on peut fixer au niveau des services NodePorts raccrochés à nos app Odoo et PgAdmin. Grâce à l'ingress ces applications pourront être accessible depuis un FQDN et depuis l'extérieur.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-ic-webapp
  namespace: icgroup
  labels:
    env: prod
	company: icgroup
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  replicas: 2
  selector:
    matchLabels:
      app: ic-webapp
  template:
    metadata:
	  namespace: icgroup
      name: ic-webapp
      labels:
        env: prod
        app: ic-webapp
        type: pod
        company: icgroup
    spec:
      containers:
        - image: lianhuahayu/ic-webapp:1.0
          name: ic-webapp
          ports:
            - containerPort: 8080
          env:
            - name: ODOO_URL
              value: http://ec2-54-242-241-106.compute-1.amazonaws.com:32020
            - name: PGADMIN_URL
              value: http://ec2-54-242-241-106.compute-1.amazonaws.com:32125
```

Sinon on peut aussi avoir l'URL de nos service avec la commande suivante : (A noter que des clusterIP n'ont pas d'URL car n'expose pas de ports)
```shell
minikube service deploy-ic-webapp --url
```


Ensuite je crée mon clusterIP pour que l'application ne soit accessible que par les applications du cluster, je ne veux pas exposer un port ou l'adresse IP de mes pods ic-webapp* Je n'ai pas besoin d'un service ayant un port exposer car mon ingress va s'accrocher à un service pour l'exposer à l'extérieur. 
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: ic-webapp-clusterip
  namespace: icgroup
  labels:
    env: prod
    type: clusterip
    company: icgroup
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: ic-webapp
```

La vitrine est à destination du grand public, je m'arrange donc qu'elle soit tout de même accessible pour l'externe avec ce manifeste, une fois avoir activé l'ingress vu précédemment, on peut configurer notre ic-webapp pour être consultable sur notre adresse de la machine.
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ic-webapp-ingress
  namespace: icgroup
  labels:
    env: prod
    type: ingress
    company: icgroup
spec:
  rules:
  - host: <url de notre instance aws>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ic-webapp-clusterip
            port:
              number: 80
```


Problématique pour un déploiement rapide j'aimerais récupéré le FQDN de mon instance, voici la commande magique qui peut être passé sur l'instance puis l'ajouter à une varaible d'environnement pour l'utiliser dans nos manifestes sinon avec le déploiement via Terraform on peut récuéprer le public_dns directement via aws_instance.mon_instance.public_dns :

```shell
#Avec dig l'utilitaire dns
dig -x `dig +short myip.opendns.com @resolver1.opendns.com` +short | sed 's/.$//'
```


## Instant troubleshooting 
### Parlant rapidement de Odoo  

Quand SERVER INTERNAL ERROR avec Odoo, il se peut simplement que le bdd ne soit pas bien initialisé lors du déploiement, à ce moment la j'ai ajouté une commande ou j'ai variabilisé les paramètres avec le -d pour la database, le -r pour le user et le -w pour le mdp, ceci permet de forcer en quelque sorte la connexion à la bdd puis on redémarre le container pour que ça soit prendre.

il faut ouvrir un bash sur le frontend et lancer la commande suivante avec les bon parametre

```shell
odoo -i base -d odoo --stop-after-init --db_host=backend_odoo -r odoo -w odoo
```

Cette commande a été passé dans le template docker-compose pour déployer le serveur de test Odoo par la suite via Ansible :).


![[Pasted image 20220114110209.png]]


