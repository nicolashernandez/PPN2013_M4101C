# Créer un tunnel sécurisé et faire suivre des services

---
## Objectifs

L'objectif de ce travail est d'apprendre à manupuler des tunnels sécurisés.


---
## Votre travail

Il vous est demandé de rédiger avec vos mots un guide succin des commandes que vous utilisez (quelle commande avec quel paramétrage pour quel usage). 


---
## A votre disposition

Pour réaliser ce travail, les réponses sont à chercher dans le support de cours et les `man`(uels) des commandes suggérées. 

Je vous invite aussi à relire sur madoc
* "l'aide mémoire des commandes "linux"" dans la section "généralités" de ce cours,
* ainsi que les éléments de "Configuration du navigateur et du terminal pour accéder au web" tout en gardant l'accès aux machines de la salle.

Ci-dessous une sélection des commandes les plus importantes vous permettant d'auditer l'état des services ouverts, des ports en écoute, et des connexions en cours sur votre machine.
- `ss -taupe` (`netstat` déprécié) permet de connaître les services qui tournent sur votre machine, les programmes clients qui attaquent des services distants ainsi que les noms des programmes exécutés, les connexions _established_, les IP/port source et cible. Pour les services connus sur les ports réservés (cad déclarés dans `/etc/services`) le nom du service est donné plutôt que son port.
- `nmap -A -T4 -p1-65535 IP` vous informe les services qui tournent sur une machine désignée par son `IP`
- La commande `last` montre la liste des dernières connexions sur la machine (elle fonctionne à partir du journal `/var/log/wtmp`). Si jamais l'un de vous s'amuse à arrêter volontairement la machine d'un autre, non seulement il serait identifié mais en plus il aurait 1 point en moins au DS.
- De même n'hésitez pas à abuser des commandes `who` et `ip` (`ifconfig` est déprécié) 
- `wireshark` pour voir qui communique et le contenu ce qui est communiqué. A noter que l'écoute du réseau implique que le mode promiscious soit activé. Habituellement cette activation n'est seulement effective qu'avec un wireshark lancé avec des droits root. Dans le système actuel (février 2022), wireshark depuis root ne se lance pas mais l'écoute est possible depuis un compte utilisateur.
- La commande `nc` (alias `netcat`) est présentée comme le couteau suisse de la gestion des connexions TCP/IP.
Netcat est un outil très facile d'utilisation qui peut lire et écrire sur des connexions réseaux (socket) utilisant les protocoles TCP ou UDP.
Il permet ainsi de faire du debugging de réseau, de l'exploration (port scanning). Il peut agir en tant que client ou en tant que serveur, servir à du transfert de données (input standard ou fichier), exécuter des scripts lors de connexions, voire même servir de relais (proxy) entre pour un serveur.
Netcat peut aussi être utilisé pour diverses autres pratiques : banner-grabbing, Tar, Shell Shoveling UDP, port scanning UDP, Spoofing, Cryptcat... Nous ne les verrons pas ici.


---
## Configuration de votre environnement réseau
Pour ce travail, vous pouvez au choix utiliser l'interface `JAUNE` ou `BLEU` (les deux sont interconnectées sur le même réseau physique). Par contre, ne touchez pas à la `ROUGE`. Si vous faites un `ip a`, vous pourrez constater que cette interface a une IP pré-configurée. C'est par celle-ci que votre machine reçoit son OS et les mises à jour de ce dernier. 

Supposons que vous choisissiez de travailler avec `JAUNE`. Configurez cette interface à l'aide de la commande `ip`. Chaque machine reçoit une adresse IP privée de classe C de type : `192.168.0.<votreIdentifiantSurLaPriseAuDosDeVotreMachine>`. Fermer `BLEUE` pour contrôler votre réseau (`ip link set BLEUE down`).

Pour rappel pour passer `root` vous pouvez faire `sudo su` ou bien si vous ne voulez pas risque l'accident préfixer par `sudo` toutes les commandes. Cette dernière option ne sera possible que depuis le compte `tdreseau` qui est enregistré dans le groupe des `sudoers`. Le password du compte `tdreseau` devrait être `tdreseau`. 

