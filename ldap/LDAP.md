LDAP
==============

Dans ce TP nous allons apprendre à 
- installer et configurer un serveur LDAP
- créer la structure d'un annuaire LDAP avec des attributs de groupes et d'utilisateurs de compte Linux Posix. 
- peupler  un annuaire LDAP
- rediriger l'authentification d'un service ssh sur un serveur LDAP
- rediriger l'authentification d'un service http access sur un serveur LDAP

Dans le TP suivant nous exploiterons cet annuaire pour authentifier un client distant.

1 Introduction
--------------------------------------------------------------------------------------

Derrière le terme **LDAP** se cache plusieurs aspects :
* Lightweight Directory Access Protocol*
* light = simplification du modèle X.500 édicté par l'UIT-T
* protocole TCP/IP permettant *l'interrogation et la modification d'un annuaire*
* normes pour représenter les systèmes d'annuaire
  * modèle *de données* : comment organiser et typer l'information contenues dans l'annuaire
  * modèles *de nommage* : comment référencer l'information dans l'annuaire
  * modèle fonctionnel : comment accéder à l'information
  * modèle de sécurité : comment protéger l'information
  * modèle de réplication : comment la base est répartie entre serveurs
* *LDIF* (Ldap Data Interchange Format) un format d'échange de données

Ces différents aspects sont une des raisons pour laquelle il est difficile de trouver un bon tutoriel sur le web. Souvent un tuto considère qu'un seul aspect, présente tel ou tel aspect technique, n'est pas forcément à jour...

Dans le cadre de ce TP on utilisera l'implémentation libre OpenLDAP http://www.openldap.org.

2 Théorie
--------------------------------------------------------------------------------------

Vous pouvez consulter les références en bas de la page. Beaucoup de choses mais tout n'est pas à connaître pour faire quelques premières manipulations. Retenez :

Modèle de données

* *structure arborescente* : pensée pour des applications nécessitant plusieurs accès en lecture et peu d'accès en modification

Quelques concepts :

* Les *noeuds* de l'arbre sont appelés des entrées (entry).
* Chaque noeud est identifié par un nom appelé "`distinguished name`" ou `DN`. Celui-ci reprend le `DN` de son noeud parent et rajoute quelque chose.
* Une *entrée* est un ensemble d'attributs `nom:valeur`.
* Il existe quelques *classes* (on dit aussi `objectClass`) prédéfinies largement utilisées pour définir des entrées par exemple : `organizationalUnit, posixGroup, inetOrgPerson, posixAccount`... A chaque classe correspond des attributs obligatoires comme `cn` (`common name`) pour la classe `inetOrgPerson`. Il est possible de créer ses propres attributs et ses propres classes. Les schémas qui définissent les classes les plus communes se trouvent dans le répertoire `/etc/ldap/schema/`.
* Un annuaire est identifié par un "`base name`" ; il correspond en général à votre nom de domaine DNS. Le nom est décomposé en éléments appelés `dc` (`domain component`) ;ainsi "`test.com`" est décomposé en "`dc=test,dc=com`"

Pour initialiser un annuaire, déclarer une organisation des données, remplir cette organisation différentes solutions sont à disposition

* l'utilisation du client en ligne de commande pour demander l'exécution de fichiers `ldif`
* l'utilisation du client web `phpldapadmin`

Dans le cadre de ce TP nous utiliserons principalement le second, plus simple car prenant en charge pas mal de choses...

A noter que les anciennes versions de openldap utilisaient un fichier `/etc/ldap/slapd.conf`. Désormais la configuration est directement des données de l'annuaire et suit une arborescence sur le système de fichier (`ls /etc/ldap/slapd.d/`). Cette dernière solution n'est pas toujours très simple à comprendre de prime abord. En ligne de commandes, beaucoup continue d'utiliser le fichier déprécié `/etc/ldap/slapd.conf` puis exporte la configuration avec les outils qui vont bien. L'utilisation de `phpldapadmin` nous masquera tout cela.

3 Exercice : Comment installer et configurer un serveur LDAP, un client en CLI, un client web sur une debian like ?
--------------------------------------------------------------------------------------

### 3.1 Installation et configuration d'un serveur LDAP

Installer le démon slapd (_Stand-alone LDAP Daemon_) qui écoute les connexions LDAP par défaut sur le port 389 et répond aux opérations LDAP qu'il reçoit via ces connexions.

