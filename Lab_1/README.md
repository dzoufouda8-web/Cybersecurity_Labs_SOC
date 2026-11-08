Lab1_Détection d'une attaque SSH et d'un scan de ports avec Splunk

OBJECTIF : 
Simuler deux attaques classiques (force brute SSH et scan de ports) contre une machine Ubuntu, puis les détecter dans Splunk avec des requêtes SPL, des alertes planifiées et un tableau de bord. Ce lab realisé a titre d'apprentissage personnel reproduit le travail d'un analyste SOC de niveau 1 : repérer un comportement anormal dans des logs, le confirmer et le documenter.

**************************ARCHITECTURE ET ENVIRONNEMENT :*********************************************

  1- Machine Hote : serveur SIEM ; outils : Splunk Entreprise
  2- VM Ubuntu : Cible ; outils : Splunk Forwarder,ufw ,Openssh 
  3- VM Kali : Attaquant ; outils : Hydra, Nmap
  Réseau :  Virtualbox

*************************FLUX GÉNERAL DE FONCTIONNEMENT :**********************************************

La VM kali lance les attaques vers la VM ubuntu, lorsque la VM Ubuntu subit l'attaque elle écrit dans ses fichiers de logs en fonction du type d’attaque détecté. Dans notre cas les fichiers concernés sont : /var/log/auth.log et /var/log/ufw.log, le forwarder configurer envoit ces logs a splunk installé sur la Machine Hote Ubuntu via le port d'ecoute configuré 9998, et  splunk détecte l'attaque a travers les logs recus et alertes configurées d’une éventuelle attaque.

*************************PRESENTATION ET EXPLICATION DES RESSOURCES SPLUNK :*************************** 

Afin d'assurer la bonne collecte et le traitement des logs par Splunk deux outils doivent etre installés et configurer prealablement dans les differents systemes des acteurs intervenants dans ce flux a savoir : 
  1- Sur le système serveur - Splunk enterprise : le cerveau des operations c'est une plateforme logicielle puissante utilisée pour collecter, analyser et indexer les données générées par les systèmes informatiques, applications, réseaux et appareils de sécurité (comme les logs, les métriques ou les événements). Celui-ci est installé sur la Machine Ubuntu Hote qui joue le role de serveur SIEM, il recoit tous les logs des differents systemes affiliés et les centralise en un seul point afin d'y effectuer des traitements personnalisés.
  
  2- Sur les systèmes a surveiller - Splunk Forwarder : Les forwarders sont des agents installés sur les systèmes à surveiller. Leur rôle est de transférer en temps réel les logs sélectionnés des systèmes voulus. Deux commandes entrent en jeu ici : splunk add forwarder-server (pour définir vers quel serveur SIEM les logs doivent être transférés) et splunk add monitor (pour définir quel dossier de logs doit être surveillé et dont le contenu sera envoyé au serveur SIEM distant).

*************************MISE EN OEUVRE ET DÉTECTION :**************************************************

