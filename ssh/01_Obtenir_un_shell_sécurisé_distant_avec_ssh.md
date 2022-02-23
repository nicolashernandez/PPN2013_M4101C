# Obtenir un shell sécurisé distant avec ssh

---
## Objectifs

L'objectif de ce travail est d'apprendre à se connecter sur un système distant à l'aide de la commande ssh.


---
## Introduction

Dans le cadre de ce TP on utilisera _OpenSSH_. La version 2 du protocole `ssh` est la plus récente. La commande `ssh` permet de se connecter à distance ou de lancer des commandes à distance. La commande `scp` permet de copier des fichiers à distance. La commande `sftp` fournit une interface FTP pour les nostalgiques.
Il existe plusieurs protocoles Internet permettant de se connecter à un ordinateur distant : `telnet`, les `r-`commandes (`rlogin`, `rsh` ou encore `rcp`), `ssh` (Secure Shell). Alors que `telnet` et les `r-`commandes font circuler les informations en clair sur le réseau, `ssh` est beaucoup plus sûr :
* le client  et le serveur s'authentifient mutuellement, ce qui évite que des pirates se fassent passer pour l'un ou pour l'autre
* les données échangées sur le réseau par le biais de `ssh` sont chiffrées, ce qui garantit leur confidentialité.

À partir dici oublier que `telnet`, `rlogin` ou `ftp` aient pu exister , et ne les utilisez plus que sous la menace.

Pour la petite histoire, ce sujet est inspiré d'un TD proposé par Keryel, Leroy et Ménard, 2003


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
### Configuration de votre environnement réseau
Pour ce travail, vous pouvez au choix utiliser l'interface `JAUNE` ou `BLEU` (les deux sont interconnectées sur le même réseau physique). Par contre, ne touchez pas à la `ROUGE`. Si vous faites un `ip a`, vous pourrez constater que cette interface a une IP pré-configurée. C'est par celle-ci que votre machine reçoit son OS et les mises à jour de ce dernier. 

Supposons que vous choisissiez de travailler avec `JAUNE`. Configurez cette interface à l'aide de la commande `ip`. Chaque machine reçoit une adresse IP privée de classe C de type : `192.168.0.<votreIdentifiantSurLaPriseAuDosDeVotreMachine>`. Fermer `BLEUE` pour contrôler votre réseau (`ip link set BLEUE down`).

Pour rappel pour passer `root` vous pouvez faire `sudo su` ou bien si vous ne voulez pas risque l'accident préfixer par `sudo` toutes les commandes. Cette dernière option ne sera possible que depuis le compte `tdreseau` qui est enregistré dans le groupe des `sudoers`. Le password du compte `tdreseau` devrait être `tdreseau`. 

Tout le monde est tenu de changer le mot de passe de son compte `root`. Pour ce faire utiliser la commande `passwd` en étant connecté en tant que `root` ou bien `passwd root` depuis n'importe quel compte ayant les droits suffisants. Créer ensuite un compte (`adduser`) avec comme nom de login et mot de passe : `distant/distant`.

---
### Prise en main de `ssh`

Personnellement je tourne avec la version `OpenSSH_7.6p1 Ubuntu-4ubuntu0.5` de `ssh`. Et la version d'OpenSSL utilisée est la `1.0.2n  7 Dec 2017`.

Et vous ? Taper `ssh -V`. Attention selon les distributions le `-v` est en minuscule. `man ssh` est votre ami...

Sachant que le service `ssh` tourne sur le port 22, interrogez le avec `nc` (alias `netcat`) :

    nc localhost 22
    
Tourne-t-il ? Si vous obtenez une banière qui vous parle de ssh avec un numéro de version, c'est le serveur qui vous répond en se présentant. 

Si `netcat` ne semble pas être disponible, vous pouvez l'installer avec des droits `root`

    apt-get update
    apt-get install netcat


---
### Lancement du serveur `ssh` et format de demande de connexion

Tentez de vous connecter sur une machine voisine

    ssh compte-distant@ip-machine-distante

Si cela ne marche pas, cela peut venir de plusieurs raisons

#### Permission denied (publickey)

A partir de la debian 8, et c'est valable pour la debian 10, une tentative de connexion `ssh` vous retourne le message `Permission denied (publickey)` lorsque vous commandez une connexion sur un compte donné présent une certaine machine.

Ainsi dans l'exemple suivant on observe ce que donne la commande de connexion `ssh` sur un compte `distant` présent sur la machine `192.168.0.32` : 

    tdreseau@debian:~$ ssh distant@192.168.0.32
    distant@192.168.0.32: Permission denied (publickey).

