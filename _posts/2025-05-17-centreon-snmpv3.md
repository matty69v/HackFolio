---
date: 2025-05-17 00:00:00 +0100
title: "Centreon : comment superviser des serveurs Linux avec SNMPv3 ?"
author: Matty
categories: [Centreon]
tags: [Linux, supervision, centreon, installation, snmpv3]
render_with_liquid: false
image:
    path: /images/centreon_snmpv3/Centreon-Supervision-Linux-avec-SNMPv3.jpg.jpeg
---

## I. Présentation

Comment superviser un serveur Linux à l'aide de Centreon ? Voici la question à laquelle nous allons répondre. En effet, dans cet article, nous allons apprendre à superviser un serveur Linux en utilisant SNMPv3, une version du protocole SNMP beaucoup plus sécurisée que les précédentes. Contrairement à SNMPv2, qui utilise une simple "community string", SNMPv3 offre un chiffrement et une authentification forte, garantissant l'intégrité et la confidentialité des échanges entre Centreon et vos serveurs.

Au programme de cet article :

- Pourquoi utiliser SNMPv3 pour la supervision Linux ?
- Configuration du démon SNMP sur un serveur Linux (Net-SNMP)
- Ajout de l'hôte sous Centreon avec SNMPv3
- Vérification des premiers retours et des métriques supervisées

Ce tutoriel s'adresse aussi bien aux débutants qu'aux admins souhaitant apprendre à superviser leurs serveurs sous Linux. À la fin de cet article, vous serez capable de monitorer vos serveurs Linux grâce à Centreon.

Notre objectif sera de superviser un serveur Linux que l'on nommera **Serveur-IT-Connect** ayant comme adresse IP **10.30.111.20**. Nous allons surveiller l'état général de la machine en vérifiant l'utilisation du CPU, de la RAM, du Swap, etc.

Avant de commencer, voici, pour rappel, nos précédents tutoriels sur Centreon :

- Centreon : installation de votre serveur de supervision sous Linux
- Sécurisation Centreon : protection des utilisateurs, SELinux, pare-feu, etc.
- Sécurisation Centreon : comment configurer l'accès HTTPS pour l'interface Web ?

## II. Supervision des serveurs Linux

### A. Pourquoi utiliser du SNMPv3 pour la supervision Linux ?

Le protocole SNMP (Simple Network Management Protocol) est l'un des protocoles les plus utilisés pour la supervision des équipements réseau et des serveurs. Il permet d'interroger des informations système (CPU, mémoire, disque, etc.) à distance, de manière standardisée.

Il existe plusieurs versions du protocole :

- **SNMPv1** : une version standard et fonctionnelle, mais sans sécurité.
- **SNMPv2c** : la version 2 du SNMP permet une amélioration des performances, mais toujours aucune sécurité (juste une "community string", transmise en clair).
- **SNMPv3** : désormais avec SNMPv3, nous avons une version sécurisée qui introduit :
  - Authentification (auth) : avec MD5 ou SHA
  - Chiffrement (priv) : avec DES ou AES
  - Contrôle d'accès basé sur l'utilisateur (user-based)

SNMPv3 se distingue avant tout par sa capacité à sécuriser les échanges entre Centreon et les serveurs supervisés, grâce à l'authentification et au chiffrement des données. Contrairement aux versions précédentes, il protège efficacement contre les interceptions et les attaques de type spoofing. Ce protocole s'inscrit dans une démarche de conformité aux bonnes pratiques IT et est souvent requis dans les environnements soumis à des normes de sécurité strictes comme ISO 27001. En production, surtout lorsque les serveurs manipulent des données sensibles ou sont exposés sur un réseau non isolé, SNMPv3 doit être systématiquement privilégié.

Notons qu'il existe également des alternatives à SNMP, comme NRPE (Nagios Remote Plugin Executor), qui permet d'exécuter des commandes à distance sur un hôte Linux/Unix supervisé, ou l'agent Centreon (Centreon Monitoring Connector / Centreon Plugin Packs) qui joue le rôle de relais entre les équipements et le serveur de supervision.