Une série de questions peuvent vous être posées (cf. ci-dessous). En général au moins le mot de passe pour configurer le serveur LDAP.

    # passer root 
    sudo su
    # apt-get update --allow-releaseinfo-change # en salle réseau iutna en 2021/2022
    apt-get update
    apt-get install slapd

Si l'invite de commande vous demande un mot de passe pour un administrateur de l'annuaire LDAP, donnez lui en un (et retenez le).

A ce stade là vous avez le « démon » slapd qui tourne sur votre machine. Vérifiez qu'il est bien actif avec la commande suivante. Pour rappel, à la place de status vous pouvez mettre start/stop/restart pour démarrer, arrêter ou redémarrer un service.

    $ systemctl status slapd.service

S'il est actif, vous devriez avoir un affichage similaire à celui suivant :

    ● slapd.service - LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)

    Loaded: loaded (/etc/init.d/slapd)

    Active: active (running) since mar. 2018-02-20 18:06:25 UTC; 23s ago

    CGroup: /system.slice/slapd.service

    └─2718 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d

    févr. 20 18:06:25 debian slapd[2717]: @(#) $OpenLDAP: slapd (May 30 2017 07:55:01) $

    buildd@binet:/build/openldap-_xApQ_/openldap-2.4.40+dfsg/debian/build/.../slapd

    févr. 20 18:06:25 debian slapd[2718]: slapd starting

    févr. 20 18:06:25 debian slapd[2713]: Starting OpenLDAP: slapd.

    févr. 20 18:06:25 debian systemd[1]: Started LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol).

    Hint: Some lines were ellipsized, use -l to show in full.

Pour rappel pour consulter les journaux (log) relatifs au service slapd taper la commande suivante. Par défaut vous aurez aussi les informations sur l'état du service. La commande peut éventuellement se combiner (par concatenation) avec un filtre sur le niveau de log tel que ‘-p err'. L'exécuter en tant que root !

    journalctl -u slapd

Reconfigurer le paquet LDAP :

    dpkg-reconfigure slapd

Et répondre comme indiqué ci-dessous

* Omit OpenLDAP server configuration? No
* DNS domain name? Ceci sert à structurer l'arborescence de votre base. Lire le message. Il n'y a pas de règles sur comment le configurer. C'est ce que vous voulez, cela peut correspondre à votre nom de domaine. Pour l'exercice choisir testID.com où ID est votre identifiant de machine
* Organization name? A nouveau, c'est comme vous voulez. Pour l'exercice choisir example
* Administrator password? Utiliser le mot de passe de l'installation ou en choisir un autre.
* Database backend to use? MDB (jusqu'en 2017 je conseillais HDB mais j'ai testé le défaut et cela semble fonctionner)
* Remove the database when slapd is purged? No
* Move old database? Yes
* Allow LDAPv2 protocol? No

Par défaut le `user` et le `group` openldap sont créés sur le système, ainsi que le compte `admin` pour accéder à la base via le mot de passe que vous avez spécifié.

### 3.2 Installation et configuration d'un client en ligne de commande

Le paquet ldap-utils compte quelques utilitaires et notamment des clients openLDAP en ligne de commande :

    apt-get install ldap-utils

utiliser le client CLI (command line) openldap opensearch comme consultation alternative à l'application web. En bas de ce mail vous trouverez une annexe décrivant les arguments les plus communs.

    ldapsearch -x -L -b "dc=testID,dc=com"

Vous devriez avoir un premier résultat. Ci-dessous un rendu avec un annuaire configurée avec ID=25. Combien d'entrées sont retournées ? Quels sont les classes qui définissent les nœuds ?

    version: 1
    #
    # LDAPv3
    # base <dc=test25,dc=com> with scope subtree
    # filter: (objectclass=*)
    # requesting: ALL
    #
    
    # test25.com
    dn: dc=test25,dc=com
    objectClass: top
    objectClass: dcObject
    objectClass: organization
    o: example
    dc: test25

    # admin, test25.com
    dn: cn=admin,dc=test25,dc=com
    objectClass: simpleSecurityObject
    objectClass: organizationalRole
    cn: admin
    description: LDAP administrator

    # search result

    # numResponses: 3
    # numEntries: 2

Vous pouvez aussi jeter un oeil sur l'annuaire de votre collègue (donner le bon testID et n'oubliez pas de configurer l'IP de vos machines respectives...).
      
      ldapsearch -x -L -H ldap://IP_SERVER_DISTANT -b "dc=testID,dc=com"

