# docker-traefik
Intégration Docker API / Treafik avec gestion des domaines/sous domaine

> Ce "hands-on" est parti d'une problématique rencontré avec [Sébastien Laporte](https://github.com/seblaporte) sur la mise en production d'une serie d'application derrière un traefik sans interruption de service de reverse-proxy.

Le but de ce petit projet est de montrer l'intégration de [Traefik](https://traefik.io/) et de son interaction avec l'API docker. 
Cette interaction permet la prise en compte des container monté en tant que route au sein du reverse proxy qu'est Traefik à l'aide des labels docker-compose.
***Et surtout si tous les container, Traefik et les container branché sur Traefik sont au sein d'un même reseau*** ce qui permet de ne pas linker Traefik au nouveau container, et donc de ne pas arreter le reverse-proxy pour une prise en compte.

> Les labels de compose permettent l'exposition de pseudo propriété qui peuvent être utilisées par des service tiers, dans notre cas Treafik. 
>
> Mais nous verrons cela plus loin.

Nous étion parti sur le principe de créer un reseau via le fichier descriptif docker-compose.
Mais il s'est avéré que docker-compose va créer un réseau en le préfixant avec le nom du repertoire ou se trouve le fichier.
Ce prefixage peut être modéré en fixant un nom de projet comme cela :

```shell
docker-compose -p shipstone -f traefik/docker-compose.traefik.yml up
```
Mais au final, cette méthode s'est révélé peu satidaisante, et a surtout mis en lumière notre non compréhénsion de la philosophie du docker-compose...
Et surtout c'était peu satisfaisant, car cela oblige à marquer d'uun même projet des descriptif compose qui ne font justement pas partie d'un même projet !

Il faut le reconnaitre, c'est surtout une non connaissance du fonctionnement du principe de reseau au sein de docker qui m'a induit en erreur.

Alors, partant d'une page blanche, voici pour nous la meilleure manière de gérer la mise en exposition de site de manière automatique dans le contexte d'un deploiement au sein de docker.

La premiere étape de créer un reseau indépendament de tout (on le nommera ```shipstone_net```) en utilisant le CLI de gestion des reseaux de docker : 
```shell
$ docker network create shipstone_net
e0d6ecb92f898f3422e3c7e0729ab86d4d9c2f0a446a203338b538a9cdc946b4
```

On peut donc verifier la création du réseau : 
```shell
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
b8f5bc540db4        bridge              bridge              local
72c5dfdf71bf        host                host                local
e0d6ecb92f89        shipstone_net       bridge              local
```
Maintenant il ne reste plus qu'a le déclarer comme réseau "externe" au niveau de chaque compose : 

```yaml
networks:
    shipstone_net:
        external: true
```

et de déclarer le rattachement au niveau de chaque container (dans le cas de notre container Taefik, par exemple) : 
```yaml
services:
    traefik:
        image: traefik:latest
        container_name: traefik
        volumes:
            - ./traefik.toml:/etc/traefik/traefik.toml
            # liaison socket API Docker (pallier bug ? version beta ????)
            - /var/run/docker.sock:/var/run/docker.sock
        networks:
          - shipstone_net
        ports:
            - "80:80"
            - "8080:8080"
```
Rattachement qui se fait via la section ```networks```. Et il faut faire cela pour chaque container.

Décrivons un peu notre cas de test. Il s'agit d'exposer un domaine (```shipstone.local```) et deux sous domaines (```web1.shipstone.local``` et ```web2.shipstone.local```). Chaque sous domaine ayant 2 backend (2 nGinx en load balancing). 
Pour simuler cela, j'ai juste modifié mon host afin d'ajouter des URL pointant sur mon localhost. Je vous donne une petite astuce au passage (j'ai rencontré le problème sur OSX), si la carte reseau de votre PC prend en charge l'IPv6, la seule définition de la liaison avec l'IP localhost en IPv4 (```127.0.0.1```) entraine une latence pas possible. L'astuce consiste à déclarer un double mapping (IPv4 & IPv6) : 
```
# shipstone local
fe80::1%lo0	shipstone.local
127.0.0.1	shipstone.local
fe80::1%lo0   web1.shipstone.local
127.0.0.1   web1.shipstone.local
fe80::1%lo0   web2.shipstone.local
127.0.0.1   web2.shipstone.local
```

Maintenant, il faut déclarer l'adhérence de Traefik à l'API Docker (dans le fichier ```traefik.toml```) :
```toml
[docker]
  endpoint = "unix://var/run/docker.sock"
  domain = "shipstone.local"
  watch = true
``` 
Vous remarquerez qu'il écoute un socket UNIX (que les barbus de passage me pardonne ma non connaissance du fonctionnement d'un UNIX en ce qui concerne les écoutes).
Dans mon cas, visiblement docker ne créé pas cette liaison, c'est pourquoi je l'ai rajouté en tant que volume mappé dans le compose de mon Traefik (cf. plus haut).

Vous noterez aussi que je défini un domaine par défaut (```shipstone.local```), ce qui nous permettra d'acceder à la console de Traefik [http://shipstone.local:8080](http://shipstone.local:8080).

Maintenant lançons notre compose pour Traefik (sans prefixage par un nom de projet) : 
```shell
docker-compose -f traefik/docker-compose.traefik.yml up
```
Une fois cela fait, vous pouvez ouvrir la console Traefik, et voir qu'une seule route a été créer en mappant le domaine de base sur le container de Traefik.

Maintenant, comment configurer Traefik pour que les container décrit dans mes deux fichiers compose (front1 et front2) soit pris en compte ?

L'astuce reside dans le fait de decaler les informations de configuration du reverse-proxy de chaque container au niveau des labels exposé par ceux-ci et que Traefik pourra interpreter via sa liaison sur l'API Docker.
Les labels devront être prefixé par ```traefik``` et décrire ce que l'on trouve d'habitude dans le fichier de configuration de Traefik. Exemple pour le serveur 1 du Front 1 pour le sous domaine ```web1.shipstone.local``` :
```toml
        labels:
            - "traefik.backend=front_http_1"
            - "traefik.frontend.rule=Host:web1.shipstone.local"
            - "traefik.port=80"
            - "traefik.backend.loadbalancer.method=wrr"
            - "traefik.backend.loadbalancer.sticky=false"
            - "traefik.protocol=http"
            - "traefik.weight=10"
``` 
Nous y configurons (dans l'ordre d'apparition) : 
* le nom du serveur back
* la route du front, le sous domaine, mais on peut aussi ajouter d'autre règle (voir même faire de la récriture d'URL, mais cela fera l'objet d'un autre exemple)
* le port exposé
* La méthode de load-balancing, juste informatif, car elle porte une valeur par défaut ```wrr```
* Si on reste adhérent à une IP (session), même chose je ne l'ai mis qu'a titre indicatif, car je le change pas la valeur par défaut.
* On fixe le protocole utilisé - *Petit rappel : Traefik joue avec de l'HTTP2, ce qui veux dire que par défaut, il sert du https (port 443)*.
* Le poid du backend, utilisé pour le load balancing.

Maintenant, on peut lancer les container de nos fronts : 

```shell
docker-compose -f front/docker-compose.front1.yml up
```
et
```shell
docker-compose -f front/docker-compose.front2.yml up
```

Et si vous regardez dans le console Traefik, vous les verrez apparaitre ! *C'est de la magie ? Non, c'est Traefik et Docker :bowtie: 



