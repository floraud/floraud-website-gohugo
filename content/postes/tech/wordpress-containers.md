+++
title = 'Déploiement de containers WordPress sur un serveur avec Docker Compose'
categories = ["Tech"]
tags = ["DevOps", "Docker Compose", "Container", "WordPress", "Traefik"]
featured_image = "/images/tech/wordpress-containers_cover_erwan-hesry.webp"
date = 2024-01-14T18:07:07+01:00
+++

## Introduction

Nous allons étudier au cours cet article comment déployer plusieurs containers WordPress avec leurs bases de données et leurs certificats Let's Encrypt sur un seul serveur avec Docker Compose. Mon **but** est d'**optimiser l'utilisation de mon serveur VPS au maximum**, **faire des économies** et aussi pouvoir **PoCer plus facilement des solutions d'infra** qui proposent souvent des containers pour tester leurs services rapidement. Le code lié à cet article est disponible sur [le repo sur mon serveur Forgejo du lab](https://git.floraud.fr/floraud/wordpress-docker).

### Définition des concepts

- **Container** : Un **environnement isolé sur un serveur** dans lequel est **cloisonnée une application ainsi que ses dépendances**. Son intérêt réside dans son isolation car tout en partageant les ressources du noyau, les différents containers peuvent chacun avoir une version différente d'une dépendance qui aurait pu être commune sur le serveur. Par exemple, dans le contexte des serveurs WordPress, chaque conteneur peut avoir sa propre version de PHP sans interférer avec l'environnement du serveur ou des autres containers, offrant ainsi une gestion des dépendances plus flexible et évitant les conflits potentiels entre les différentes applications hébergées sur le même serveur. Aussi, ils sont **indépendants de l'environnement hôte**, ce qui permet leur portabilité et de les déployer aussi bien sur un PC personnel que sur un serveur Windows ou Linux.
- **Docker** : Docker est une plateforme de conteneurisation qui permet de **gérer lesdits containers**. Il regroupe **différents outils permettant l'exploitation des containers** tels que son moteur exécutant les containers, le registre pour les consulter ainsi que leurs volumes de données et leurs réseaux. Il y a aussi **Docker Compose** permettant de spécifier les paramètres de lancement des containers. Docker fournit aussi des images officielles standardisées mises à disposition par les mainteneurs des applications via son catalogue **Docker Hub**.
- **Let's Encrypt** : Autorité de certification permettant de **générer des certificats** gratuitement et automatiquement pour activer l'**HTTPS** sur les sites web et offrir une identification ainsi qu'un chiffrement de bout en au bout aux utilisateurs.
- **Traefik** : Dans notre cas, ça sera notre **reverse proxy**, c'est-à-dire que ça sera l'application qui **triera les requêtes HTTP** en fonction du nom de domaine contenu dans la requête pour les diriger vers le bon container. Il nous permet aussi de **gérer le certificat Let's Encrypt et rediriger le trafique de l'HTTP vers l'HTTPS**.
- **WordPress** : Un CMS (Content Management System) permettant de **gérer simplement le contenu d'un site web** comme celui sur lequel vous êtes actuellement connecté. WordPress est le plus populaire et il permet d'alimenter son site ou son blog sans avoir à mettre les mains dans le code. Son container contient les dépendances nécessaires au bon fonctionnement de l'application (Apache, PHP...) et nécessite l'utilisation d'un container de base de donnée en parallèle.

## Déploiement du lab

### Prérequis

- Un **serveur** Linux fraîchement installé (**lab testé sur Alma 8.5** et Debian 12). Personnellement j'ai pris l'offre STD-2 sur **[PulseHeberg](https://pulseheberg.com/cloud/vps-linux)**, c'est 4€ par mois, ça fonctionne bien et c'est français.
- **Docker installé** : ça prend 5 minutes en suivant la [documentation officielle](https://docs.docker.com/engine/install/), il faut suivre la Platform CentOS si vous prenez Alma ou Rocky.
- **Enregistrement DNS** pour chaque site que l'on veut déployer. Par exemple : *site1.votredomaine.com, site2.votredomaine.com*...
- Téléchargez le **[code du git du lab](https://git.floraud.fr/floraud/wordpress-docker/archive/main.tar.gz)** en fonction de l'hôte. Sur Alma il suffit de faire `curl -L0 https://git.floraud.fr/floraud/wordpress-docker/archive/main.tar.gz | tar -xz` pour télécharger le code et décompresser le fichier.

### Architecture

L'objectif à la fin est de disposer de l'architecture suivante sur le serveur :
- Seul Traefik doit être en contact avec Internet.
- Chaque container WordPress ne doit entrer qu'en communication avec Traefik et son propre MariaDB.
- Chaque site a son propre réseau isolé de l'autre site.

![topologie](/blog/images/tech/wordpress-containers_topology.webp)

Pour que le code soit plus facilement exploitable, il a été divisé en différents fichiers :
- `docker-compose.yml` : contient la configuration réseau.
- `docker-compose-site1.yml` et `docker-compose-site2.yml` contiennent les informations relatives aux différents sites que l'on va mettre sur le cluster.
- Pour tout autre site que l'on voudrait ajouter, il suffit d'ajouter un réseau dans le fichier `docker-compose.yml` relatif à Traefik comme nous le verrons et copier un des fichiers `docker-compose-sitex.yml` en modifiant les variables nécessaires.
- Il y a finalement les dossiers comprenant les fichiers de mot de passe par défaut qu'il faudra modifier avec vos propres comptes et mdp.

### Premier fichier : Traefik

Traefik, configuré dans le fichier `docker-compose.yml`, sera le cœur de notre architecture. Comme présenté auparavant, c'est lui qui va rediriger les requêtes aux différents WordPress sur la base des URL appelées et qui va obtenir les certificats Let's Encrypt pour le chiffrement des sites. On déclare aussi dans ce fichier nos réseaux. On va étudier les différentes lignes de code que l'on y trouve. 

```yml
version: "2.21.0"

services:
  traefik:
    image: "traefik:v2.10"
    container_name: "traefik"
    command:
      #- "--log.level=DEBUG"
      #- "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      # Redirection vers l'https
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entryPoints.web.http.redirections.entrypoint.scheme=https"
      # HTTPS
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      #- "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      - "--certificatesresolvers.myresolver.acme.email=exemple@domain.com" # Do not forget to modify mail address
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      #- "8080:8080"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

    networks:
      wp1-net:
      wp2-net:
  
networks:
  wp1-net:
  wp2-net:
```

- `version` : On donne la version de Docker Compose pour laquelle le fichier doit être interprété.
- `services` : Nomme les services employés, ici c'est `traefik` et c'est ainsi qu'on y fera référence si besoin pour d'autres services.
- `image` : On explicite quelle image de Docker Hub utilisée. Personnellement je préfère expliciter la version afin de la figer et de pouvoir corréler en cas de problème un bug avec une version. Il est aussi possible de créer ses images Docker customisées mais nous ne le verrons pas dans cet article.
- `#- "--log.level=DEBUG"` : utilisé de concert avec `#- "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"` pour du debug certificat, je l'ai laissé en as de besoin.
- `#- "--api.insecure=true"` associé à la ligne `- "8080:8080"` dans les ports permet d'avoir accès au dashboard Traefik. Dans le cadre d'un PoC ça peut être utilisé pour comprendre un peu le fonctionnement, mais n'y voyant pas d'utilité, je l'ai désactivé.
- `- "--providers.docker=true"` : on dit à Traefik de surveiller Docker via son API pour voir les nouveaux containers mis en ligne.
- `networks` : Le premier niveau, celui avec la même indentation que `volumes` sert à placer le container Traefik dans les réseaux généraux que l'on a déclarés ensemble, à savoir `wp1-net` et `wp2-net`. Le deuxième, tout en bas du fichier, sert à créer les réseaux en question. Un /16 par défaut.
- `- "--providers.docker.exposedbydefault=false"` : on lui dit de ne pas exposer par défaut tous les containers qu'il voit via l'API Docker, comme les bases de données où les ***sidecars*** WP-CLI.
- Le bloc ci-dessous permet simplement de rediriger le trafique HTTP qui arrive sur le port 80 vers l'HTTPS pour que tout le trafique soit chiffré.
```
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entryPoints.web.http.redirections.entrypoint.scheme=https"
```
- `- "--entrypoints.websecure.address=:443"` : On définit le port **443** pour utiliser l'HTTPS avec le point d'entrée websecure.
- `- "--certificatesresolvers.myresolver.acme.tlschallenge=true"` : on active le challenge TLS pour pouvoir demander un certificat Let's Encrypt.
```
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      #- "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      - "--certificatesresolvers.myresolver.acme.email=exemple@domain.com" # Do not forget to modify mail address
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
```
- `volumes` : on explicite où stocker le certificat Let's Encrypt.
- `"/var/run/docker.sock:/var/run/docker.sock:ro"` : permet à Traefik d’interagir avec le démon Docker en montant le fichier socket du système hôte pour apprendre dynamiquement la présence d'un container.

### Deuxième fichier : stack du site web
Sans revenir sur les catégories déjà définies, nous allons étudier les lignes spécifiques au site web. Ce fichier permet de déclarer dans leur réseau respectif un container WordPress associé à une base de donnée et un container WP-CLI qui permet de préconfigurer le site web. J'utilise WP-CLI parce que je trouvais ça ennuyeux de refaire la configuration de base avant de fournir les identifiants de connexion à un client.

```yml
version: "2.21.0"

services:  
  mariadb-1:
    image: "mariadb:11.1.3-jammy"
    restart: "always"
    container_name: "mariadb-1"
    volumes:
      - "db-data-1:/var/lib/mysql"
    environment:
      - "MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db1-root"
      - "MYSQL_DATABASE_FILE=/run/secrets/db1-db"
      - "MYSQL_USER_FILE=/run/secrets/db1-user"
      - "MYSQL_PASSWORD_FILE=/run/secrets/db1-pwd-user"
    networks:
      - "wp1-net"
    labels:
      - "traefik.enable=false"
    secrets:
      - "db1-root"
      - "db1-db"
      - "db1-user"
      - "db1-pwd-user"

  wordpress-1:
    depends_on: 
      - "mariadb-1"
    image: "wordpress:6.4.1"
    restart: "always"
    container_name: "wordpress-1"
    volumes:
      - "wp-data-1:/var/www/html"
    environment:
      - "WORDPRESS_DB_HOST=mariadb-1"
      - "WORDPRESS_DB_USER_FILE=/run/secrets/db1-user"
      - "WORDPRESS_DB_PASSWORD_FILE=/run/secrets/db1-pwd-user"
      - "WORDPRESS_DB_NAME_FILE=/run/secrets/db1-db"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress-1.rule=Host(`site1.floraud.fr`)"  # Do not forget to change this line and have your dns record
      - "traefik.http.routers.wordpress-1.entrypoints=websecure"
      - "traefik.http.routers.wordpress-1.tls.certresolver=myresolver"
    networks:
      - "wp1-net"
    secrets:
      - "db1-db"
      - "db1-user"
      - "db1-pwd-user"
  
  wordpress-cli-1:
    depends_on:
      - "mariadb-1"
      - "wordpress-1"
    image: "wordpress:cli"
    container_name: "wordpress-cli-1"
    environment:
      - "WORDPRESS_DB_HOST=mariadb-1"
      - "WORDPRESS_DB_USER_FILE=/run/secrets/db1-user"
      - "WORDPRESS_DB_PASSWORD_FILE=/run/secrets/db1-pwd-user"
      - "WORDPRESS_DB_NAME_FILE=/run/secrets/db1-db"
    volumes:
      - "wp-data-1:/var/www/html"
    entrypoint: sh
    command: -c 'sleep 40; wp core install --url="site1.floraud.fr" --title="toto" --admin_name=toto --admin_password="toto" --admin_email=toto@toto.com' # You have to change url, title, admin_name, admin_password and admin_email. After deployment, do not forget to change admin_password
    networks:
      wp1-net:
    labels:
      - "traefik.enable=false"
    secrets:
      - "db1-db"
      - "db1-user"
      - "db1-pwd-user"

volumes:
  db-data-1:
  wp-data-1:

secrets:
  db1-root:
    file: "db1-secret/db1-root.txt"
  db1-db:
    file: "db1-secret/db1-db.txt"
  db1-user:
    file: "db1-secret/db1-user.txt"
  db1-pwd-user:
    file: "db1-secret/db1-pwd-user.txt"

networks:
  wp1-net:
```

- `restart` : Permet de toujours redémarrer le conteneur en cas de problème, il existe aussi `no`, `unless-stopped` ou `on-failure`.
- `environment` : Permet de passer des variables d'environnement au container, ici on lui définit les secrets qui sont configurés après pour ne pas mettre les mots de passe en dur dans le Docker Compose.
- `container_name` : permet de référencer le container plus facilement dans Docker plutôt que laisser Docker lui donner un nom aléatoire mais ce n'est pas lié au Docker Compose comme `service`.
- `secrets` : On donne des noms des fichiers qui contiennent les variables secrètes comme les mots de passe.
- `labels` : Permet de définir l'interaction avec Traefik. Si on souhaite que Traefik expose le service ou non, par quel nom de domaine le service est appelé, sur quel protocole et si on souhaite obtenir un certificat Let's Encrypt.
- `depends on` : Indique que le service dépend d'un autre service et Docker démarre les services dans l'ordre que l'on configure.
- `entrypoint` : Comment on entre sur le container. Ici c'est `sh` parce que l'on entre via le shell.
- `command` : qui suit l'`entrypoint` pour passer la commande shell que l'on désire.
 
 ## Exploitation

 ### Paramétrage

- Dans `/db1-secret` et `/db2-secret`, modifiez les comptes et mots de passe.
- Dans `docker-compose.yml`, modifiez votre adresse mail pour savoir si votre certificat arrive à expiration à cette ligne `- "--certificatesresolvers.myresolver.acme.email=exemple@domain.com"`.
- Dans `docker-compose-site1.yml`, modifiez :
	- ```"traefik.http.routers.wordpress-1.rule=Host(`site1.floraud.fr`)" : pour mettre votre domaine.```
	- ``command: -c 'sleep 40; wp core install --url="site1.floraud.fr" --title="toto" --admin_name=toto --admin_password="toto" --admin_email=toto@toto.com : changez avec vos informations.``
	- Faire de même avec vos autres sites pour les fichiers `docker-compose-site[x].yml`.

### Déploiement

Dans le dossier, lancez le déploiement des containers à l'aide de `docker compose -f docker-compose.yml -f docker-compose-site1.yml up -d` (ça prend environ 40 secondes pour que le sidecar WP-CLI fasse son office, vous pouvez allonger ou réduire le temps en fonction de la réactivité de votre host en remplaçant `command: -c 'sleep 40` par la valeur que vous préférez).
- Vous pouvez lancer le site2 à l'aide de `docker compose -f docker-compose-site2.yml up -d`.
- Vous pouvez arrêter les containers en utilisant les commandes `docker compose -f docker-compose-site2.yml down` ou `docker compose -f docker-compose.yml -f docker-compose-site1.yml down`.

### Visualisation et navigation

- Voir les logs d'un container : `docker logs <name>`.
- Contrôler les containers toujours présents : `docker ps`.
- Contrôler les volumes toujours présents : `docker volume ls`.
- Contrôler les réseaux toujours présents : `docker network ls`.
- Pour supprimer tous les volumes créés qui ne sont pas en cours d'exécution `docker volume prune --all`.
- Pour supprimer des réseaux qui ne sont pas en cours d'exécution `docker network prune --all`.

## Conclusion

Bien que ce lab soit imparfait pour de la production car il n'y a pas d'orchestration pour gérer le cycle de vie des containers et la haute-disponibilité (ce que permet de faire un outil comme Kubernetes), on est maintenant en mesure de déployer simplement et à volonté des containers sur notre serveur pour réaliser nos tests sans nous ruiner. Cela nous a permis de découvrir le concept de container et une des solutions majeures sur le marché qu'est Docker pour pouvoir monter des labs rapidement.

*Photo de bannière par [Erwan Hesry](https://unsplash.com/fr/@erwanhesry?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) sur [Unsplash](https://unsplash.com/fr/photos/plusieurs-conteneurs-de-fret-RJjY5Hpnifk?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)*