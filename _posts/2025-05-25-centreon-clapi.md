---
date: 2025-05-25 00:00:00 +0100
title: "Centreon : comment importer des hôtes via un fichier CSV et l'API CLAPI ?"
author: Matty
categories: [Centreon]
tags: [Linux, supervision, centreon, clapi, csv, ]
render_with_liquid: false
image:
    path: /images/centreon_clapi/Centreon-importer-des-hotes-en-masse-avec-CSV.jpg.jpeg
---

## I. Présentation

Dans cet article, nous allons découvrir comment importer efficacement vos hôtes dans Centreon à l'aide d'un fichier CSV. Cette méthode permet de gagner un temps précieux, surtout lorsque vous avez de nombreux équipements à superviser. Plutôt que d'ajouter chaque hôte manuellement via l'interface web, l'import CSV vous permet de centraliser toutes les informations dans un fichier structuré et de les intégrer en une seule opération. Autrement dit, vous allez pouvoir importer des hôtes en masse dans Centreon.

Avant de commencer, voici, pour rappel, nos précédents tutoriels sur Centreon :

- Centreon : installation de votre serveur de supervision sous Linux
- Sécurisation Centreon : protection des utilisateurs, SELinux, pare-feu, etc.
- Sécurisation Centreon : comment configurer l'accès HTTPS pour l'interface Web ?
- Centreon : comment superviser des serveurs Linux avec SNMPv3 ?
- Centreon : comment superviser Windows Server ?

## II. Import des hôtes via un CSV

### A. Préparation du CSV

Afin que l'API Clapi puisse interpréter et utiliser correctement vos données, vous devez préparer un fichier CSV respectant le format suivant :

- **Nom de l'hôte** : identifiant unique de l'hôte à ajouter.
- **Désignation de l'hôte** : nom complet ou alias de l'hôte pour l'affichage.
- **Adresse IP** : l'adresse IP de l'hôte à ajouter.
- **Templates de l'hôte** : liste des templates à associer à l'hôte.
- **Instance** : instance de notre collecteur (ici, utilisez Central comme collecteur par défaut).

Le fichier CSV doit donc être structuré de cette manière :

```
<nom d'hôte>;<désignation de l'hôte>;<IP hôte>;<Templates d'hôte>;<Instance>
```

Dans notre cas, nous allons faire un import de 15 serveurs Linux et de 15 serveurs Windows. Pour séparer plusieurs noms de templates, utilisez le caractère `|`.

Tout d'abord, vous devez créer un fichier CSV qui contiendra les informations de vos hôtes. Notre fichier sera créé dans le répertoire `/root/` et appelé `import.csv`.

Utilisez la commande suivante pour créer et éditer le fichier :

```bash
nano /root/import.csv
```

Voici l'exemple de notre fichier `import.csv` :

```cs
IT-Connect-Linux1;server IT-Connect-Linux1;192.168.10.1;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux2;server IT-Connect-Linux2;192.168.10.2;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux3;server IT-Connect-Linux3;192.168.10.3;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux4;server IT-Connect-Linux4;192.168.10.4;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux5;server IT-Connect-Linux5;192.168.10.5;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux6;server IT-Connect-Linux6;192.168.10.6;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux7;server IT-Connect-Linux7;192.168.10.7;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux8;server IT-Connect-Linux8;192.168.10.8;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux9;server IT-Connect-Linux9;192.168.10.9;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux10;server IT-Connect-Linux10;192.168.10.10;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux11;server IT-Connect-Linux11;192.168.10.11;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux12;server IT-Connect-Linux12;192.168.10.12;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux13;server IT-Connect-Linux13;192.168.10.13;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux14;server IT-Connect-Linux14;192.168.10.14;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Linux15;server IT-Connect-Linux15;192.168.10.15;generic-active-host-custom|OS-Linux-SNMPv3;Central
IT-Connect-Windows1;server IT-Connect-Windows1;192.168.20.1;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows2;server IT-Connect-Windows2;192.168.20.2;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows3;server IT-Connect-Windows3;192.168.20.3;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows4;server IT-Connect-Windows4;192.168.20.4;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows5;server IT-Connect-Windows5;192.168.20.5;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows6;server IT-Connect-Windows6;192.168.20.6;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows7;server IT-Connect-Windows7;192.168.20.7;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows8;server IT-Connect-Windows8;192.168.20.8;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows9;server IT-Connect-Windows9;192.168.20.9;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows10;server IT-Connect-Windows10;192.168.20.10;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows11;server IT-Connect-Windows11;192.168.20.11;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows12;server IT-Connect-Windows12;192.168.20.12;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows13;server IT-Connect-Windows13;192.168.20.13;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows14;server IT-Connect-Windows14;192.168.20.14;generic-active-host-custom|OS-Windows-SNMP;Central
IT-Connect-Windows15;server IT-Connect-Windows15;192.168.20.15;generic-active-host-custom|OS-Windows-SNMP;Central
```