Ce problème peut avoir plusieurs raisons et il y a plusieurs façons de le résoudre. Nous allons le résoudre d'une manière : 

Editer `/etc/ssh/sshd_config`

    nano /etc/ssh/sshd_config

Puis "_enable challenge-response passwords_" en passant à yes la ligne suivante.

    ChallengeResponseAuthentication yes

Recharger le fichier de config 

    systemctl restart ssh.service 

Et votre commande `ssh` fonctionnera

#### ssh: connect to host ... port 22: Connection refused

Si vous obtenez le message `ssh: connect to host ... port 22: Connection refused` cela peut venir du fait qu'il n'y a pas de serveur sur la machine distante ou bien que le serveur ssh n'est pas actif (` systemctl status ssh.service`).

Afin que chacun puisse se connecter par `ssh` sur votre machine lancer, exécuter la commande suivante avec les droits `root`:

    systemctl start ssh.service

---
### Se connecter via ssh sur une machine distante d’un collègue.

Remarquez qu’à la première connexion `ssh` sur un nouveau serveur, votre client `ssh` dit qu’il ne connaît pas la clé publique du `sshd` de la machine distante et l’affiche. 

    tdreseau@debian:/home/user# ssh distant@192.168.0.32
    The authenticity of host '192.168.0.32 (192.168.0.32)' can't be established.
    ECDSA key fingerprint is d6:b5:86:2d:26:6b:6c:75:83:b6:ea:d4:6e:06:1a:87.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '192.168.0.32' (ECDSA) to the list of known hosts.

En cliquant sur `yes`, comment être sûr que la clé présentée est bien celle du `sshd` sur lequel on veut se connecter et que ce n'est pas la clé d'un `sshd` pirate qui espionnera tout ce que l'on fera ?

Le seul moyen d'être sûr de se connecter sur le bon `sshd` est de posséder la clé proposée dans son trousseau de _clés connues_ ( `~/.ssh/known_hosts`). 

Ainsi face à ce dilemne de la question qui vous est possée à la connexion, plusieurs solutions s'offrent à vous :
- vous allumez un cierge au dessus de votre écran et vous priez pour que ce soit la bonne clé qui est présentée ; 
- vous vous déplacez physiquement jusqu'à la machine qui héberge le `sshd` et récupérez la clée désirée sur une disquette, un CDROM, une carte ou un ruban perforée afin de la copier ensuite à la mano dans votre trousseau ;
- vous convertissez la clé proposée en binaire, jetez des dés, regardez à l'extérieur, et récitez 3 fois l'alphabet à l'envers et là vous avez oublié ce que vous vouliez faire ;
- vous n’avez pas pas lu la question et vous avez cliqué `yes` sans réfléchir.

La bonne solution se cache parmi ces options.

Si vous possédez la clé publique de `sshd` dans `~/.ssh/known_hosts`, alors votre client `ssh`, reconnaissant une machine amie, ne vous posera pas la question précédente. De surcroît, si votre connexion est détournée, `ssh` refusera la connexion car la clé renvoyée par le `sshd` distant ne sera probablement pas celle de `~/.ssh/known_hosts`.

Continuez la connexion en acceptant l'ajout et authentifiez vous. Puis déconnectez vous (`CTRL + D`) et reconnectez vous. L'affichage est-il le même ? 

Supprimez le fichier et tentez de vous connecter. L'affichage est-il le même qu'à la question précédente ?

Observez le contenu du fichier `~/.ssh/known_hosts` avant/après une 1ère connexion ou 2e connexion sans suppresion de fichier. 

---
### Auditer votre connexion

Testez les commandes `last`, `ss` (`netstat` est déprécié), `nmap`, `who`, `ip` (`ifconfig` est déprécié) avec les options qui vont bien sur le poste pour déterminer qui se connecte chez vous.

Quelle commande vous permet d'identifier combien de personnes sont connectées sur votre machine ? 

Quelles commandes vous informent sur le login, l'IP de ces personnes ?

Testez ces commande sur le compte `distant` après connexion. Quelle commande vous permet de savoir que vous êtes bien sur distant ? 

Essayer de sniffer le contenu d'une connexion `ssh` avec `wireshark` (pas nécessairement une connexion entrante ou sortante). Identifiez vous qui est connecté sur qui ? Arrivez vous à lire le contenu ?

---
### Pour aller plus loin

Après avoir supprimé `~/.ssh/known_hosts`, lancez une connexion en mode verbeux (voir le `man`) et analyser la procédure.