### B. Configuration du SNMP sur un serveur Linux (Net-SNMP)

Nous allons maintenant configurer un serveur Linux (par exemple : Debian/Ubuntu/CentOS/RHEL) pour qu'il expose ses informations via SNMPv3. Pour débuter, il suffit d'installer le démon SNMP permettant d'effectuer par la suite notre collecte de métriques.

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install snmp snmpd

# CentOS/RHEL
sudo yum install net-snmp net-snmp-utils

# Rocky/Alma
sudo dnf install net-snmp net-snmp-utils
```

Après avoir installé le démon SNMP (snmpd) sur notre serveur, il est nécessaire de configurer un utilisateur SNMPv3. En effet, contrairement à SNMPv1 et SNMPv2c, SNMPv3 offre une sécurité renforcée grâce à l'authentification et au chiffrement.

Pour cela, nous allons créer un utilisateur en lecture seule avec des mécanismes de sécurité basés sur les algorithmes SHA pour l'authentification et AES pour le chiffrement des données :

```bash
net-snmp-create-v3-user -ro -A "VotreMotDePasse" -a SHA -X " VotreMotDePasse " -x AES centreon
```

Quelques explications sur la syntaxe de cette commande :

- **-ro** : crée un utilisateur en lecture seule (read-only). Il n'aura pas le droit de modifier des données via SNMP.
- **-A "VotreMotDePasse"** : le mot de passe pour l'authentification.
- **-a SHA** : spécifie l'algorithme d'authentification.
- **-X "VotreMotDePasse"** : le mot de passe pour le chiffrement.
- **-x AES** : spécifie l'algorithme de chiffrement. Ici, AES est utilisé.
- **centreon** : nom de l'utilisateur SNMPv3 qui sera créé.

Le résultat attendu est le suivant :

```
adding the following line to /var/lib/net-snmp/snmpd.conf:
createUser centreon SHA "VotreMotDePasse" AES " VotreMotDePasse "