### B. Préparation du script d'importation

Désormais, nous devons créer notre script Bash de manière à importer les hôtes dans Centreon via l'API CLAPI. Dans notre cas, nous avons créé le script dans le répertoire `/root` et avons nommé le fichier `import_hosts.sh`. Créez le fichier du script en exécutant la commande suivante :

```bash
nano /root/import_hosts.sh
```

Désormais, copiez et collez le code suivant dans votre script :

```bash
#!/bin/bash
CLAPI=/usr/share/centreon/bin/centreon
INPUT=/root/import.csv
USER=admin
PASS=votre_mot_de_passe
OLDIFS=$IFS

IFS=$';'
[ ! -f $INPUT ] && { echo "$INPUT file not found"; exit 99; }
while read host lblhost ip template instance
do

# Ajouter l'hôte
$CLAPI -u $USER -p $PASS -o HOST -a ADD -v "$host;$lblhost;$ip;$template;$instance"

# Appliquer les templates
$CLAPI -u $USER -p $PASS -o HOST -a APPLYTPL -v "$host"

done < $INPUT
IFS=$OLDIFS
```

Ce script va :

- Lire chaque ligne du fichier CSV.
- Ajouter l'hôte avec les informations de nom, alias, IP, templates et instance.
- Appliquer les templates à chaque hôte.

Veillez bien à remplacer le `PASS` par votre mot de passe admin Centreon (adaptez éventuellement le nom de l'utilisateur aussi) et de vérifier que le `INPUT` soit l'endroit où votre fichier CSV a été créé.

Une fois le fichier créé, vous devez rendre le script exécutable avec la commande suivante :

```bash
chmod +x /root/import_hosts.sh
```

### C. Exécutez le script

Maintenant que tout est en place, vous pouvez exécuter le script pour importer les hôtes dans Centreon. Dans notre cas, le script est `/root/import_hosts.sh`. Veillez à bien changer le répertoire si celui-ci est différent.

Exécutez la commande suivante pour lancer le script :

```bash
/root/import_hosts.sh
```

Le résultat attendu en ligne de commande est le suivant, celui-ci signifiera qu'aucun des paramètres a mal été interprétée par CLAPI et que vos hôtes ont bien été importés :

```
Return code end :
Return code end :
```

Si le script échoue avec des erreurs "Object not found", cela peut signifier que les templates spécifiés dans le fichier CSV ne sont pas encore présents dans Centreon. Assurez-vous que les templates comme `OS-Linux-SNMPv3`, `generic-active-host-custom`, etc… existent bien avant de tenter l'importation.

Pour afficher vos templates : accédez à **Configuration > Templates** pour vérifier que tous les templates sont bien disponibles.

Désormais, connectez-vous à l'interface WEB de Centreon et rendez-vous dans **Configuration -> Hosts** afin de vérifier si vos hôtes ont bien été importés, le résultat attendu est le suivant :

![Import des hôtes via CSV dans Centreon]( /images/centreon_clapi/Centreon-Import-CSV-800x389.jpg.jpeg)

## III. Conclusion

En suivant ces étapes, vous êtes capable d'automatiser l'importation de vos hôtes dans Centreon grâce à un fichier de données au format CSV. L'API Clapi est très utile comme point d'entrée pour effectuer des actions en masse ou dans le contexte de l'automatisation. Vous pouvez désormais automatiser l'ajout de nombreux hôtes de manière simple et rapide.

Pour aller plus loin dès maintenant, vous pouvez consulter la documentation officielle de Centreon.