# kurbernetes-ingress

# Kubernetes Ingress

## Installer Kind et créer votre premier cluster
https://kind.sigs.k8s.io/docs/user/quick-start/

### Installation de Kind
```bash=
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
$ chmod +x ./kind
$ sudo mv ./kind /usr/local/bin/kind
```

### Création de cluster
```bash=
$ kind create cluster

Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

## Installer le Nginx Ingress Controller
https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx

### Création du cluster avec les variables extraPortMappings et node-labels

```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```


### Création du Nginx ingress Controller

#### Paramétrer le ingress

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

```
$ kubectl get pods --namespace=ingress-nginx
NAME                                       READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-mzbzd       0/1     Completed   0          72m
ingress-nginx-admission-patch-xnpqr        0/1     Completed   1          72m
ingress-nginx-controller-6bccc5966-pt4lz   1/1     Running     0          72m

```

## Schéma avec des objets Kubernetes
![](https://i.imgur.com/xN8x3Px.png)


## Builder et publier (à partir de l’image nginx) sur le DockerHub, une image docker pour chacun des sites web présent sur le schéma précédent. Vous devez avoir 3 images (une par magasin tacos, pizzas et burgers)


* Créer un Dockerfile et une fichier html pour chacun des sites web

**Exemple de fichier html pour les pizzas** :
```
Voici la liste des pizzas :
- toto
- titi
```

**Exemple de Dockerfile pour les pizzas** : 
```bash=
FROM nginx:latest
COPY ./index.html /usr/share/nginx/html/index.html
CMD ["nginx", "-g", "daemon off;"]
```

* Construire les images

```bash=
docker build -t pizza -f Dockerfile .
docker build -t bt-burger -f Dockerfile .
docker build -t bt-tacos -f Dockerfile .
```

```bash=
$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED          SIZE
pizza                         latest    a097fbb96dbf   4 seconds ago    142MB
bt-burger                     latest    a359951be361   25 seconds ago   142MB
bt-tacos                      latest    926c8e30b9e1   46 seconds ago   142MB
```

* Faire le tag pour chacune des images

```bash=
$ docker tag pizza nicoss0013/restaurants:pizza
$ docker tag bt-burger nicoss0013/restaurants:bt-burger
$ docker tag bt-tacos nicoss0013/restaurants:bt-tacos
```

* Se connecter a Docker Hub

```bash=
 docker login --username=votreusername
```

* Pousser les images sur Docker Hub

```bash=
docker push nicoss0013/restaurants:pizza
docker push nicoss0013/restaurants:bt-tacos
docker push nicoss0013/restaurants:bt-burger
```

* Vérifier la présence des images sur Docker Hub

![](https://i.imgur.com/3F6Vl6u.png)


# Ecrire les fichiers yaml vous permettant de déployer sur votre cluster kind installé en local les composants décrits sur le schéma de la question 3 et les images crées à la question 4

Arborescence des fichiers : 

```bash=
├── bt-burger
│   ├── Dockerfile
│   ├── conf-burger.yml
│   └── index.html
├── bt-tacos
│   ├── Dockerfile
│   ├── conf-tacos.yml
│   └── index.html
├── ingress.yml
└── pizza
    ├── Dockerfile
    ├── conf-pizza.yml
    └── index.html
```

**Fichier ingress.yml à la racine du dossier**

```bash=
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend-restaurants
  annotations: 
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: mypizza.eatsout.com
    http:
      paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: nginx-pizza-web-service
              port:
                number: 80
  - host: burgerandtacos.eatsout.com
    http:
      paths:
        - pathType: Prefix
          path: "/burgers"
          backend:
            service:
              name: nginx-burger-web-service
              port:
                number: 80
        - pathType: Prefix
          path: "/tacos"
          backend:
            service:
              name: nginx-tacos-web-service
              port:
                number: 80
```

**Fichier conf-pizza.yml** 

```bash=
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-pizza-web-service
spec:
  selector:
    app: nginx-web-pizza
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pizza-web-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-web-pizza
  template:
    metadata:
      labels:
        app: nginx-web-pizza
    spec:
      containers:
      - name: cont-pizza
        image: nicoss0013/restaurants:pizza
        ports:
        - containerPort: 80
