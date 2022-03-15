# Déni de service d'un serveur

Ce travail sera réalisé par groupe de 3-4 étudiants.

Configurez au moins 3 machines sur un même réseau IP e.g. 192.168.X.0/24 où X est l'identifiant d'une de vos machines. Chaque groupe utilisera le même identifiant pour être sur le même réseau.

- L'un des ordinateurs jouera le rôle de serveur 
- Un autre un client
- Et un ou plusieurs autres des attaquants

A titre d'illustration le serveur sera désigné par l'adresse `192.168.1.1` par la suite.

Attention veillez à ce que votre adresse IP ne "saute" pas. Exécutez des wiresharks pour observer.


## Communication "normale"

Depuis la machine client, vérifiez que vous pouvez récupérer une page web du serveur web distant. 

La commande suivante exécute une requête http et retourne le statut de la réponse ainsi que le temps total pour recevoir une réponse.

Donnez la signification des paramètres.

    curl -s -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null 192.168.1.1
    # Http code: 200
    # Total time:0.010976s

Si vous obtenez une erreur 403, donnez une requête simple qui vous indique ce qui ne marche pas.

Réessayez la commande en supprimant le proxy dans le terminal courant

    http_proxy=

La commande suivante calcule le temps moyen d'obtenir une réponse en 200 suite à une requête http sur 100 tentatives :  

     for i in `seq 1 100`; do curl -s -w '%{time_total} ' -o /dev/null 192.168.1.1 ; done | perl -ne '@sum = split(/ /, $_); $sum = 0; for ($j = 0 ; $j<=$#sum; $j++) {$sum += $sum[$j];}; $sum = $sum / ($#sum+1) ; print ("Average time $#sum+1 runs : $sum\n")'

Calculez ce temps moyen dans une situation idéale.

## Attaque flood ping

Faites tourner un wireshark sur la machine cliente.

Sur la machine de l'attaquant, que signifie les paramètres -f, -i, -l, -s et éventuellement -b de la commande `ping` suivante ?

    sudo ping -f 192.168.1.1 -s 65000 -i 0

Exécutez l'attaque et calculez le temps moyen depuis le client pour récupérer une page. Le déni est-il effectif ? Vous pouvez augmenter le nombre d'attaquant, changer les valeurs de la commande ou tester la suivante.

## Attaque flood TCP SYN

Depuis la machine de l'attaquant, installez un autre outil vous permettre de faire un autre type d'innondation

    sudo su # et non en sudo
    apt-get update --allow-releaseinfo-change
    apt-get update
    apt-get install hping3

Après avoir expliqué les paramètres de la commande suivante, lancez l'attaque.

    sudo su
    hping3 -S -p 80 -i u10 --flood 192.168.1.1
    # HPING 192.168.1.1 (BLEUE 192.168.1.1): S set, 40 headers + 0 data bytes
    # hping in flood mode, no replies will be shown

Puis quasi au même moment calculez le temps moyen depuis le client. Le déni est-il effectif ? 

Qu'observez-vous avec wireshark ? L'attaque aboutit-elle à un déni total ?

Trouvez un moyen d'observer ce qui se passe sur le serveur en terme de connexions. 
Indice : utilisez ss et recherchez les états SYN-RECV. Attention c'est très fugace.

Sinon comment identifer l'IP de l'attaquant ?


## Une réduction possible de l'effet de l'attaque

Il vous est possible de protéger le serveur. Les commandes suivantes sont à réaliser sur le serveur.

En supposant que 192.168.1.2 est l'attaquant, vous pouvez rejeter systématiquement tous ses paquets entrants.

    iptables -I INPUT -s 192.168.1.2 -p tcp -j REJECT

En théorie, le temps moyen n'est pas altéré.

Une autre solution est limiter le nombre de syn concurrentes par seconde ainsi que le nombre de connexions possibles par IP :

    # Limit the number of syn concurrency to 1 per second
    iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT

    #Limit the number of newly established connections for a single IP in 60 seconds to 10
    iptables -I INPUT -p tcp --dport 80 --syn -m recent --name SYN_FLOOD --update --seconds 60 --hitcount 10 -j REJECT

Calculez le temps moyen. Le déni est ressenti ? Quelle adaptation proposez-vous de ces commandes contre les attaques par ping flood ?

