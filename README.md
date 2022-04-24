# mini-projet-kubernetes
Déploiement de l'application WordPress avec persistance des données

Pour réaliser notre mini projet, il est important de connaitre que l'application WordPress est structurée en deux parties:

- Une partie front end (IHM) 
- Une partie backend (Base de données)

## Indications littérales

Pour ce faire, nous allons:
- Déployer un **pod** Mysql.
- Ce pod Mysql sera dans un objet de type **deployment**.
- Nous allons définir un seul **replicat**.
- Le pod Mysql aura en frontal un service de type **clusterIP** qui permettra d'exposer le **pod mysql** en interne dans k8s.
- Tout ceci sera couplé à un **pod WordPress**.
- Le **pod WordPress**, créé sera lui aussi dans un objet de type **deployment**.
- A ce **pod WordPress** sera apposé un service de type **nodeport**. Ce service sera le point d'entrée de l'application WordPress.
- Notre application **WordPress** viendra interroger l'application **Mysql** en passant par le service **clusterIP**.

## Persistance des données

L'application Mysql devra avoir un stockage persistant pour stocker ses données.

Ici, le volume sera fait comme dans le cas de docker. Ce volume sera mis dans le dossier **data** et dans le répertoire mysql (**/data/mysql**).

Ce qui sera stocker dans ce volume est /var/lib/mysql.

Dans le **Pod Mysql**, nous pouvons aussi associer un volume simple qui sera stocké dans **/data/wordpress**

Ce qui sera stocké à l'intérieur de ce volume sera le contenu **/var/www/html** de notre pod frontal.

Tous ces objets ainsi que toutes les ressources seront dans un **Namespace** appelé **WordPress**. 

Afin que l'appication soit consommée par une personne venant de l'extérieure, celle-ci sera obligée de passer par le **service NodePort**. NodePort expose le service depuis l’IP publique de chacun des noeuds du cluster en ouvrant port directement sur le nœud, entre **30000** et **32767**. Cela permet d’accéder aux pods internes répliqués. Comme l’IP est stable on peut faire pointer un DNS ou Loadbalancer classique dessus.

# Illustration graphique de notre solution

![Image solution](/images/illustration.png.jpeg "Image Solution Proposée")

# Réalisation du mini projet k8s

## Les outils utilisés et fichiers du projet

- Visual Studio Code.
- L'hyperviseur de type 2 Oracle VM VirtualBox.
- Cluster kubernetes (Orchestrateur).
- minikube, est un outil qui fait tourner un cluster Kubernetes à un noeud unique dans une machine virtuelle sur votre machine.
- Vagrantfile pour créer notre VM kunernetes.
- install_minikube.sh fichier de provisionning qui va automatiser l'intallation des outils nécessaire dans la VM.
- Les manifestes de deployement (WordPress et Mysql).
- Les manifestes des services (clusterip et nodeport).
- Les manifestes du namespace et du secret.
- Le docker-compose pour automatiser l'installation de Mysql.

## Mise en place de la VM Kubernetes

1. On lance la VM
```
vagrant up --provision
```

2. Je me connecte à ma VM minikube via ssh
```
vagrant ssh
```
3. Je démarre minikube
```
minikube start --driver=none
```
4. Je vérifie dans la VM le(s) pod(s), node(s) ou deployment qui pourrai(ent) exister voire même tourner.
```
kubectl get po
```

**Rappel**: Si nous avons le(s) pod(s), node(s) ou deployment qui pourrai(ent) exister voire même tourner, il faut les supprimer avant de commencer un nouveau projet.

Si ces objets sont dans un **deploy**, alors il faut juste supprimer cet objet **deploy**. 

```
kubectl delete deploy [nom-deploy]
```
Nous devons avoir le node **minikube** en statut **Ready** comme le rendu ci-dessous.

```
kubectl get node
```
Résultat:
```
NAME       STATUS   ROLES                  AGE   VERSION
minikube   Ready    control-plane,master   30s   v1.23.3
```
```
kubectl get deploy
```
Pas de **Pods** ni de **deploy**, nous passons à la phase de test.


# Analyse explicative des manifestes de notre projet k8s.