Vous devriez voir quelque chose de similaire à ce que vous avez vu chez vous. 
Ci-dessous nous allons utiliser le client web phpldapadmin. 

### 3.3 Installation et configuration du client phpldapadmin

Celui-ci va servir à la création effective de l'annuaire et de son contenu.

Dans debian 10 et debian 11, le package phpldapadmin n'est plus disponible (`apt-get install phpldapadmin` ne fonctionnera plus pour installer simplement le paquet). Désormais il faut télécharger _à la mano_ la version la plus récente de phplpdapamin. Pour cela rendez vous sur http://ftp.de.debian.org/debian/pool/main/p/phpldapadmin/. En 2021/22 il s'agit de `phpldapadmin_1.2.6.3-0.2_all.deb`.
Puis faire (depuis le répertoire de téléchargement)
    
    sudo su
    apt-get update --allow-releaseinfo-change
    dpkg -i phpldapadmin_1.2.6.3-0.2_all.deb

si vous rencontrez des problèmes vous pouvez jouer avec les commandes suivantes
    
    apt-get install php-ldap php-xml php
    apt --fix-broken install

Avec les privilèges root ouvrir le fichier /etc/phpldapadmin/config.php

    nano /etc/phpldapadmin/config.php

Changer la valeur `DOMAIN_NAME_or_IP_ADDRESS` pour référencer votre serveur avec le nom du domaine ou adresse IP. Ici, dans le cadre de ce TP, on vous conseille fortement de mettre 127.0.0.1. Si vous choisissez un nom de domaine alors il faudra pouvoir résoudre les noms de domaines (avec un DNS ou au moins avec le fichier /etc/hosts). Limitez l'imprévisible.
    
    $servers->setValue('server','host','DOMAIN_NAME_or_IP_ADDRESS');

La suite requiert de refléter votre configuration du démon slapd.
Ici spécifier votre DNS domain name en format ldif. Pour cela utiliser l'attribut "dc" signifiant "domain component name". "testID.com" devient "dc=testID,dc=com". Attention la String "ID" est à changer par votre identifiant machine !!!
    
    $servers->setValue('server','base',array('dc=testID,dc=com'));
    
    $servers->setValue('login','bind_id','cn=admin,dc=testID,dc=com');

Vous aurez noté que par défaut le fichier à connaissance du compte "admin" sur la base.
Et finalement décommenter la ligne suivante et la mettre à "true pour éviter des alertes ennuyeuses et non importantes.

    $config->custom->appearance['hide_template_warning'] = true;

Sauver et fermer le fichier.

phpldapadmin requiert que vous (re)démarriez apache2 pour fonctionner

   systemctl restart apache2

Désormais vous pouvez vous connecter via votre navigateur à l'adresse suivante

    domain_name_or_IP_address/phpldapadmin

#### 3.3.1 !!! **Troubleshooting** !!! Si vous obtenez une erreur dans votre navigateur du genre « L'url demandée n'a pas pu être trouvée » alors adapter la configuration proxy comme suit :

- Menu Préférence > Avancé > Réseau > paramètres de connexion
- Sélectionner "Configuration manuelle du proxy".
- Définir l'IP "10.0.0.254" et le port "3128" du proxy.
- Cocher "utiliser ce serveur proxy pour tous les protocoles"
- Vérifier qu'il y a bien pas de proxy pour localhost, 127.0.0.1.

#### 3.3.2 !!! **Troubleshooting** !!! Certains d'entre vous rencontreront l'erreur `ldap_bind: Invalid credentials (49).`

Vérifiez que vous avez bien mis le « .com » dans le nom de domaine lors du slapd reconfigure il faut mettre testID.com (et non testID), sans quoi je n'ai rien trouvé de mieux que de redémarrer la machine…


4 Exercice Création d'un annuaire à l'aide de phpldapadmin
--------------------------------------------------------------------------------------

Se connecter au serveur avec le Login DN (distinguished name) suivant "`cn=admin,dc=testID,dc=com`" et le mot de passe que vous avez choisi lors de la configuration de slapd.

Cliquer sur le "+" à côté des domain components (dc=testID,dc=com) et vous verrez le login "admin" que nous utilisons.

LDAP est très flexible, on peut créer des hiérarchies et des relations de plusieurs façons différentes en fonction des cas d'utilisation et des types d'information à accéder. Ici nous allons créér une structure de base puis la remplir par du contenu. Dans un premier temps on construit des catégories et ensuite on remplit ces catégories.

