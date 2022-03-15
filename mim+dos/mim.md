# Attaque par Sniffing et attaque Man-In-the-Middle (MIM) de type ARP spoofing à l'aide de la technique ARP poisoning


## 1. Objectif

Ce travail a pour objectif de sensibiliser aux attaques par sniffing ainsi que celles de type MIM. L'attaque par sniffing sera rendu possible après l'attaque MIM. L'attaque MIM sera illustré dans le cadre d'un réseau local par la technique de *ARP Poisoning*.

Via cette attaque, un *pirate* pourra détourner (*dé-commuter*) une communication entre deux ordinateurs distants (i.e. présents sur d'autres ports d'un switch) afin que la communication passe par lui et qu'il puisse consulter le contenu de celle-ci.

Une attention sera aussi portée sur les techniques pour observer ce type d'attaque.


## 2. Préparation (A LIRE avant d'AGIR)

Lire entièrement le sujet. L'action débutera à la section *Exercices pratiques*.

En 2021-2022, la salle réseau compte 2 grandes tablées avec chacune 2x8 ordinateurs à peu près sur lesquels vous pouvez être _root_. Des ordinateurs sont interconnectés via des hub et des hub sont interconnectés via des switches, de la sorte que des ordinateurs se trouvent sur des liaisons au switch différentes. Via l'interface BLEUE, les ordinateurs d'une même tablée _devraient_ être connectés sur un même hub, les deux tablées étant réunies par un switch. La salle possède aussi d'autres ordinateurs qui peuvent vous fournir un accès "ouvert" à l'Internet. 

Pour cet exercice vous travaillerez par groupe de 3 étudiants avec la contrainte que 2 étudiants/ordinateurs doivent se trouver derrière un port de switch et l'autre derrière un autre port. 

Chaque ordinateur sera interconnecté via une adresse IP privée de type : 192.168.0.<votreIdentifiantSurLaPriseAuDosDeVotreMachine>/24. 


## 3. Exercices théoriques : ARP spoofing et ARP poisoning

_Qu'est ce que le *sniffing* ? Qu'est ce qu'une *attaque MIM* ? Qu'est le *spoofing* ? Comment fonctionne le *poisoning* ? Pourquoi celui-ci est possible avec le protocole *ARP* ?_


## 4. Exercices pratiques

Avant de passer à la partie pratique, jetez un oeil aux sous-titres de la section "Manipulation utiles". 


### 4.1 Interconnexion au niveau liaison

Dans la baie de brassage, chaque machine est identifiée grâce à un numéro qui correspond à l'identifiant de la prise RJ45 sur laquelle la machine est branchée (voir au dos de la machine sur la table ou bien sur le plan). Dans la baie, sont aussi précisées les interfaces. Finalement la baie supporte l'interconnexion de certaines machines via des hubs (même domaine de collision) et 1 switch (même domaine de diffusion).

Vérifiez dans la baie de brassage que vous avez bien chacun de vos deux groupes d'ordinateurs est bien connecté sur le même hub, que ces hubs sont bien différents et qu'ils sont bien interconnectés via le switch. La connectique peut être changée à la discrétion de votre chargé de TD ou bien vous pouvez permuter des ordinateurs avec d'autres collègues.


### 4.2 Configuration réseau

Configurez votre interface réseau. Fermez l'interface que vous n'utilisez pas, pour contrôler votre réseau. Ne touchez pas à l'interface ROUGE...


### 4.3 Ecoute de ce qui se dit sur votre hub

Vérifiez que pour entendre une communication il faut que l'un des 2 ordinateurs impliqués dans la communication soit sur votre hub. En théorie, si vous écoutez ce qu'un serveur http répond à une requête, vous êtes censé pouvoir lire le contenu du message retourné. Vous utiliserez wireshark pour cela.


### 4.5. Ecoute de ce qui se dit sur une autre liaison du switch

Vérifiez que vous n'entendez pas la communication entre 2 ordinateurs qui ne sont pas sur votre hub. Vous utiliserez wireshark pour cela.