Ces manifestes se trouvent dans le dossier **stack-k8s**.
- **Vagrantfile** a permit d'installer notre *minikube*.
- **install_minikube.sh** est le script d'installation.
-  **docker-compose.yml** pour déployer l'insfrastructure dans *k8s*.
-  **mysql-deployment** est le manifeste qui va nous permettre de déployer la base de données *MySql*.
-  **app-wordpress-secret.yml** est le manifeste de l'objet *secret* qui contient tous mes credentials de wordpress et mysql.
-  **app-workpress-namespaces.yml** est le namespace de wordpress.
-  **wordpress-deployment.yml** est le manifeste qui va nous permettre de déployer l'application *wordpress*.
-  **service-clusterip-mysql.yml** est le service qu va permettre à notre *deployement mysql* d'exposer via un *ClusterIp* l'application en interne.
-  **service-nodeport-wordpress.yml** est le service qu va permettre à notre *deployement wordpress* d'exposer via un *NodePord* l'application en externe.

## Lancer le déploiement

- Je me positionne dans le repertoire où se trouve les manifestes
```
$ cd [nom_dossier] 
$ cd stack-k8s
```
- Dans le répertoire, je lance la commande ci-dessous pour créer tous les objets
```
kubectl -f ./
```
Résultat:

```
namespace/wordpress created
secret/app-wordpress-secret created
deployment.apps/wp-mysql created
service/wp-mysql created
service/wordpress created
deployment.apps/wordpress created
```
1. Je vais visualiser le namespace

```
kubectl get namespace
```
Résultat:
```
NAME              STATUS   AGE
default           Active   29m
kube-node-lease   Active   29m
kube-public       Active   29m
kube-system       Active   29m
wordpress         Active   8m23s
```

2. Je vais visualiser les podes qui sont dans le namespace wordpress

```
kubectl get po -n wordpress
```
Résultat: Mes podes sont *running* depuis 12m
```
NAME                        READY   STATUS    RESTARTS   AGE
wordpress-d546d667-w2dpd    1/1     Running   0          12m
wp-mysql-84bbbdc74f-5rnkb   1/1     Running   0          12m
```

3. Je vais visualiser que mes podes sont bien dans un deploiement
```
kubectl get deploy -n wordpress
```
Résultat:
```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
wordpress   1/1     1            1           15m
wp-mysql    1/1     1            1           15m
```

4. Je vais visualiser que mes services existent bien et sont à l'écoute.
```
kubectl get svc -n wordpress
```
Résultat:
```
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
wordpress   NodePort    10.106.36.111   <none>        80:30008/TCP   18m
wp-mysql    ClusterIP   10.101.89.37    <none>        3306/TCP       18m
```

4. Je vais visualiser que mes secrets
```
kubectl get secret -n wordpress
```
Résultat:
```
NAME                   TYPE                                  DATA   AGE
app-wordpress-secret   Opaque                                3      21m
default-token-d2sfl    kubernetes.io/service-account-token   3      21m
```

**En conclusion, tout est ok**


# TESTS

Pour faire les tests, je vais récupérer l'ip de ma VM qui a été fourni par *vagrant*.
```
ip a
```
Résultat:
```
192.168.56.39
```
Je vais taper cette *ip* dans le navigateur en *http* sur le port *30008* afin de consommer notre application *WordPress*.

```
http://192.168.56.39:30008
```
1. Image pages d'installation de l'application WordPress.

![Image App](/images/wordpress.png "Image Application WordPress")

2. Vérifions le stockage

```
cd /data/
```
```
ll
```
Résultat: Dans */data/* j'ai bien un repertoire *mysql* et *wordpress* qui ont été créés.

![Image stock](/images/stock.PNG "Image des stockages Créés")

3. Finalisons l'installation

![Image param](/images/inst.PNG "Image Parametrage")

Puis je clique sur le bouton *Install WordPress*.

![Image success](/images/succes.PNG "Image Install Success")

Puis je clique sur le bouton *Log In*.

![Image log](/images/log.PNG "Image Log")

![Image Accueil](/images/accueil.PNG "Image Accueil")

![Image Accueil1](/images/accueil1.PNG "Image Accueil1")

# Recommandations

Je recommande de le site de [cours hadrien pelissie](https://cours.hadrienpelissier.fr/03-kubernetes/) afin de reforcer vos connaissance théoriques et pratiques en **Ansible**, **Docker** et **Kubernetes**.

Je vous recommande aussi les cours de [eazytraining](https://eazytraining.fr/parcours-devops/) en général et particulièrement le **Parcours DevOps**.



# Liens

- Installer [minikube](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/)
- Installer [docker-compose](https://docs.docker.com/compose/install/).
- [Docker Hub](https://hub.docker.com/_/wordpress) pour récupérer le modèle de **docker-compose.yml** afin d'installer wordPress avec sa base de données.
- Repository git de l'application est fourni par [Dirane](https://github.com/diranetafen/student-list).