```

**Fichier conf-burger.yml**

```bash=
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-burger-web-service
spec:
  selector:
    app: nginx-web-burger
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-burger-web-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-web-burger
  template:
    metadata:
      labels:
        app: nginx-web-burger
    spec:
      containers:
      - name: cont-burger
        image: nicoss0013/restaurants:bt-burger
        ports:
        - containerPort: 80
```

**Fichier conf-tacos.yml**

```
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-tacos-web-service
spec:
  selector:
    app: nginx-web-tacos
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-tacos-web-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-web-tacos
  template:
    metadata:
      labels:
        app: nginx-web-tacos
    spec:
      containers:
      - name: cont-tacos
        image: nicoss0013/restaurants:bt-tacos
        ports:
        - containerPort: 80
```

**Etat des déploiement :**
```
$ kubectl get pods
127.0.0.1 mypizza.eatsout.com
 get deployments
kubectl get ingress
127.0.0.1 mypizza.eatsout.com
kubectl get servicesNAME                                           READY   STATUS    RESTARTS   AGE
nginx-burger-web-deployment-67f8765ff9-57jt4   1/1     Running   0          6m41s
nginx-pizza-web-deployment-865547cc-f859z      1/1     Running   0          6m32s
nginx-tacos-web-deployment-8db6c7f74-nb7wm     1/1     Running   0          8m22s

$ kubectl get deployments
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
nginx-burger-web-deployment   1/1     1            1           6m42s
nginx-pizza-web-deployment    1/1     1            1           8m40s
nginx-tacos-web-deployment    1/1     1            1           8m23s

$ kubectl get ingress
NAME                                   CLASS    HOSTS                                            ADDRESS     PORTS   AGE
ingress-resource-backend-restaurants   <none>   mypizza.eatsout.com,burgerandtacos.eatsout.com   localhost   80      8m58s**
```

**Modifier le /etc/host sur wsl**

```
127.0.0.1 mypizza.eatsout.com
127.0.0.1 burgerandtacos.eatsout.com
```

**Modifier le fichier /etc/host sur Windows**

```
C:\Windows\System32\drivers\etc\hosts
```

![](https://i.imgur.com/dEzozys.png)
![](https://i.imgur.com/KrBIvzg.png)
![](https://i.imgur.com/BawCQq3.png)

# Votre magasin de tacos devient très populaire (il va avoir 3 fois plus de commandes). Il va vous falloir gérer une charge importante sur le Service de commande des tacos. Comment gérez-vous cela ? Comment vérifier que la charge est bien répartie (avec quelle commande kubectl ?) ?


* Modifier le replica du restaurant de tacos à 3 et vérifier la commande kubectl get rs

```
$ kubectl get rs
NAME                                     DESIRED   CURRENT   READY   AGE
nginx-burger-web-deployment-67f8765ff9   1         1         1       28m
nginx-pizza-web-deployment-69945d58c7    0         0         0       30m
nginx-pizza-web-deployment-865547cc      1         1         1       28m
nginx-tacos-web-deployment-8db6c7f74     3         3         3       30m
```

Il est possible de vérifier la répartition de charge en faisant un **kubectl top** . Cependant il faut que le serveur soit connecté à une API pour y voir les détails.


# Question bonus : Créer une nouvelle version de votre carte des pizzas et publiez-la dans une nouvelle version de votre image. Appliquer la modification à votre déploiement. Qu’observez vous sur la disponibilité du service qui présente la carte des pizzas pendant la mise à jour ? 
	
Pour mieux visualiser celà vous pouvez en parallèle de la mise à jour exécuter les commandes suivantes dans d’autres terminaux : 
	
watch -n 1 -c kubectl get pods	
watch -n 1 -c curl mypizza.eatsout.com

* Modification du fichier index.html des pizzas 

```
Voici la liste des pizzas :
- toto
- titi
- 4 fromages
- 6 fromages
- Napolitaine

Ceci est la carte v2
```
* Construire la nouvelle image
```
docker build -t pizza -f Dockerfile .
```

* Mettre un tag avec la nouvelle version
```
docker tag pizza:v2 nicoss0013/restaurants:pizzav2
```

* Utiliser la nouvelle image sur le déploiement actuel

```
kubectl set image deployment/nginx-pizza-web-deployment cont-pizza=nicoss0013/pizzav2
```

On remarque qu'en mettant dans la configuration du déploiement un replica à 2, on assure une continuité de service lors de la mise en place de la nouvelle image. 

![](https://i.imgur.com/NWRjApw.png)
