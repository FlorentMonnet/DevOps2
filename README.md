# Day 1: Discover Kubernetes

## Etape 1 : Premières ressources : Pods/Replicaset/Deployment

**Quelles sont les informations que l'on retrouve dans le fichier kubeconfig ?** 

Dans ce fichier, on retrouve des informations concernant:
- les clusters:
    - certificat 
    - nom du serveur
- les contextes:
    - nom
    - namespace
    - user 

On retrouve aussi d'autres informations comme le token de l'utilisateur et ses préférences.


**Quelle est la différence ?**

```shell
#premiere commande
kubectl get pods -n florent-monnet

#deuxième commande
kubectl get pods -n default
```

- premiere commande:
    - On obtient tous les pods qui sont déployés sur mon namespace (nom du pod, l'état et le nombre de restart, la durée depuis le dernier déploiement)

     - On ne peut pas lister les pods car on a pas les droits sur ce namespace


**Quelles sont les propriétés principales que l'on retrouve dans le yaml du nginx?**

```shell
kubectl get pods mynginx -o yaml
```
On retrouve des informations concernant le container (image, tag, ressources, volumes attachés), on a aussi l'ip de l'host, l'ip du pod.

**Que remarquez-vous dans la description des properties spec: template ? À quoi sert le selector: matchLabels ?**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: unicorn-front-replicaset
  labels:
    app: unicorn-front
spec:
  template:
    metadata:
      name: unicorn-front-pod
      labels:
        app: unicorn-front
    spec:
      containers:
      - name: unicorn-front
        image: public.ecr.aws/l3x6e3t5/takima-training/nginx
  replicas: 3
  selector:
    matchLabels:
      app: unicorn-front

```

La propriété matchLabels permet de match le pod/replicaSet avec une d'autres ressources kubernetes (un service, un deploiment, etc.).

Dans les propriétés template de spec, on remarque qu'il y a des informations concernant les métadata ainsi que le conteneur utilisé pour le déploiement des pods. 

**Combien y a-t'il de pods déployés dans votre namespace ?**

Il y a 3 pods dans mon namespace (3 unicorn-front). 

**Maintenant, supprimez un Pod.**

**Que se passe-t'il ?**

Le pod se redéploie quand on le supprime.

**Supprimez le ReplicaSet.**

**Que se passe-t'il ?**

Tous les pods se suppriment quand on supprime le ReplicaSet.

**Quels sont les changements enter un Deployment et un ReplicaSet ?**

Avec les déploiements, on peut gérer les montée de version facilement en utilisant les RollingUpgrade. Il est aussi simple de gérer les rollback avec les Deployments. Cela n'est pas le cas avec les ReplicaSet.

**Combien y a-t'il de ReplicaSet ? De Pods ?**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unicorn-front-deployment
  labels:
    app: unicorn-front
spec:
  replicas: 3
  selector:
    matchLabels:
      app: unicorn-front
  template:
    metadata:
      labels:
        app: unicorn-front
    spec:
      containers:
      - name: unicorn-front
        image: public.ecr.aws/l3x6e3t5/takima-training/nginx:1.7.9
        ports:
        - containerPort: 80

```

À l'issue du déploiement, on a 3 pods et 1 ReplicaSet.



**Une fois terminé, combien y a-t-il de replicaset ? Combien y a-t-il de Pods ? Allez voir les logs des événements du déploiement avec kubectl describe deployments. Qu’observez vous ?**

```shell
kubectl rollout status deployment.v1.apps/unicorn-front-deployment
```

Une fois le RollingUpgrade terminé, on a 3 pods et 1 ReplicaSet. 

**Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable**

En regardant les logs des événements du déploiement, on voit 3 pods ont été mis à jour.

**Que se passe-t'il ? Pourquoi ?**

```shell
kubectl set image deployment.v1.apps/unicorn-front-deployment nginx=public.ecr.aws/l3x6e3t5/takima-training/nginx:1.91-falseimage --record=true
```

On remarque que l'image nginx n'est pas la même que celle de la version déployée. 
On est donc un état avec 2 Deployments, 2 ReplicaSets, 4 pods dont 1 qui est down. 
A chaque fois, qu'un pod de la nouvelle version est déployée avec succès, l'ancien pod est supprimé. Il en est de même pour le RéplicaSet et le Deployment quand tous les nouveaux pods sont déployés.

**Combien y a-t-il de révisions ? À quoi correspond le champ CHANGE-CAUSE ?**

On voit qu'il y 2 révisions. Le CHANGE_CAUSE nous fourni la commande qui a été éxécuté. 

**Combien y a-t'il de Pods?**

```shell
kubectl scale deployment.v1.apps/unicorn-front-deployment --replicas=5
```

Après cette commande, on a 5 pods.

**Que se passe-t-il au niveau ReplicaSet ?**

```shell
kubectl rollout pause deployment.v1.apps/unicorn-front-deployment
kubectl set image deployment.v1.apps/unicorn-front-deployment nginx=public.ecr.aws/l3x6e3t5/takima-training/nginx:1.20.2
```

On met en pause le déploiement, la montée de version n'est pas encore à l'issu de ces deux commandes.


**Que se passe-t-il au niveau ReplicaSet ?**

```shell
kubectl rollout resume deployment.v1.apps/unicorn-front-deployment
```
Le déploiement reprend et donc la montée de version se réalise, on a donc un ReplicaSet qui a été créé, etc.

## Etape 2 : Publication Service/Ingress

**Que se passe-t'il ? Pourquoi ?**

L'image est disponible sur un registry qui requiert une connexion, or on a pas encore mis en place de secret pour se connecter et donc pour pouvoir pull l'image présente dnas le registry. On ne peut donc pas déployer nos pods.


**Décrivez ce que répond la Web App ? Actualisez votre page avec CTRL + F5. Que se passe-t-il ?**

On a un texte qui n'est pas complet (l'ip est manquante). Lorsque l'on actualise la page, la couleur du background change.

**Que constatez-vous sur le navigateur ?**

Le texte est désormais complet, l'ip ainsi que le nom du node ont été rajouté.

# Day 2: Deep Dive

**Que se passe-t-il au niveau des Pods de l’API ? Vous pouvez jeter un oeil aux logs. (kubectl logs -f nomdupod)**

Les pods ne se déploient car l'API a besoin d'une connexion à une base en PostgresSQL pour fonctionner. Or, ce n'est pas encore le cas. Le déploiement des pods fail donc en boucle.


**Quel est le nom du service de la base de données ?**

Le nom du service est **pg-service.florent-monnet**.

**Pourquoi le computer a disparu ?**

Ma persistence de données n'est pas encore présent dans notre base postgres. Les données ne sont donc pas sauvegardées lorsque l'on détruit un pod.

**D’après le tableau, quel est le type d’accès implémenté par notre Storage Class EBS ? Pourquoi cela convient parfaitement pour la persistance de la base de données Postgres ?**

Nous on a mit en place le `ReadWriteOnce` car nous avons 1 seul replicas du pod postgres.