### 4.1 Etape 1 : Création d'unités organisationnelles à savoir `groupes` et `personnes` 

`groupes` va servir à définir les types de groupes d'utilisateurs de votre organisation à savoir le groupe `administrateur` (_aka super user_) et le groupe `utilisateur` (_aka simple user..._). Tandis que `personnes` va servir à déclarer l'identité de chacun des individus de votre organisation avec un attribut spécial pour déterminer si ils sont du groupe `administrateur` ou `utilisateur`.

Sous les domain components,

- Cliquer sur "Créér une nouvelle entrée",
- Choisir "Générique : Unité Organisationnelle"
- Nommer la "groupes" (choix arbitraire)
- Faire Créer l'objet
- Puis Valider la création de l'entrée

La nouvelle entrée devra être visible dans la division de gauche sous le domain component.

*Reproduire la manipulation pour créer l'unité organisationnelle "`personnes`"*.

Utiliser le client opensearch pour observer l'évolution de votre annuaire

    ldapsearch -x -L -b "dc=testID,dc=com"

### 4.2 Etape 2 : Création de sous-entrées sous l'unité organisationnelle `groupes`

Nous allons créer deux types de groupes avec des privilèges distincts : `administrateur` et `utilisateur`.

- Cliquer sur l'unité organisationnelle "`groupes`"
- Dans la division de droite, cliquer sur "créer une sous entrée"
- Choisir "Générique : Groupe Posix". Les comptes POSIX sont le système d'authentification utilisés par LINUX.
- Créer le groupe "`administrateur`"

Reproduire la manipulation pour créer le groupe posix générique "`utilisateur`" comme sous entrée à l'unité `groupes`.

#### 4.2.1 !!! **Troubleshooting** !!! ** 

**Si** lorsque vous cliquez dans la division de gauche sur une unité organisationnelle créée, vous obtenez l'affichage suivant

    Sélectionner un modèle pour le processus de création
    Modèles:        
     [ ] Générique : Entrée Carnet d'Adresse
     [ ] Générique : Groupe Posix
     [ ] Valeur par défaut

**Alors** sélectionnez "Valeur par défaut" !

**Sinon** ne rien faire...


#### 4.2.2 !!! **Troubleshooting** !!! **Aucun GID  indiqué et champnon éditable**
Suivant les versions, le GID (Group ID) est initialisé et incrémenté automatiquement.
Si aucun GID est indiqué et que le champ n'est pas éditable, alors éditer le fichier suivant (en `sudo`)

    nano /etc/phpldapadmin/templates/creation/posixGroup.xml

et commenter la ligne suivante dans l'attribut gidNumber

    <!-- <readonly>1</readonly> -->

Rafraichir, et recommencer la manipulation. Opter pour un gid initial de 500 (vérifier avec la commande `groupes` si il n'est pas utilisé).

### 4.3 Etape 3 : Création d'une première sous-entrée sous l'unité organisationnelle `personnes`

Maintenant nous allons créer deux personnes, chacun avec des droits différents. Ci-dessous, nous prenons comme prénom/nom de personne *Iz Nogoud* et *Haroun ElPoussah* mais prenez deux noms qui vous parlent, et différents de ceux de vos collègues pour vous aider à vous retrouver.

Affectez votre *Iz Nogoud* au groupe `utilisateur` et votre *Haroun ElPoussah* au groupe `administrateur`.

Dans un premier temps créer seulement *Iz Nogoud*.
Les personnes ne peuvent être créées avant les groupes car un attribut des personnes doit indiquer le groupe d'appartenance.

- Cliquer sur l'unité organisationnelle "personnes"
- Dans la division de droite, cliquer sur "créer une sous entrée"
- Choisir "Générique : Compte Utilisateur".
- Créer l'utilisateur "Iz Nogoud"

Suivant les versions les champs à remplir ne sont pas dans le même ordre.

* Common Name: "Iz Nogoud"
* Firstname: "Iz"
* Lastname: "Nogoud"
* uid=1000 (souvent de type uidNumber, vérifier dans `/etc/passwd` si le numéro est libre !)
* GID="utilisateur" (Il faudra mettre *Haroun* dans "administrateur").
* Opter pour un mot de passe chiffré par "crypt" !!!!!!!!!!!
* shell: `/bin/sh`
* home: ce que vous voulez ou bien /home/users/inogoud
* ...

Utiliser le client opensearch pour observer l'évolution de votre annuaire

    ldapsearch -x -L -b "dc=testID,dc=com"

#### 4.3.1 !!! **Troubleshooting** !!!

Si à la création de l'utilisateur, vous rencontrez un **problème similaire que pour le champ GID non éditable**, alors éditer le fichier

    nano /etc/phpldapadmin/templates/creation/posixAccount.xml 

et commenter la ligne suivante dans l'attribut uidnumber.

    <!-- <readonly>1</readonly> -->

Vous noterez ainsi qu'il existe plusieurs fichiers template notamment pour la modification de groupe `/etc/phpldapadmin/templates/modification/posixGroup.xml`... si d'autres problèmes similaires se produisent...

Les versions récentes de phpldapadmin proposent directement un champ GID qui permet d'associer le compte utilisateur à un groupe prédéfinie. Dans les versions plus anciennes il fallait (via l'interface) ajouter un attribut "memberuid", puis  donner la valeur et faire un "update".