Nous sommes partis sur une architecture simple, trois acteusrs ici comme enumerés plus haut.
Mais avant tout il faut bien comprendre les actions a mener sur chacun d'eux pour assurer un bon fonctionnement de notre architecture. Nous allons donc lister chacun de ces acteurs ensuite présenter leurs roles et les actions et commandes a effectuer :

  1- La Machine Hote : Serveur SIEM
    C'est sur celle ci qu'est installé le logiciel Splunk chargé de la collecte, analyse et indexiation des données recues. L'installation du logiciel SPlunk differe suivant le systeme d'exploitation utilisé, dans notre cas de figure le logiciel est executé sur une distribution Linux a savoir : Ubuntu 24.04.5 LTS. Les etapes qui suivent montrent comment l'installer : 
 
  -----Télécharger et installer le paquet-----
    
  a) Crée un compte gratuit sur splunk.com, puis va sur la page de téléchargement de Splunk Enterprise.
  b) Choisis Linux → .deb.
  c) Clique sur Download.
  d) Apres son telechargement rend toi sur la page contenant le paquet .deb telechargé, ouvre l'invite de commande sur le dossier puis saisi la commande : sudo -i dpkg <nom_du_fichier>.deb
  
  -----Installation de splunk------
  
  a) le paquet etant installé rend toi dans le dossier /opt/splunk/bin/ qui contient le fichier executable de splunk.
  b) Saisi la commande sudo ./splunk start : on te demandera de créer un compte administrateur après sa creation clique sur l'url fournit et authentifie toi.
      
  -----Configurations Initiale----------------------------
 
   a) Sur le logiciel Splunk : 
   Apres avoir eu acces au dashboard de splunk il faut premièrement configurer le port d’écoute. c'est ce port qui va se charger de recevoir les données provennant des systemes surveillés, données collectées grace aux differents splunk-forwarder. Apres avoir configuré le port d'écoute il suffit d'attendre la réception des données.
   Étapes : settings > Transmission et réception > Réception de données > Nouveau port de réception.


  b) Sur le système surveillé :
   Il faut premièrement télécharger et installer le splunk-forwarder depuis le site officiel. Puis configurer deux éléments fondamentaux : le monitor et le forward-server.
   Le monitor représente le fichier de logs a surveiller ( le fichier log a surveiller )
   Le forward-server représente le serveur splunk vers lequel les fichiers logs seront envoyés. On peut on configurer plusieurs sur le même système pour des besoins de redondance ou de haute disponibilité.
   
 Etapes et commandes :
  - Définir le serveur de destination : sudo /opt/splunkforwarder/bin/splunk add forward-server <IP_DU_SERVEUR:9997>
    Le terminal doit confirmer que le serveur de destination a bien été ajouté
  - Indiquer le dossier ou le fichier de logs à surveiller : sudo /opt/splunkforwarder/bin/splunk add monitor <chemin vers les logs>
    Le terminal doit confirmer que la surveillance du chemin spécifié est active.

 ------Configuration des alertes----------------------------
 
 La configuration des alertes est la partie la plus importante du traitement car permet d'identifier et d’être notifié lors de l'apparition d'une attaque sur le système surveillé, une mauvaise configuration peut engendrer des conséquences significatives sur celui ci, causées par une remédiation tardive ou dans le pire des cas l’absence même de détection d'une attaque.
Une bonne configuration nécessite de comprendre les parametres a notre disposition et leurs fonctionnement :

  a) Titre : Le nom unique qui identifie l'alerte.   
      Configuration : Utilisez un nom explicite (ex: Alerte - Scan de ports suspect)
      
  b) Description : Explique le contexte ou l'objectif de l'alerte pour les autres analystes.   
      Configuration : Décrivez brièvement la menace ciblée (ex: Détecte un balayage de ports UFW supérieur à 20).
      
  c) Permissions : Définit qui peut voir ou modifier l'alerte.   
      Configuration : Laissez en Privé pour vos tests personnels, ou mettez sur Partagé dans l'app si l'équipe SOC doit y accéder.
      
  d) Type d'alerte (Planifié vs Temps réel) : Détermine quand la requête SPL est exécutée.   
      Configuration : Choississez planifié (exemple: toutes les heures ou toutes les 5 minutes) pour économiser les ressources du serveur. Le Temps réel consomme beaucoup de CPU en permanence.

  c) Expire : La durée pendant laquelle les déclenchements de cette alerte sont conservés en mémoire/historique.   
      Configuration : 24 heures est idéal pour garder une trace quotidienne sans saturer l'espace disque
      
  d) Conditions de déclenchement : La règle mathématique ou logique qui active l'alerte (exemple: Par résultat, c'est-à-dire dès que la recherche retourne au moins 1 ligne).
      Configuration : Laissez sur par résultat dans la mesure ou la requete associée a l'alerte a deja des conditions comme dans notre cas (where ports_distincts > 20)
      
  e) Throttle (Limitation) : Empêche l'envoi d'alertes en double si le même événement se répète sans arrêt pendant un laps de temps.  
      Configuration : Cochez-le et définissez une période (exemple: 1 heure) si vous voulez éviter d'être spammé par la même attaque. 
      
  f) Déclenchement d'actions : L'action automatisée exécutée lorsque l'alerte se déclenche.   
      Configuration : Conservez << Ajouter aux alertes déclenchées >> et définissez la Gravité (ex: Moyenne ou Élevée) selon la criticité de l'attaque détectée. Vous pouvez aussi y ajouter un envoi d'e-mail si besoin.























   





















    