adding the following line to /etc/snmp/snmpd.conf:
rouser centreon
```

Remplacez **VotreMotDePasse** par un mot de passe fort et sécurisé. Il est recommandé d'utiliser des mots de passe différents pour l'authentification et le chiffrement.

Une fois l'utilisateur créé, il est nécessaire de redémarrer le service snmpd pour que les modifications soient prises en compte. Utilisez les commandes suivantes :

```bash
sudo systemctl restart snmpd
sudo systemctl enable snmpd
```

Le port 161/UDP est utilisé par le protocole SNMP. Pour permettre à votre serveur d'échanger correctement les données via SNMP avec Centreon, il est nécessaire d'ouvrir ce port sur votre machine Linux à l'aide des commandes suivantes :

```bash
sudo firewall-cmd --permanent --add-port=161/udp
sudo firewall-cmd --reload
```

Si vous utilisez iptables pour gérer le pare-feu sur votre machine Linux, assurez-vous d'autoriser ce port avec la commande suivante. Pour que la règle soit persistante après un redémarrage, pensez à sauvegarder la configuration, par exemple, avec :

```bash
sudo iptables -A INPUT -p udp --dport 161 -j ACCEPT
sudo service iptables save
```

Le service SNMP est désormais prêt à l'emploi sur notre machine Linux.

### C. Ajout de l'hôte sous Centreon

Afin de pouvoir récolter les différentes métriques de notre serveur Linux, nous devons créer un hôte utilisant la même adresse IP que notre serveur Linux que l'on souhaite superviser ainsi que la Template d'hôtes associés.

Dans un premier temps, nous devons installer la Template de notre hôte afin que le collecteur de Centreon sache quelles sont les données à superviser. Rendez-vous dans : **Configuration -> Monitoring Connector Manager**.

Dans "Keyword", saisissez "Linux" et nous choisissons dans notre cas la template **« Linux SNMPv3 »**.

Il existe également d'autres templates selon les besoins et les environnements à superviser, comme :

- **Linux NRPE** : permet la supervision via l'agent NRPE installé sur le serveur, pour aller chercher des métriques plus spécifiques ou exécuter des commandes personnalisées.
- **Linux SSH** : permet de superviser via des commandes exécutées en SSH, sans passer par SNMP ni agent supplémentaire.
- **Linux SNMP** : même supervision que SNMPv3, mais sans les mécanismes de sécurité avancés (plus simple à déployer, mais non chiffré)
- **Linux Monitoring Centreon Agent** : utilise l'agent Centreon installé directement sur le serveur Linux, offrant une supervision plus détaillée et plus flexible que SNMP, avec la possibilité de superviser des services spécifiques ou des métriques personnalisées

![Linux SNMPv3 Template]( /images/centreon_snmpv3/Supervision-serveur-Linux-avec-Centreon-Etape-6-800x267.png.png )
Cliquez sur la template puis sur le **+** afin d'installer la template sur votre collecteur Centreon, un descriptif apparaîtra sur les services que vous pourrez superviser avec cette template.

La template Linux SNMPv3 permet de superviser les principales métriques système du serveur Linux, telles que l'utilisation CPU, la mémoire, l'espace disque, les interfaces réseau…

![Linux SNMPv3 Services]( /images/centreon_snmpv3/Supervision-serveur-Linux-avec-Centreon-Etape-7.png.png)

Une fois installé, rendez-vous dans **Configuration -> Hosts -> Hosts** afin d'ajouter notre serveur Linux :

![Ajouter un hôte Centreon]( /images/centreon_snmpv3/Centreon-Ajouter-un-hote-Linux.jpg.jpeg)

Une fois rendue sur la page, ajoutons notre serveur Linux et renseignons toutes les informations nécessaires à la supervision de notre hôte.

Il va falloir donner un nom à notre serveur afin qu'il soit reconnaissable sous votre interface, lui attribuer l'adresse IP de la machine supervisée (ici **10.30.111.20**), et éventuellement un alias plus lisible (par exemple, **Serveur-IT-Connect**). Cela permet de l'identifier facilement dans les tableaux de bord Centreon.

Ensuite, sélectionnons la version du protocole SNMP à utiliser. Dans notre cas, il s'agit de **SNMP version 3**.

Il est maintenant nécessaire d'indiquer quel serveur de supervision va interroger notre hôte. Ici, on sélectionne "**Central**", qui représente l'instance principale de Centreon.

On passe ensuite au choix du template, le modèle de supervision préconfiguré. Dans notre exemple, nous sélectionnons le template personnalisé **OS-Linux-SNMPv3-custom**, installé précédemment. Ce template contient les commandes de supervision nécessaires pour récupérer des informations telles que la charge CPU, la mémoire, l'espace disque, etc. Il est important de cocher l'option "**Create Services linked to the Template too**" afin que les services définis dans ce template soient automatiquement créés pour l'hôte (héritage).

Dans Centreon, une **macro** est une variable qui permet de centraliser et personnaliser des paramètres spécifiques à un hôte ou à un service sans modifier les commandes de supervision elles-mêmes.

Autrement dit, au lieu d'inscrire directement les valeurs (comme le nom d'utilisateur SNMP ou les mots de passe) dans chaque commande, on utilise des macros comme `$SNMPV3USERNAME$` ou `$SNMPV3AUTHPASSPHRASE$`. Centreon remplacera automatiquement ces macros par leur valeur réelle au moment de lancer les contrôles.

Cela permet :

- de rendre les templates réutilisables sur plusieurs hôtes,
- de gagner du temps lors de la configuration,
- d'éviter les erreurs en centralisant les informations sensibles à un seul endroit,
- et de mieux sécuriser les données sensibles.

Nos macros agissent comme des "boîtes" où l'on place les informations dont Centreon a besoin pour superviser correctement l'hôte, sans les exposer directement dans les commandes.

Pour permettre l'accès aux données SNMPv3, il faut définir nos macros personnalisées. Ces champs sont essentiels :

- **SNMPV3USERNAME** contient le nom d'utilisateur SNMP défini sur le serveur, ici `centreon`.
- **SNMPV3AUTHPASSPHRASE** est le mot de passe utilisé pour l'authentification.
- **SNMPV3AUTHPROTOCOL** précise l'algorithme utilisé pour l'authentification, ici `SHA`.
- **SNMPV3PRIVPASSPHRASE** correspond au mot de passe pour le chiffrement des données.
- **SNMPV3PRIVPROTOCOL** indique le protocole de chiffrement, ici `AES`.

Toutes ces informations doivent correspondre exactement à ce qui est configuré dans le fichier `/etc/snmp/snmpd.conf` sur le serveur distant. Sinon, la supervision échouera.

![Macros SNMPv3 Centreon]( /images/centreon_snmpv3/Supervision-serveur-Linux-avec-Centreon-Etape-1-800x739.png.png)

Dans la section "**Scheduling options**", on définit la fréquence de supervision de l'hôte.

- **Check Period** : 24x7 signifie que l'hôte est supervisé en continu.
- **Max Check Attempts** : Centreon fera 3 tentatives avant de considérer le serveur comme en erreur.
- **Normal Check Interval** : vérification toutes les 1 minute si tout va bien.
- **Retry Check Interval** : si erreur, nouvelle tentative toutes les 3 minutes.
- **Active Checks** et **Passive Checks** : tous deux activés, permettent une supervision directe ou via un autre outil.

![Options de planification Centreon]( /images/centreon_snmpv3/Supervision-serveur-Linux-avec-Centreon-Etape-2-800x246.png.png)

### D. Exporter la configuration

Après avoir ajouté ou modifié un hôte, il faut exporter la configuration vers le poller pour que les changements soient pris en compte.

Dans le menu "**Pollers**", on clique sur "**Export configuration**", puis sur "**Export & reload**". Cela applique les modifications et recharge la supervision sur toute la plateforme. Attention, sans cette étape, Centreon ne prendra pas en compte les nouveaux paramètres.

![Exporter la configuration Centreon]( /images/centreon_snmpv3/Supervision-serveur-Linux-avec-Centreon-Etape-3-800x394.png.png)

> **Remarque** : il est indispensable d'exporter la configuration après chaque modification (ajout, suppression ou mise à jour d'un hôte, d'un service ou d'un poller). Sans cette étape, les changements ne seront pas pris en compte par le moteur de supervision, ce qui pourrait entraîner un décalage entre ce qui est configuré et ce qui est réellement surveillé.

### E. Aperçu de la supervision de l'hôte Linux

Une fois l'opération réalisée, rendez-vous dans vos services afin d'observer les métriques que vous souhaitez pour votre serveur Linux :

![Services supervisés Centreon]( /images/centreon_snmpv3/Supervision-serveur-Linux-avec-Centreon-Etape-4.png.png)

Voici un exemple des services et des valeurs que vous pouvez obtenir. Nous vérifions les éléments suivants :

- Utilisation du processeur
- Uptime du serveur (durée de fonctionnement depuis le dernier redémarrage)
- Utilisation de la mémoire RAM et de la SWAP
- Synchronisation NTP (date et heure)
- Charge système globale
- Connectivité réseau via un ping

![Détails des services supervisés Centreon]( /images/centreon_snmpv3/Supervision-serveur-Linux-avec-Centreon-Etape-5-800x259.png.png)

## III. Conclusion

En suivant ces étapes, nous avons appris à superviser un serveur Linux avec notre plateforme Centreon et à configurer les paramètres essentiels pour assurer une supervision efficace. Il est important de ne pas oublier d'exporter la configuration après chaque modification pour que les changements soient bien pris en compte par le moteur de supervision.

Dans le prochain article, nous verrons comment superviser un serveur Windows, afin de compléter notre supervision multi-systèmes. Pour aller plus loin dès maintenant, vous pouvez consulter la documentation officielle de Centreon.