Quand l'objet a été créé et validé, à coup de "rafraîchir l'écran" dans les différents menus et "d'affichage par défaut", vous obtiendrez peut être un affichage qui informe du memberuid d'un utilisateur (spécifiant son appartenance à un groupe).

#### 4.3.2  !!! **Troubleshooting** !!!

Si vous obtenez l'erreur suivante "**Error Error trying to get a non-existant value (appearance,password_hash)**" alors consulter la page
http://stackoverflow.com/questions/20673186/getting-error-for-setting-password-feild-when-creating-generic-user-account-phpl


### 4.4 Etape  : Création d'une autre sous-entrée sous l'unité organisationnelle `personnes`

Créer maintenant "*Haroun ElPoussah*". Pour tester la fonction de déplacement et surtout manipuler les "_distinguished name_" créer l'entrée dans le groupe "administrateur" et non dans l'unité organisationnelle "personnes".
Une fois l'entrée créée, la déplacer.
Pour ce faire :
- cliquer sur l'entrée *Haroun ElPoussah*
- cliquer sur "Copier ou déplacer cette entrée" ;
- Vous avez alors deux moyens de déplacer l'entrée soit en parcourant l'arbre soit en spécifiant directement le DN destination. Essayer de définir vous même le DN de destination (vous pouvez jeter un oeil sur "parcourir" ainsi que le DN de Sébastient Faucou pour vous aider. Donnez le DN de départ et celui de destination que vous avez défini.
- Déplacer effectivement l'entrée. N'oublier pas de cocher "Supprimer après la copie (déplacement)" (la copie ne devrait néanmoins pas fonctionner car les comptes posix sont censés avoir des uid uniques...).

Vous avez maintenant configuré un serveur ldap de base avec quelques groupes et utilisateurs. Vous pouvez l'étendre et ajouter toutes les structures organisationnelles que vous souhaitez pour reproduire la structure de votre "entreprise".
Cet aspect là sera abordé par la suite.

Utiliser le client openldap opensearch comme consultation alternative à l'application web.

    ldapsearch -x -H ldap://localhost -b "dc=testID,dc=com" "objectClass=*"

Le contenu de l'annuaire est censé s'afficher dans le terminal.


#### 4.4.1 !!! **Troubleshooting** !!! S'il n'est pas possible de déplacer alors supprimer et recréer la au bon endroit


5 Exercice : Comment exploiter une base LDAP comme moyen d'authenfication distant pour un client ssh
-------------------------------------------------------------------------------------------------------
Cette partie est à réaliser sur une autre machine. Travaillez en binôme. Configurer ce qu'il faut pour utiliser la base ldap distante de votre collègue. Le cas d'usage sera l'authentification ssh.

Configurer votre interface eth1 avec une adresse IP. Elle servira au client pour se connecter à votre annuaire distant.
Verifiez que l'annuaire de votre collègue vous est bien accessible (donner le bon testID cad celui de votre collègue).

    ldapsearch -x -L -H ldap://IP_SERVER_DISTANT -b "dc=testID,dc=com"

LDAP est un moyen de garder des informations d'authentification dans un endroit centralisé. Dans cet exercice, nous allons voir comment configurer une machine cliente pour utiliser un annuaire de comptes distants (LDAP) et s'authentifier localement pour un service ssh.

Pour ce faire, nous allons avoir recours à PAM et NSS.

### 5.1 Installation et configuration de Pluggable Authentication Modules (PAM)

PAM, Pluggable Authentication Modules, est un système qui offre des moyens d'authentification à des applications qui le nécessite mais qui n'en intègre pas nativement. PAM est implémenté dans la plupart des OS et fonctionne de manière transparente.

Sur la machine cliente, installer les paquets suivants (en tant que root)

    apt-get update
    apt-get install libpam-ldap 

libpam-ldap est un module PAM pour interfacer des serveurs LDAP. Il est requis.

A nouveau, on va vous poser diverses questions. Si ce n'est pas le cas relancer la configuration des paquets avec

    dpkg-reconfigure ldap-auth-config # (ubuntu)

ou

    dpkg-reconfigure libpam-ldap # (debian)

Les questions et les réponses à donner sont similaires à celles que vous avez données dans la configuration du serveur :

* LDAP server Uniform Resource Identifier: ldap://LDAP-server-IP-Address Attention à bien changer "ldapi:/" en "ldap:" **Ici donner l'adresse IP du serveur distant de votre collègue** !!!!
* Distinguished name of the search base: Dans notre cas, nous avions choisi "dc=testID,dc=com"
* LDAP version to use: 3
* Make local root Database admin: Yes
* Does the LDAP database require login? No
* LDAP account for root: Dans notre cas cela donne "cn=admin,dc=testID,dc=com".
* LDAP root account password: Ici donner le mot de passer du SERVEUR LDAP défini par votre collègue
* Local crypt to use when changing passwords: crypt

### 5.2 Installation et configuration du Name Service Switch (NSS)

Name Service Switch (NSS) autorise le remplacement des traditionnels fichiers Unix de configuration (par exemple /etc/passwd, /etc/group, /etc/hosts) par une ou plusieurs bases de données centralisées. Pour rappel ces fichiers informent entre autres sur les id et les noms d'utilisateurs et de groupes.


Le démon qui gère ce service s'appelle : nscd, "name service caching daemon".

Sur la machine cliente, installer les paquets suivants (en tant que root)

    apt-get update
    apt-get install nscd

Spécifier où rechercher les fichiers de configuration des solutions d'authentification.
Le fichier /etc/nsswitch.conf permet de spécifier les sources où chercher. Ces sources sont par exemple nis, files (les fichiers /etc/passwd et /etc/shadow locaux), dns...
Pour information, compat un format fondé sur nis... nis et compat sont hors cadre de ce TP. Pour en savoir plus consulter http://serverfault.com/questions/532008/what-is-nsswitch-compat-mode

Editer /etc/nsswitch.conf en tant que root.

    nano /etc/nsswitch.conf

Et ajouter ldap en tête des valeurs possibles pour passwd, group et shadow. Cela peut donner quelque chose comme

    passwd:         ldap compat
    group:          ldap compat
    shadow:         ldap compat

Redémarrer

    systemctl restart nscd.service

### 5.3 Configuration de sshd

Autoriser tous les password logins. Pour cela éditer /etc/ssh/sshd_config et vérifier que la ligne PasswordAuthentication est soit commentée soit fixée à yes.

    #PasswordAuthentication no

ou

    PasswordAuthentication yes

Redémarrer sshd

    systemctl restart ssh.service


### 5.4 Configuration du module sshd de PAM

Editer /etc/pam.d/sshd et insérer les lignes suivantes AVANT toutes les auters lignes de configuration.

    # PAM configuration for the Secure Shell service
    
    auth    sufficient      pam_ldap.so
    account sufficient      pam_permit.so

### 5.5 Configuration de la session commune de PAM

Editer le fichier /etc/pam.d/common-session en tant que root.

    nano /etc/pam.d/common-session

Et ajouter la ligne suivate (peu importe où)
session required pam_mkhomedir.so skel=/etc/skel umask=0022
Cela créera un home sur la machine cliente quand un utilisateur LDAP se connectera dans le cas où il n'a pas de home.

### 5.6 Expérimenter l'authentification ssh via LDAP


C'est ce que tout le monde attendait !!!!
Vous venez de configurer votre système pour consulter un annuaire distant pour les questions d'authentification. Tentez de faire un ssh sur vous-même en indiquant un compte que vous savez être présent dans l'annuaire distant.

Taper la commande suivante

    ssh LDAP_user@LDAP_client_IP_Address

Par exemple

    ssh inogoud@localhost

Cela ne marche et c'est normal... Il manque un truc. Mais cela va vous permettre d'apprendre à lire des logs... cf. paragraphes ci-après. 
Quand vous aurez résolu votre problème et que la commande ssh fonctionnera, taper la commande `whoami` pour bien vérifier que vous êtes bien qui vous croyez être.


### 5.6.1 consulter les journaux de ssh

Il vous faut consulter les journaux de ssh de la machine cliente, soit via un simple

    cat /var/log/auth.log

Soit et c'est plus propre sur le client vous pouvez taper en tant que root :

    journalctl -p err

Faire défiler avec la barre d'espace pour aller à la fin ou bien faire un

    journactl -f -p err

pour sauter directement jusqu'à la fin et CTR+C pour terminer.

ou bien

    journalctl -u ssh

Et peut être aussi le même genre de chose sur le serveur...

#### 5.6.2 !!! Troubleshooting !!! **Invalid user/Invalid credentials**

Si vous constatez l'erreur « Invalid user/Invalid credentials » ici visible avec la commande « journalctl -u ssh »

    févr. 20 19:15:17 debian sshd[11849]: Server listening on :: port 22.

    févr. 20 19:17:54 debian sshd[11858]: Invalid user inogoud from ::1

    févr. 20 19:17:54 debian sshd[11858]: input_userauth_request: invalid user inogoud [preauth]

    févr. 20 19:17:57 debian sshd[11858]: pam_ldap: error trying to bind as user "cn=Iz Nogoud,ou=personnes,dc=test25,dc=com" (Invalid credentials)

    févr. 20 19:17:57 debian sshd[11858]: pam_unix(sshd:auth): check pass; user unknown

    févr. 20 19:17:57 debian sshd[11858]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=localhost

    févr. 20 19:17:57 debian sshd[11858]: pam_ldap: error trying to bind as user "cn=Iz Nogoud,ou=personnes,dc=test25,dc=com" (Invalid credentials)


L'élément clé dans ces logs est "invalid user". Cela signifie que le name service n'arrive pas à identifier l'utilisateur. La conséquence est que sshd refuse de fournir le vrai password au LDAP PAM module. Mais même si il le faisait, vous ne serez toujours pas capable de vous connecter.

Pour vérifier que le name service arrive à vous identifier, vous n'avez pas forcément besoin de lancer le client ssh, vous pouvez aussi taper

    id inogoud

ou bien

    passwd inogoud

La commande id permet de tester si le « user id » et le « group id » de « inogoud » sont connus.

Si vous obtenez respectivement « id: inogoud : utilisateur inexistant » et « passwd : l'utilisateur inogoud n'existe pas », cela signifie que les requêtes NSS (le service qui permet de réorienter la consultation des fichiers qui déclarent les utilisateurs) n'arrivent pas au serveur ldap. (Vous pouvez en lire davantage sur https://wiki.debian.org/fr/LDAP/NSS).

Nous avons « oublié » d'installer le paquet qui permet de  configurer les requêtes NSS via LDAP.
Installer la dépendance suivante chez la machine cliente en le configurant en cohérence avec la définition de l'annuaire. C'est-à-dire en donnant l'IP distant, le DN distant, et le pass distant !!!

    apt-get install libnss-ldap

Vous aurez peut être besoin de le configurer

    dpkg-reconfigure libnss-ldap

Si besoin redémarrer nscd
    
    /etc/init.d/nscd restart

Testez à nouveau les commandes `id`, `passwd` et `ssh`. Les messages et les résultats ont dû changés (la commande passwd ne vous permettra pas de changer le mot de passe « passwd : Impossible de récupérer les informations d'authentification. Mot de passe non changé).

Si la manipulation ci-avant n'a pas fonctionné, vérifiez la configuration de /etc/nsswitch.conf tel que décrit plus haut et redémarrer nscd

    systemctl restart nscd.service

#### 5.6.3   !!! Troubleshooting !!! **Can't contact LDAP server**

Si sur le client vous observez

    Mar 15 21:37:19 hobbes perl: nss_ldap: could not connect to any LDAP server as cn=admin,dc=test,dc=com - Can't contact LDAP server
    Mar 15 21:37:19 hobbes perl: nss_ldap: failed to bind to LDAP server ldapi:///192.168.0.29:389: Can't contact LDAP server
    Mar 15 21:37:19 hobbes perl: nss_ldap: reconnecting to LDAP server...
...
    Mar 15 21:37:19 hobbes perl: nss_ldap: reconnecting to LDAP server (sleeping 1 seconds)...
...
    Mar 15 21:37:20 hobbes perl: nss_ldap: could not search LDAP server - Server is unavailable

Et si la commande suivante fonctionne

    ldapsearch -x -h 192.168.0.29 -b "dc=testID,dc=com" 

ou

    ldapsearch -x -H ldap://LDAP_SERVER_IP -b "dc=testID,dc=com"

(c'est la même chose)

Alors relancer la configuration du package pam sur le client en veillant bien à l'URI indiqué pour joindre le serveur.

6 Exercice Etendre la base en ligne de commande
-----------------------------------------------------------

Le lien suivant https://doc.ubuntu-fr.org/openldap-server décrit comment initialiser, organiser et remplir un annuaire ldap avec les clients openldap en ligne de commandes... Jetez un oeil pour noter que c'est pas évident... et que phpldapadmin vous cache tout de même pas mal de chose.

Observer la sortie d'un affichage de tout le contenu de l'annuaire au format ldif.

    ldapsearch -x -L -h 192.168.0.29 -b "dc=testID,dc=com" 

Quelles classes d'objet ont été utilisées pour définir les groupes et les personnes ?

En isolant les différentes entrées... Editer différents fichiers où vous spécifier différentes définitions.

Tenter de rajouter un groupe "irc"

Créer le fichier ldif qui va bien et charger le dans la base

    ldapadd -x -f irc.ldif -D "cn=admin,dc=testID,dc=com" -W

Tentez de rajouter un utilisateur "As Térix" et de le déclarer dans le groupe "personnes"

Créer le fichier ldif qui va bien et charger le dans la base

    ldapadd -x -f  asterix.ldif -D "cn=admin,dc=testID,dc=com" -W

Tenter de rajouter le sous groupe "superadministrateur dans le groupe "administrateur" ...

Tenter de modifier une entrée... 

7 Annexes
--------------------------------------------------------------------------------------

La commande `ldapsearch` est utilisée de la manière suivante :

    ldapsearch -x -L -C -b searchbase -s base|one|sub -H ldap://hostname:port/ filtre attributs

* Le paramètre « x » permet d'avoir une authentification simple.
* Le paramètre « L » permet d'obtenir un affichage des données selon une forme LDIF.
* Le paramètre « C » permet de suivre les liens de type referrals.
* Le paramètre « b » permet de rechercher les données à partir d'un endroit dans la base. Cela peut être à partir de la racine ou bien à partir d'un sous-arbre.
* Le paramètre « s » permet de spécifier la profondeur de recherche. Trois valeurs sont possibles : base, one et sub. Par défaut, la valeur est sub.
* Le paramètre H permet de spécifier l'adresse et le port du serveur LDAP désiré.
* « filtre » permet de spécifier le filtre de recherche à appliquer. Par défaut "(objectclass=*)"
* « attributs » permet de spécifier quels seront les attributs qui seront retournés en cas de réussite.


Sources en ligne
--------------------------------------------------------------------------------------

Ce cours s'appuie sur différentes ressources en ligne

Concepts
* http://www.linux-france.org/prj/edu/archinet/systeme/ch52.html
* deadlink https://sites.google.com/site/openldaptutorial/Home/openldap---beginners/what-s-needed-to-start
* http://www.openldap.org/doc/admin24/

Configurer un serveur via slapd puis export ldif avant ajout sur le serveur ldap
* deadlink http://t.chemineau.me/blog/2011/01/21/installer-openldap
 
Configurer un serveur en ligne de commande
* http://www.majorxtrem.be/2009/05/05/installation-et-configuration-dun-serveur-openldap-sous-debian/

Configurer via phpldapadmin
* https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-a-basic-ldap-server-on-an-ubuntu-12-04-vps

Authentification posix
* https://www.digitalocean.com/community/tutorials/how-to-authenticate-client-computers-using-ldap-on-an-ubuntu-12-04-vps
* https://wiki.debian.org/fr/LDAP/PAM
* https://wiki.debian.org/fr/LDAP/NSS
* https://wiki.debian.org/LDAP/OpenLDAPSetup
* http://wiki.linuxquestions.org/wiki/Pam_ldap#Configure_LDAP_client_access

Authentification sur apache htaccess et ldap
* https://wiki.debian.org/fr/LDAP
* http://www.majorxtrem.be/2009/05/15/sauthentifier-sur-apache2-avec-ldap/

Search
* forbidden http://wawadeb.crdp.ac-caen.fr/iso/tmp/ressources/cvs.orion.education.fr/homepages/docbooks/ldap/recherche.html


```python

```
