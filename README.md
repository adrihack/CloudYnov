# TP Cloud

## Prérequis

Mise en place de l'environnement de travail grâce à un Vagrantfile que l'on va retoucher.


Il faut récupérer [ce vagrantfile](https://github.com/coreos/coreos-vagrant). C'est celui qui nous servira de base pour travailler.

Pour créer 5 VM, on va modifier la variable **$num_instances = 1** et la changer par **$num_instances = 5** à partir de là on a précisé que Vagrant devait nous sortir 5 VM.

Ajouter ensuite ceci
dans le **config.vm.provider** : 

```
datadisk1 = "./DATA#{i}.vdi"
unless File.exist?(datadisk1)
  vb.customize ['createhd', '--filename', datadisk1, '--size', 10 * 1024]
end
  vb.customize ['storageattach', :id, '--storagectl', 'SATA Control', '--port', 0, '--device', 0, '--type', 'hdd', '--medium', datadisk1]
```
Ici, on précise que l'on va configurer un disque de 10 Go pour chaque VM et leur donner un nom propre à chacun à savoir DATA1, DATA2 ...

On va ensuite donner une ip dans le même réseau à chaque VM pour qu'elles puissent communiquer ensemble.
Pour ce faire, on ajoute les lignes suivantes au Vagrantfile : 

```
 ip = "10.33.20.1#{i}"
      config.vm.network :private_network, ip: ip
      # This tells Ignition what the IP for eth1 (the host-only adapter) should be
      config.ignition.ip = ip
	  
```
Ainsi on aura 5 VM qui auront pour ip de 10.33.20.11 à 10.33.20.15
Pour se connecter en ssh à une des VM, on peut exécuter la commande **vagrant status** qui va nous afficher le résultat suivant : 
```
Current machine states:

core-01                   running (virtualbox)
core-02                   running (virtualbox)
core-03                   running (virtualbox)
core-04                   running (virtualbox)
core-05                   running (virtualbox)
```
Puis faire simplement **Vagrant ssh "NomDeVM"**.

# Docker Swarm
*Sur CoreOS, il y a un "mode détaché" qui empêche initd de surveiller le processus propriétaire du conteneur car il le place en arrière-plan. Il faut donc enlever le -D lorsqu'on lance un conteneur en daemon.*
Pour rappel, [Docker Swarm](https://docs.docker.com/engine/swarm/#feature-highlights) permet de mettre en place de la haute disponibilité au niveau du lancement et de l'entretien des conteneurs. Nous allons mettre en place **un swarm de 5 noeuds** : 3 managers et 2 workers. Le cluster restera sain tant que 2 managers seront en vie.

## Mise en place

Nous allons récuperer les métriques grâce à la clause **metrics-addr** et configurer sa valeur sur le port **9323**. De ce fait on pourra récupérer des données afin d'établir les graphiques pour la supervision.
Il faut créer s'il ne l'est pas le fichier /etc/docker/daemon.json et y ajouter les lignes suivantes
```
{
  "metrics-addr" : "127.0.0.1:9323",
  "experimental" : true
}
```
Pour vérifier, il faut éxecuter la commande **dockerd** et on vérifie qu'il n'y ait aucune erreur.


* **un swarm avec 3 managers et 2 workers.**
Configuration du premier worker qui sera la machine core-01 ici.
```
core-01 core # docker swarm init --advertise-addr 10.33.20.11
Swarm initialized: current node (enf4cfup1yx58ub363xrxygay) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-68eezhpit42omq95jprs4ep71pz944vualwrc7qlv0x8lnq3ep-7le95fmgbw281hedtd0239nu2 10.33.20.11:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
Et on fait de même sur toutes les VM.

Il faut executer ensuite ces commande sur les 2 managers qui signifient que l'on va les ajouter en tant que "manager". Il faudra ensuite faire la même chause pour les 3 autres postes dits "workers" avec l'argument worker au lieu de l'argument manager.
```
core-01 core # docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-68eezhpit42omq95jprs4ep71pz944vualwrc7qlv0x8lnq3ep-ewwz7jrfe98zmkxz5qrfl0i2u 10.33.20.11:2377
```
```
core-02 / # docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2ugdlcacke1zsc3pjwxygo6k1bankcfvxwad369gzf9j54279u-efbqmlirj2486evk3kkb9g78r 10.33.20.12:2377
```
# Dumb Service - Part 1
Tous nos services, nos applicatifs, tourneront sous la forme de stacks ou services Swarm
On va utiliser le le fichier [`compose.yml`](./app) fourni qui permet d'avoir une stack de test, il va pop l'archi suivante : `BDD <--- AppPython <--- NGINX`
Ici, j'ai créé à la racine un dossier qui s'appelle composition dans lequel on mettre le fichier docker-compose.yml
Il faut ensuite lancer cette commande sur notre premier worker pour la déployer.
```
core@core-01 /composition $ docker stack deploy python_dirty_app -c docker-compose.yml
Ignoring unsupported options: build

Creating network python_dirty_app_main
Creating service python_dirty_app_python-app
Creating service python_dirty_app_database
Creating service python_dirty_app_app-proxy
```

## Commandes de base

* `docker node ls`

* `docker stack ls`
* `docker stack deploy MA_STACK -c docker-compose.yml`
  * déploie une stack depuis un fichier compose
* `docker stack services MA_STACK`
  * liste les services d'une stack
* `docker stack ps MA_STACK`
  * liste les conteneurs d'une stack et leur état

* `docker service ls`
* `docker service logs SERVICE`
  * affiche les logs d'un service (basé sur `docker logs`)

# Dumb Service - Part 1

Tout au long du TP, nous déploierons ce service, mais en changeant notre process de déploiement à chaque fois.  

. On crée un `docker-compose.yml` pour chacune des stacks. On trouvera aussi un répertoire de donnée pour chacune d'entre elles.


* les images utilisées dans le `docker-compose.yml` doivent être accessibles sur tous les noeuds.

```
docker stack deploy python_dirty_app -c docker-compose.yml
```

* une fois démarrée l'appli devrait être joignable sur le port 8080 de votre cluster

# Weave Cloud

[Weave Cloud](https://cloud.weave.works/signup) permet d'accéder à des fonctionnalités de monitoring et métrologie directement depuis une interface dans le cloud (du SaaS donc :)). 

* utiliser [Weave Cloud](https://www.weave.works/docs/cloud/latest/install/docker-swarm/) pour monitorer votre déploiement Swarm
  * il vous faudra un compte Weave
  * inscrivez-vous (n'hésitez pas à utiliser un mail jetable :) )
  * demandez une connexion à une instance de Docker Swarm pour récupérer votre token
  * utilisez un conteneur Docker Weave([toujours la même page](https://www.weave.works/docs/cloud/latest/install/docker-swarm/)) avec votre token
    * celui-ci va déployer  une stack sur votre swarm et s'auto-détruire
  * le conteneur Weave va utiliser votre swarm pour lancer des services.
  * **Q1 : comment ce conteneur a-t-il pu lancer des conteneurs sur votre hôte ?**  

Une fois lancé, RDV sur l'interface graphique de Weave pour voir la magie. Explorez un peu y'a une tonne d'infos. 

# A. CEPH ooooouuuuu...

## Présentation

[CEPH](https://ceph.com/) est un outil permettant de mettre en place (notamment) un système de fichiers distribué ; plutôt que d'avoir une partition locale utilisé sur un filesystem local, nous allons mettre en commun des partitions (en réalité, des blocs) à travers le réseau, et rendre le tout accessible sur tous nos hôtes comme une simple partition.  

**CEPH est un outil complexe**, il se décompose en plusieurs entités que nous installerons séparément : 
* *monitors* : sont en charge de maintenir les cartes représentant le cluster (crucial pour que les démons CEPH soit synchro)
  * 3 suffiront pour notre petit lab
* *managers* : récupère et expose des métriques, ainsi que l'état du cluster
  * 3 managers aussi, sur les mêmes noeuds que les monitors
* *OSD* : le coeur de l'outil, les OSDs sont en charge de stocker de la donnée (et d'autres opération avancées comme la réplication, la restauration etc.)
  * sur TOUS les noeuds où du stockage est dispo, sur tous nos noeuds donc
* *MDS* : ce sont les serveurs de metadata, indispensable pour utiliser de façon simple le filesystem de type `ceph`
  * sur tous les noeuds

Le fonctionnement en détail de CEPH dépasse le cadre du cours, nous allons donc mettre en place un setup simple.  

Vuuuu qu'on est des guerriers **on va le mettre en place à l'aide de Docker** :)

## Mise en place

**Suivez bien les étapes, c'est crucial.**

Une fois lancés, vous aurez accès à la commande `ceph` à l'intérieur des conteneurs. Vous pouvez vous en servir pour vérifier l'état du cluster au long de l'install, par exemple : 
* `ceph status` affiche l'état du cluster
* `ceph df` affiche l'état d'occupation des pools de stockage CEPH

### 1. Monitors

* Sur votre premier host

```
docker run -d --net=host \
--restart always \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=<IP_HOST_ONLY>  \
-e CEPH_PUBLIC_NETWORK=<HOST_ONLY_CIDR_NETWORK> \
--name="ceph-mon" \
ceph/daemon mon
```
* `MON_IP` c'est pour "*monitor* IP"
* suite à ça, déplacer tout le contenu de `/etc/ceph/` sur **tous les autres noeuds**
* exécuter de nouveau la commande `docker run` ci-dessus sur deux autres noeuds (ce sera vos 3 *monitors*)
* n'oubliez pas de changer la variable `MON_IP` sur chacun de vos 3 *monitors*
* check :
```
docker exec ceph-mon ceph status
  cluster:
    id:     faba0138-849f-491d-8ba3-3ce1b92cff19
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum core-01,core-02,core-03
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs: 
```

### 2. Managers

Sur les 3 mêmes hôtes que les *monitors* :

```
docker run -d --net=host \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
--name="ceph-mgr" \
--restart=always \
ceph/daemon mgr
```
* check
```
docker exec ceph-mon ceph status
  cluster:
    id:     faba0138-849f-491d-8ba3-3ce1b92cff19
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum core-01,core-02,core-03
    mgr: core-01(active), standbys: core-03, core-02
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
```

### 3. OSDs

Rentrez dans un conteneur *monitor* en utilisant un `bash` (commande `docker exec`).  

Récupérez l'output de la commande `ceph auth get client.bootstrap-osd` et stockez le dans un fichier.  

Sur tous les noeuds : 
* copier le contenu du fichier précédemment créé dans `/var/lib/ceph/bootstrap-osd/ceph.keyring`
  * effacez s'il existe déjà, créez s'il n'existe pas
* puis :
```
docker run -d --net=host \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-v /dev/:/dev/ \
-e OSD_FORCE_ZAP=1 \
-e OSD_DEVICE=<CHEMIN_VERS_NEW_DISK_IN_/dev> \
-e OSD_TYPE=disk \
--name="ceph-osd" \
--restart=always \
ceph/daemon osd_ceph_disk
```
* n'oubliez pas de changer `CHEMIN_VERS_NEW_DISK_IN_/dev` par `/dev/sdb` par exemple


* check cluster status
```
docker exec ceph-mon ceph status
  cluster:
    id:     9f1ed298-688b-4143-bc53-5b869519ba73
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum core-01,core-02,core-03
    mgr: core-02(active), standbys: core-03, core-01
    osd: 5 osds: 5 up, 5 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   10 GiB used, 37 GiB / 47 GiB avail
    pgs:
```

### 4. MDS

* Sur tous les noeuds : 
```
docker run -d --net=host --name ceph-mds --restart always -v /var/lib/ceph/:/var/lib/ceph/ -v /etc/ceph:/etc/ceph -e CEPHFS_CREATE=1 -e CEPHFS_DATA_POOL_PG=128 -e CEPHFS_METADATA_POOL_PG=256 ceph/daemon mds
```

* Sur un noeud, pour éviter de péter vos machines :

```
ceph osd pool set cephfs_data size 2
ceph osd pool set cephfs_metadata size 2
ceph osd set nodeep-scrub
```

### 5. Config finale

* Dans un des conteneurs *monitor*, on va créer un secret qui nous permettra d'accéder au volume CEPH sur tous nos noeuds :
```
ceph auth get-or-create client.dockerswarm osd 'allow rw' mon 'allow r' mds 'allow' > keyring.dockerswarm
cat keyring.dockerswarm
```

* Puis sur tous les noeuds :
```
mkdir /data
echo "<MONITOR_IPS_COMMA_SEPARATED>:6789:/      /data/      ceph      name=dockerswarm,secret=<YOUR_SECRET_HERE>,noatime,_netdev 0 2" > /etc/fstab
mount -a`
```

**Le répertoire `/data` devrait être accessible sur les noeuds.**  

**Conseil : utilisez le pour stocker vos configurations par la suite (et applications)**

### 6. Un peu de réflexion

* **Q2 : expliquer rapidement le principe d'un système de fichiers distribué** (distributed filesystem)  

* **Q3 : proposer une façon d'automatiser le déploiement cette conf CEPH** (Vagrant ? Swarm stack ? autres ?) Par ex, si on veut rajouter un noeud ?

* QB1 : nous allons utiliser ce répertoire data pour stocker des données sur le FS des noeuds Docker (sur les hôtes). Serait-il possible que nos conteneurs utilisent directement les volumes CEPH, sans passer par un volume de l'hôte ? Illustration
  * actuel : CEPH --*MDS*--> Host --`run -v`--> conteneur
  * demandé : CEPH --*MDS*--`run -v`--> conteneur

# B..... NFS
Ou un simple partage NFS. Je ne donnerai pas d'instructions pour cette partie (très simple normalement). Pour les non-initiés, NFS (sobrement Network FileSystem) et un... système de fichiers sur le réseau :|. Il existe plusieurs articles sur internet qui expliquent comment le mettre en place, c'est plus easy qu'un CEPH.  

Attention en revanche : NFS peut mal supporter les accès concurrents, surtout en écriture (vous pouvez monter votre partition NFS uniquement en lecture).

* **Q2 : expliquez le principe d'un partage NFS, quels pourraient être ses limites dans le cas d'un swarm comme le nôtre (qui peut être amené à grandir) ?**

* **Q3 : proposez une façon d'automatiser le déploiement cette conf NFS**

# Registry - Part 1

Pour lancer un service à travers le swarm, tous les noeuds doivent pouvoir pull le conteneur. Plusieurs choix alors :
* se log sur tous les noeuds et récupérer l'image souhaitée, partout (`docker pull`, ou archive, ou autres)
* publier l'image sur un registre public/joignable sur internet
* **monter un registre soi-même. On part sur cette option.**  

Le [Docker Registry](https://docs.docker.com/registry/) (ou simplement *registre*) doit être joignable sur tous les noeuds du Swarm. 

Pour rappel le Docker Registry permet d'héberger et distribuer des images Docker.  

On va en lancer un sur notre swarm, qui pourra être joignable en local (`127.0.0.1:5000`) sur tous les hôtes : 

```
docker service create --name registry --publish published=5000,target=5000 registry:2
```
Normalement, un `curl 127.0.0.1/v2/`  devrait fonctionner et retourner `{}` sur tous les hôtes.  

Expliquez :
* **Q4 : où est lancé le service réellement ? (sur quel hôte, et comment on fait pour savoir ?)** Combien y'a-t-il de conteneur(s) lancé(s) ? 
* **Q5 : pourquoi le service est accessible depuis tous les hôtes ?** Documentez vous sur internet.

# Dumb Service - Part 2

Faites tourner le Dumb Service mais :
* Hébergez les images du Dumb Service dans le registre
* Utilisez le répertoire `/data` pour stocker ses données (`docker-compose.yml`, configurations, applications, données)
* Observer son évolution sur Weave Cloud

```
cd /data/python_app/
docker-compose build
docker-compose push
docker stack deploy python_dirty_app -c docker-compose.yml
```

# Keepalived

Actuellement, un service est joignable sur toutes les IPs des membres du Swarm. Ok, mais une IP d'entrée unique ça serait pas mal non ?  

On va mettre en place une **IP Virtuelle**, que porteront tous nos serveurs, à l'aide de [Keepalived](http://www.keepalived.org/). Vu qu'on est des hipsters : conteneurs !

En admettant la config suivante : 
* IP Virtuelle voulue sur `172.17.8.100`
* IPs des 5 hôtes : `172.17.8.101`, `172.17.8.102`, `172.17.8.103`, `172.17.8.104`, `172.17.8.105`
* on obtient la commande suivante pour lancer Keepalived sur un noeud :

```
docker run -d --name keepalived --restart=always \
  --cap-add=NET_ADMIN --net=host \
  -e KEEPALIVED_VIRTUAL_IPS=172.17.8.100 \
  -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.17.8.101', '172.17.8.102', '172.17.8.103', '172.17.8.104', '172.17.8.105']" \
  -e KEEPALIVED_PRIORITY=200 \
  osixia/keepalived:1.3.5
```

* Lancez la commande sur tous vos hôtes, en modifiant la priorité à chaque fois
  * **une fois fait, vous n'accéderez à votre cluster plus qu'avec l'ip virtuelle**

* Expliquez :
  * **Q6 : que fait le `--net=host` de la commande exactement ? Je veux voir une utilisation de `nsenter` en réponse à cette question.** Pourquoi avoir besoin de ça sur Keepalived ? 
  * Bonus : à quoi sert `--cap-add=NET_ADMIN` ?
  * **Q7 : le principe de priorité au sein de Keepalived et le fonctionnement simplifié de `vrrp`** (schéma si vous voulez)

# Show me your metrics

Je veux un tableau de bord. Avec des chiffres. **Partout.**  

Ici, afin d'avoir quelque chose de rapidement fonctionnel, on va réutiliser un projet existant et très bien configuré : [swarmprom](https://github.com/stefanprodan/swarmprom).  

Copy/paste du `README.md` du projet :

* prometheus (metrics database) http://<swarm-ip>:9090
* grafana (visualize metrics) http://<swarm-ip>:3000
* node-exporter (host metrics collector)
* cadvisor (containers metrics collector)
* dockerd-exporter (Docker daemon metrics collector, requires Docker experimental metrics-addr to be enabled)
* alertmanager (alerts dispatcher) http://<swarm-ip>:9093
* unsee (alert manager dashboard) http://<swarm-ip>:9094
* caddy (reverse proxy and basic auth provider for prometheus, alertmanager and unsee)

* Je vous laisse suivre la doc du `README.md` pour un déploiement simple. Baladez-vous un peu, surtout sur Grafana qui offre de précieuses visualisations.  

* Expliquez 
  * **Q8 : ce qu'est un 'collector' dans ce contexte**
  * **Q9 : un peu plus en détail le fonctionnement de chacun des tools déployés par cette stack**

# Traefik

[Traefik](https://traefik.io/) est un reverse proxy dédié aux environnements micro-service.   

Au sein d'un Swarm, il est capable de détecter dynamiquement les nouveaux services qu'il doit servir, grâce à un système de *labels*.  

Mettons le en place : 
* architecture de fichiers/dossiers
```
mkdir /data/traefik
mkdir /data/traefik/certs
touch /data/traefik/traefik.toml
touch /data/traefik/docker-compose.yml
```
* générer une paire de clé/cert auto-signé, vous vous en servirez pour la mise en palce de Traefik. Utilisez une wildcard pour votre *COMMON NAME* (exemple : `*.b3.swarm`) : 
```
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /data/traefik/certs/b3.swarm.key -out /data/traefik/certs/b3.swarm.crt
```
* editer le fichier [`/data/traefik/traefik.toml`](./traefik/traefik.toml)
* editer le fichier [`/data/traefik/docker-compose.yml`](./traefik/docker-compose.yml)

* lancer une stack Traefik à l'aide du `docker-compose.yml`

* explorer l'interface de Traefik
* **Q10: expliquez un peu son fonctionnement**

# Dumb Service - Part 3

Adaptez le `docker-compose.yml` de l'app Python pour tourner derrière Traefik en HTTPS uniquement.

# Registry - Part 2

Faites tourner le registre en HTTPS derrière Traefik.

# Registry - Part 3

Faites tourner une stack [Harbor](https://goharbor.io/) plutôt qu'un Registry seul.

# Backup

* **Q11 : Proposez une solution permettant de sauvegarder nos applications**. On veut garder :
  * les configurations
  * les données
  * les outils de déploiement
  * les applications elles-mêmes

* Ici j'attends un peu de réflexion sur la question. Nous avons déjà des outils pour lancer n'importe quel applicatif (stack CoreOS + Docker + Swarm), et un répertoire de données accessibles sur tous nos hôtes (CEPH, NFS). Je ne veux pas un script `bash` qui fait du `rsync` :)

# Récap

* mettez-moi tout ça au clair, proposez : 
  * une mini-doc permettant de déployer un nouveau service sur le Swarm, qui sera monitoré, sauvegardé, servi en HTTPS, et en HA
  * une mini-doc permettant d'ajouter un noeud au swarm (rappel : il doit aussi avoir accès aux répertoire de données, être sauvegardé et monitoré)



# Q2 : expliquer rapidement le principe d'un système de fichiers distribué (distributed filesystem)

Pour commencer, l'acronyme DFS signifie Distributed File System, il s'agit d'un système de fichiers distribués. Il va donc permettre de regrouper un ensemble de partages qu'il faudra rendre accessibles de manière uniforme et de centraliser l'ensemble des espaces disponibles

# Q3 : proposer une façon d'automatiser le déploiement cette conf CEPH (Vagrant ? Swarm stack ? autres ?) Par ex, si on veut rajouter un noeud ?

# Q2 : expliquez le principe d'un partage NFS, quels pourraient être ses limites dans le cas d'un swarm comme le nôtre (qui peut être amené à grandir) ?

# Q3 : proposez une façon d'automatiser le déploiement cette conf NFS


# Q4 : où est lancé le service réellement ? (sur quel hôte, et comment on fait pour savoir ?) Combien y'a-t-il de conteneur(s) lancé(s) ?


# Q5 : pourquoi le service est accessible depuis tous les hôtes ? Documentez vous sur internet.

# Q6 : que fait le --net=host de la commande exactement ? Je veux voir une utilisation de nsenter en réponse à cette question. Pourquoi avoir besoin de ça sur Keepalived ?

#  Bonus : à quoi sert --cap-add=NET_ADMIN ?

Sur Docker l'argument --cap-add va permettre d'ajouter une capabilities. Pour des vérifications d'autorisation, sur UNIX il y a 2 types de processus, les privilégiés qui ignorent les vérifs des autorisations du noyau et les non privilégiés qui sont soumis à une vérification complète des autorisations. ici on ajoute NET_ADMIN qui va : 
Effectuer diverses opérations liées au réseau:
*  configuration de l'interface;
* administration du pare-feu IP, du masquerading et de la comptabilité;
* modifier les tables de routage;
* se lier à n’importe quelle adresse pour un mandataire transparent;
* définir le type de service (TOS)
* effacer les statistiques du conducteur;
* définir le mode promiscuous;
* permettant la multidiffusion;

# Q7 : le principe de priorité au sein de Keepalived et le fonctionnement simplifié de vrrp (schéma si vous voulez)

# Q8 : ce qu'est un 'collector' dans ce contexte

# Q9 : un peu plus en détail le fonctionnement de chacun des tools déployés par cette stack