Avant de mettre en oeuvre l'attaque ARP poisoning, notez l'adresse MAC de l'ordinateur pirate qui lancera l'attaque ainsi que celle des victimes.

Depuis l'ordinateur pirate et l'ordinateur serveur, suivez la mise à jour des associations ARP/IP sur le réseau (`arpwatch`).

Ecoutez aussi le réseau pour observer la signature de l'attaque à venir (`wireshark`) ainsi que le contenu de la trame http du serveur que vous voulez connaître.

Passez à l'action en exécutant l'attaque ARP poisoning (`ettercap`) pour détourner la communication. Constatez l'état des caches arp sur les ordinateurs victimes. 


## 5. Manipulations utiles

### Pour simuler un trafic http entre un client et un serveur web 

En tant que serveur, _placez dans le répertoire `/var/www/html` un fichier texte `msg.txt` avec un contenu quelconque et démarrez apache._

En tant que client, _demandez en continu cette ressource sur un de vos collègues à l'aide de la commande suivante_

    watch -n 1 wget -O msg.txt --no-proxy IPSERVEUR/msg.txt


### Pour cibler votre écoute avec wireshark

Filtrer les trames transportant de l'http en provenance d'une certaine IP  

    ip.src == 192.168.1.1 and http

Filtrer les trames où au moins de ces IP apparaissent en source ou en distination

    ip.addr == 192.168.1.1 or ip.addr == 192.168.1.2 

Fitrer les trames où l'adresse MAC mentionnée apparait en source ou en distination

    ether.addr == b0:5b:24:33:c1:f1 


### Pour consulter l'état de votre cache ARP 

Consulter son cache ARP et connaître les associations IP/MAC apprises à travers les échanges (notamment ARP) 

     ip neigh

Supprimer des entrées dans son cache ARP

      ip n flush dev BLEUE


### Pour suivre l'état de l'appariement de MAC avec des adresses IP

Description

    Arpwatch is a tool that keeps track of ethernet/ip address pairings. It will report on the changes of both IP and MAC addresses. It stores its entries into a database, a file, arp.dat by default. Also, by default, arpwatch will write its output to syslog.

    It listens for traffic via libpcap, it then records IP/MAC parings into the database.
    When it receives a packet, it compares the src/dst (IP & Ethernet) to those in the database, if it matches, it does nothing. If the pair is not in the database, it will create a new entry, and lastly, if it finds a match but the match is not exact e.g. IP address or MAC address is different, it generates an alert.

Installation

    sudo su # et non en sudo
    apt-get update --allow-releaseinfo-change
    apt-get update
    apt-get install arpwatch

Quelques options

    (-i) listen on BLEUE (one daemon for each interface), 
    (-f) use default database file, 
    (-m) send mail to root user, 
    (-u) drop process to user: arpwatch, 
    (-N) ignore bogons, 
    (-Q) prevents arpwatch from sending e-mail.

Exemples de commande (ne pas les exécuter maintenant)

    /usr/sbin/arpwatch -i BLEUE -f BLEUE.dat -m root -u arpwatch -N -Q
    # ou 
    arpwatch -i BLEUE -Q

    # puis 
    sudo tail -f /var/log/syslog


### Pour autoriser sa machine à router des paquets

Spécifier les variables fichiers suivants

    echo 1 > /proc/sys/net/ipv4/ip_forward
    echo 1 > /proc/sys/net/ipv6/ip_forward


### Pour mettre en oeuvre une attaque ARP poisoning

La commande ettercap est un des outils qui permet la mise en oeuvre d'une attaque ARP poisoning.

Installez le paquet `ettercap-text-only`

    sudo su # le `sudo` du simple user semble moins bien fonctionner
    apt-get update --allow-releaseinfo-change
    apt-get update
    apt-get install ettercap-text-only 

Autorisez votre machine à router des paquets.

**Avant toute exécution de la commande permettant de "dé-commuter" une communication, lire le `man` pour comprendre la signification des paramètres de la commande en question** :

    ettercap -i BLEUE -T -q -M arp:remote /192.168.1.1// /192.168.1.2// -w result

