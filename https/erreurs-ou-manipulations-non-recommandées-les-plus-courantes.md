## Erreurs (ou manipulations non recommandées) les plus courantes 

* utiliser un nom de domaine dans une url de votre navigateur mais ne pas l'avoir déclaré dans un fichier /etc/hosts

* générer un certificat auto signé sans passer par la création d'une CA comme indiqué dans http://httpd.apache.org/docs/2.4/ssl/ssl_faq.html ; par ailleurs utile ensuite pour les navigateurs clients

* dans apache, éditer le `default.conf` plutôt que d'en créer un propre (e.g. `tp.conf`)

* Sous ubuntu 18.04, si quand vous testez votre config `tp` via votre navigateur http://wiki.VOTRENOM.org/tp.html, vous obtenez la page par défaut index.html d'apache, désactivez la config `000-default` qui semble être prioritaire sur la vôtre. Faire `a2dissite 000-default`

* Sous ubuntu 18.04 et 20.04, si vous obtenez une erreur `key too small` telle que "`141A318A:SSL routines:tls_process_ske_dhe:dh key too small`", il vous faut regénérer la clef privée de votre serveur (`servwiki.key`) et de de l'autorité de certification (`ca.key`). Une autre alternative est de baisser l'exigence de sécurité mais c'est mal... https://askubuntu.com/questions/1233186/ubuntu-20-04-how-to-set-lower-ssl-security-level

* Sous windows, si vous obtenez l'erreur `unable to load Private Key 81272:error:0909006C:PEM routines:get_name:no start line:crypto\pem\pem_lib.c:745:Expecting: ANY PRIVATE KEY` avec la commande `openssl req -new -key .\servwiki.key > .\servwiki.csr`, c'est qu'il s'agit probablement d'un problème d'encodage de votre fichier `servwiki.key`. Vous pouvez utiliser "notepad" pour vérifier l'encodage et sauver sous un autre encodage à savoir `us-ascii` ou `utf-8`. Par exemple l'encodage "`UCS-2 LE BOM`" n'est pas un encodage souhaité.

* s'assurer que le `servername` de votre `virtualhost` dans la conf apache est le même que le `common name (FQDN)` de votre demande de signature de certificat (possible problème d'erreur d'identifiant signalé plus tard dans le navigateur à la connexion sur le serveur)

* oublier d'activer le `virtual host` avec un `a2ensite` (pour le rendre "`enabled`")
* dans le `virtualhost` répondant en https (tps.conf), donner les clefs de la CA et non celle du wikiserv

* ajouter la ligne `Listen 443` du fichier `ports.conf` alors que le sujet indiquait de l'ajouter "Si nécessaire...". Source de l'erreur: "`févr. 26 17:01:33 debian systemd[1]: Failed to start LSB: Apache2 web server.`"

* ne pas lire les logs d'Apache (comme demandé dans le sujet) à chaque étape de configuration d'apache2 pour détecter les erreurs.

* tester le site tps en entrant l'url du site avec le préfixe `http://` et non `https://`

* accepter l'exception dans le navigateur pour valider le serveur au lieu de chercher à bien vérifier que le serveur est correctement configuré puis que le navigateur importe bien le certificat qui va bien

* 2020-2021 Certains navigateurs lèvent une alerte signalant la présence d'un certificat auto-signé...

* Si votre navigateur n'arrive jamais sur `/var/www/html2` et que vous avez bien fait un a2ensite (enabled) sur votre conf dans apache alors faire un a2dissite sur le conf par défaut.