Tout le monde est tenu de changer le mot de passe de son compte `root`. Pour ce faire utiliser la commande `passwd` en étant connecté en tant que `root` ou bien `passwd root` depuis n'importe quel compte ayant les droits suffisants. Créer ensuite un compte (`adduser`) avec comme nom de login et mot de passe : `distant/distant`.


---
## Se connecter sur une machine distante non directement accessible

Parfois on souhaiterait pouvoir relier directement deux machines appartenant à deux réseaux distincts. C'est par exemple mon cas quand j'ai besoin d'accéder à ma machine du boulot (machine "BOULOT" sur le réseau "à l'entreprise") depuis ma machine personnelle (machine "MAISON" sur le réseau "à la maison"). Dans ce genre de cas, il existe en général une machine sur le réseau "à l'entreprise" qui possède à la fois une interface vers le réseau extérieure et une interface susceptibles d'accéder à chacune des machines du réseau interne de l'entreprise. Nous appellerons cette machine RELAIS, car elle servira de relai au tunnel.

Postulat :

* MAISON possède un client SSH capable de créer un tunnel (ssh, par exemple)
* RELAIS possède un serveur SSH qui tourne et qui est accessible de l'extérieur ; notamment par MAISON ; vous possédez un compte utilisateur sur cette machine.
* BOULOT a des services qui sont accessibles aux seules machines internes dont notamment RELAIS ; vous possédez un compte sur le service souhaité qui tourne sur cette machine.

Bien, allons y franchement, la commande, à executer depuis MAISON, permettant de créer un tunnel SSH entre MAISON et BOULOT est la suivante :

    ssh <login_sur_RELAIS>@<RELAIS_IP>  -L <MAISON_port>:<BOULOT_IP>:<BOULOT_port> 

L'option `-L` indique que le données arrivant sur le port `<MAISON port>` de la machine locale doivent être envoyées sur le port `<BOULOT port>` de la machine `<BOULOT IP>` en sortie d'une connexion SSH sur la machine `<RELAIS IP>`.

En anglais, on utiliser le terme bind/binder pour désigner le fait de "lier" un port d'une machine à un port d'une autre machine.

Pour le reste de l'exercice vous travaillerez en trinôme, chacun pouvant jouer les rôles de MAISON, RELAIS et BOULOT à la fois.

---
### Mettre les mains dans le cambouis

Vous êtes MAISON. Demandez à BOULOT de lancer un serveur apache et de mettre un fichier `msg.txt` dans `/var/www`.

Puis créer un tunnel passant par RELAIS pour atteindre BOULOT qui commence sur le port 7777 de votre MAISON. Ensuite, de MAISON, simuler un client web avec netcat pour récupérer le contenu de msg.txt en passant par votre tunnel. Est ce que cela marche ? Pourquoi si vous lancer le client dans le même terminal qui a permis l'établissement du tunnel ca ne marche pas ?

L'option `-f`, qui force `ssh` à s'exécuter en tâche de fond, peut elle vous aider à ne pas faire cette erreur ? L'option `-N`, qui permet de pas donner de prompt après la connexion à SSH, peut elle vous aider à ne pas faire cette erreur ?

A partir d'un compte utilisateur arrivez vous à forwarder un port privilégié de 1 à 1024 ?

L'option `-g` permet d'autoriser les connexions d'autres machines, différentes de celle qui établie la connexion, à utiliser le canal sécurisé.

Arrivez vous à sniffer le contenu de la communication ? La partie MAISON-RELAIS devrait être protégé mais pas la partie RELAIS-BOULOT. Remarquez que RELAIS et BOULOT pourraient être la même machine. Et dans ce cas votre communication serait sécurisée. Faire tourner un serveur apache avec un `msg.txt` dans `/var/www` sur relais. Quelle commande tappez vous pour mettre en place un tunnel de MAISON à RELAIS sans rebond ?

Mettre en place un tunnel avec ou sans rebond en sortie vers une machine BOULOT. Demandez à un de vos collègues d'essayer d'utiliser votre tunnel à partir de sa machine. Nous appelerons cette machine COLLEGUE. Y arrivez vous ? L'option -g ajoutée à la commande ssh lors du création du tunnel apporte t elle quelque chose ?

Remarques : L'option `-2` force ssh à utiliser la version 2 du protocole (A ne pas utiliser si on utilise de vieux serveurs SSH )

