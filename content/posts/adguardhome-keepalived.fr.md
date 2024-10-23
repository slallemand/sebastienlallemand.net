---
date: 2024-10-22
# description: ""
# image: ""
lastmod: 2024-10-22
showTableOfContents: true
title: "Déploiement d'AdGuard Home en haute-disponibilité avec Keepalived"
type: "post"
tags: ['adguard','ha','keepalived','self-hosting','dns','podman', 'pihole', 'pi-hole']    
---
Voilà plusieurs années que j'utilise un bloqueur de publicités et de traqueurs sur mon réseau local à la maison.  
Jusqu'à présent, j'utilisais [Pi-Hole](https://pi-hole.net) mais lors de la refonte de mon infrastrucure (article à venir) j'ai également étudié l'idée d'utiliser [AdGuard Home](https://adguard.com/fr/adguard-home/overview.html) car il offre quelques fonctionnalités supplémentaires (DoH, DoT, contrôle parental pour ne citer que ceux là ...).  
Mais une architecture touchant le DNS est souvent très sensible (_"It's always DNS"_) et nécessite donc un peu de haute-disponibilité.

![DNS](/images/dns.jpg)

## Fonctionnement d'AdGuard Home (et de Pi-Hole)
AdGuard Home et Pi-hole sont des logiciels qui agissent comme des bloqueurs de publicités sur un réseau domestique.  

En filtrant les requêtes DNS, ils empêchent les publicités et les traqueurs de se charger. Cela améliore non seulement l'expérience de navigation en rendant les pages plus rapides, mais cela peut aussi augmenter la confidentialité des utilisateurs. Les 2 fonctionnent avec des listes de serveurs de publicités mises à jour quotidiennement et permettent d'ajouter des listes qui vous sont propres.

Pour que cela fonctionne, il va donc falloir déployer 1 de ces logiciels mais aussi forcer les périphériques de votre réseau à les utiliser (je ne rentrerai pas les détails car cela dépend de votre fournisseur d'accès et de votre "box internet").

### Déploiement d'AdGuard Home avec une Quadlet [Podman](https://podman.io/)
Dans la mesure du possible, je déploie tous les services dont j'ai besoin en utilisant des conteneurs avec [podman](https://podman.io/). Outre le fait qu'il fonctionne facilement en mode rootless (c'est mieux pour la sécurité), une des fonctionnalités que j'apprécie est le système de Quadlets.  
Les Quadlets permettent en gros de lancer des conteneurs comme des services **SystemD** (je vous entends les gens qui n'aimaient pas systemd ...).

Ma Quadlet pour AdGuard Home ressemble au fichier suivant ; les plus observateurs auront vu que dans ce cas précis, ce conteneur s'éxécute en mode rootfull (c'est une simplification que j'ai prise pour publier simplement le service sur le port 53). 

```bash
# /etc/containers/systemd/adguardhome/adguardhome.container
[Unit]
Description=AdGuardHome Quadlet
After=local-fs.target

[Container]
AutoUpdate=registry
Environment="TZ=Europe/Paris"
Image=docker.io/adguard/adguardhome:latest
DNS=9.9.9.9
DNS=2620:fe::9
Label="traefik.enable=true"
Label=traefik.docker.network=systemd-traefik-net
Label="traefik.http.routers.adguardhome.rule=Host(`adguard.$DOMAIN`)"
Label="traefik.http.routers.adguardhome.entrypoints=websecure"
Label="traefik.http.routers.adguardhome.tls.certresolver=MYRESOLVER"
Label="traefik.http.services.adguardhome.loadbalancer.server.port=3000"
Label="traefik.http.routers.dns.rule=Host(`dns.$DOMAIN`)"
Label="traefik.http.routers.dns.entrypoints=websecure"
Label="traefik.http.routers.dns.tls.certresolver=MYRESOLVER"
PublishPort=53:53
PublishPort=53:53/udp
PublishPort=853:853
Volume=adguardhome-config.volume:/opt/adguardhome/conf
Volume=adguardhome-work.volume:/opt/adguardhome/work

[Install]
# Start by default on boot
WantedBy=multi-user.target default.target

[Service]
Restart=always
TimeoutStartSec=60
```

_Remarque_ : les lignes `Label=` ne sont utiles que parce que j'utilise [traefik](https://traefik.io/traefik/) pour exposer mes services sur mon réseau.

Cela fait, il nous reste un peu de configuration/adpatation à nos besoins et cela se fait directement dans l'interface d'AdGuard Home (qui se retrouve exposée sur https://adguard.$DOMAIN dans mon cas).  
Par défaut, AdGuard Home utilise les serveurs DNS de [Quad9](https://quad9.net) en upstream. Quad9 est (_je  les cite_) une fondation Suisse dont le but est de fournir un Internet plus sûr et plus robuste pour tout le monde.  
Vous pouvez bien évidemment utiliser ceux que vous voulez mais pour ma part j'ai décidé de les utiliser en privilégiant leur version DoH (DNS over HTTPS) https://dns.quad9.net/dns-query.  
Les détails des différentes options disponibles se trouve [ici](https://quad9.net/fr/service/service-addresses-and-features).


## Bon travail ... mais 
Nous avons maintenant notre service fonctionnel mais testons son fonctionnement.  
AdGuard est démarré sur la machine ayant l'adresse ip 192.168.1.200 :
```bash
$ dig @192.168.1.200 ad.doubleclick.net +short
0.0.0.0
```

Bingo ! AdGuard Home nous répond 0.0.0.0 qui évidemment ne sera pas joignable !

On reconfigure sa box Internet pour utiliser cette adresse IP sur tout le réseau et tout va bien, nous pouvons aller dormir ... Ah oui ? ... Il se passe quoi si la machine qui héberge AdGuard Home est arrêtée ou reboot ou brûle ou ... est au minimum inaccessible ?  
Ce sont toutes les machines/laptops/smartphones/tablettes qui ne fonctionnent plus car elles ne peuvent plus faire de résolution DNS !  


![Internet is down](/images/internet-down.gif)

__"_Ca marche plus Internet !!!!_"__ (quelqu'un dans la maison)


### Keepalived à la rescousse
Evidémment si on a une seule machine faisant tourner ce service c'est vite embêtant et on voit tout de suite le SPOF (Single Point Of Failure).  
Dans mon cas, en plus du (vieux) NUC sur lequel je déploie mes services, j'ai aussi 1 Raspberry Pi qui peut tout à fait prendre le relais pour ce service.  
Néanmoins, **hors de question** de devoir reconfigurer mon réseau pour que toutes les machines utilisent une nouvelle adresse IP.  
L'idée va être d'utiliser une adresse IP flottante qui ira là où le service est actif (avec une préférence pour le NUC).  
Et la bonne nouvelle, c'est que [Keepalived](https://www.keepalived.org/) est tout à fait en messure de faire çà mais aussi qu'il est disponible sur toutes les distributions Linux.

### Mise en place de Keepalived
Après avoir déployé AdGuard Home sur une 2ème machine et avoir installé Keepalived (`dnf install keepalived` par exemple), on va devoir le configurer.  
Plusieurs modes de fonctionnement sont disponibles mais dans mon cas, je veux basculer automatiquement une adresse IP d'un serveur à un autre avec une préférence pour un serveur si le service est démarré sur les deux.

Ma machine principale (le NUC) aura le rôle de MASTER et l'autre (le Raspberry Pi) SLAVE dans cette configuration.  

Sur le serveur MASTER :
```bash
# /etc/keepalived/keepalived.conf
global_defs {
    enable_script_security
    script_user root
}

vrrp_script chk_adguardhome {
       script "/usr/libexec/keepalived/chk_adguardhome"
       interval 2                      # check every 2 seconds
}

vrrp_instance adguardhome {
        interface bridge0
        state MASTER
        virtual_router_id 50
        priority 200
        authentication {
                auth_type PASS
                auth_pass s0me_random_p@ssw0rd
        }
        virtual_ipaddress {
                192.168.1.3/24 dev bridge0
        }

        track_script {
                chk_adguardhome
        }

}
```


Sur le serveur SLAVE :
```bash
global_defs {
    enable_script_security
    script_user root
}

vrrp_script chk_adguardhome {
       script '/usr/libexec/keepalived/chk_adguardhome'
       interval 2                      # check every 2 seconds
}

vrrp_instance adguardhome {
        interface eth0
        state BACKUP
        virtual_router_id 50
        priority 100
        authentication {
                auth_type PASS
                auth_pass s0me_random_p@ssw0rd
        }
        virtual_ipaddress {
                192.168.1.3/24 dev eth0
        }

        track_script {
                chk_adguardhome
        }
}
```

Les points notables :
- `state` : MASTER ou SLAVE
- `priority` : le serveur avec la plus haute priorité gagne
- `virtual_ipaddress` : où on définit l'adresse IP qui va se déplacer
- le script défini par `chk_adguardhome` : c'est ce script qui va vérifier l'état du service. Si le script renvoie 0, adguardhome est démarré sinon, il est arrêté.  
Dans mon cas avec une Quadlet, le script est celui-ci :
```bash
#/usr/libexec/keepalived/chk_adguardhome
#!/bin/sh

/usr/bin/systemctl is-active --quiet adguardhome
```

### Est-ce qu'on peut les synchroniser ?
Le problème avec cet architecture est que la plupart du temps, 1 seul serveur va être mis à jour. Lorsque l'on modifie la configuration sur un serveur (ajout de filtres par exemple), il va falloir le faire sur le 2ème aussi.  
Là encore, bonne nouvelle car il existe un utilitaire ([adguardhome-sync](https://github.com/bakito/adguardhome-sync)) qui va permettre de synchroniser une instance vers 1 ou des réplicas.

Sur le serveur principal, on va juste ajouter une nouvelle Quadlet:
```bash
#/etc/containers/systemd/adguardhome/adguardhome-sync.container
[Unit]
Description=AdGuardHome Sync Quadlet
After=local-fs.target

[Container]
AutoUpdate=registry
Environment="TZ=Europe/Paris"
Image=ghcr.io/bakito/adguardhome-sync
Label="traefik.enable=true"
Label=traefik.docker.network=systemd-traefik-net
Label="traefik.http.routers.adguardhome-sync.rule=Host(`adguard-sync.$DOMAIN`)"
Label="traefik.http.routers.adguardhome-sync.entrypoints=websecure"
Label="traefik.http.routers.adguardhome-sync.tls.certresolver=MYRESOLVER"
Label="traefik.http.services.adguardhome-sync.loadbalancer.server.port=8080"
Volume=./adguardhome-sync.yaml:/config/adguardhome-sync.yaml

[Install]
# Start by default on boot
WantedBy=multi-user.target default.target

[Service]
Restart=always
TimeoutStartSec=60
```

Et voilà, notre 2ème serveur AdGuard Home sera automatiquement et régulièrement mis à jour !

## Bonus : fonctionnement hors du réseau local
Avoir le serveur DNS qui filtre localement c'est bien mais si vous sortez de votre maison, votre Smartphone va se retrouver en 4G/5G et donc subir toutes les publicités et autres traqueurs.  
Pour remédier à ça, j'ai décidé de me connecter systématiquement en VPN chez moi et donc d'hériter d'AdGuard Home. J'ai utilisé Wireguard disponible nativement sur ma Box Internet (merci [Free](https://free.fr)) mais tout autre solution vous permettant un accès VPN fonctionnera.  
On peut même décider de ne faire passer QUE la résolution DNS à travers ce VPN.

## Conclusion
Avec tout cela, on a maintenant une architecture DNS filtrant les publicités, redondante et permettant à notre DNS de répondre dans (presque) tous les cas et qui de surcroît nous ajoute une couche de confidentialité et de sécurité grâche aux serveurs DoH de Quad9 !.  
J'espère que vous aurez trouvé l'article utile. 

## Références
- [AdGuard Home](https://adguard.com/fr/adguard-home/overview.html)
- [Podman](https://podman.io/)  
- [Quadlet](https://www.redhat.com/en/blog/quadlet-podman)
- [Keepalived](https://www.keepalived.org/)