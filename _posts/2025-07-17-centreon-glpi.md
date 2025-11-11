---
date: 2025-07-17 00:00:00 +0100
title: "Centreon : automatiser la création de tickets GLPI à partir des alertes de la supervision"
author: Matty
categories: [Centreon]
tags: [Linux, supervision, centreon, glpi, ticket, email]
render_with_liquid: false
image:
    path: /images/centreon_glpi/Centreon-Transformer-Les-alertes-en-ticket-GLPI.jpg.jpeg
---

## I. Présentation

Comment transformer une alerte Centreon en ticket de support dans GLPI ? Voici la problématique à laquelle nous allons répondre !

Nous allons voir en détail comment utiliser le système de notifications Centreon pour envoyer des informations vers GLPI, afin que ces alertes soient automatiquement reprises par le collecteur GLPI et transformées en tickets de gestion d'incidents. Nous aborderons étape par étape la mise en place de cette intégration pour centraliser et automatiser la gestion de vos alertes de supervision.

Grâce à ce mécanisme, chaque alerte critique ou majeure générée par Centreon peut être convertie en ticket dans GLPI, affectée à un groupe ou un technicien spécifique, et suivie selon les procédures de gestion des incidents en place.

Sans oublier nos tutoriels Centreon déjà disponibles, y compris pour la configuration des notifications :

- Centreon : installation de votre serveur de supervision sous Linux
- Sécurisation Centreon : protection des utilisateurs, SELinux, pare-feu, etc.
- Sécurisation Centreon : comment configurer l'accès HTTPS pour l'interface Web ?
- Centreon : comment superviser des serveurs Linux avec SNMPv3 ?
- Centreon : comment superviser Windows Server ?
- Centreon : comment importer des hôtes via un fichier CSV et l'API CLAPI ?
- Centreon : Comment superviser votre firewall pfSense
- Centreon : Comment superviser l'espace disque de vos serveurs Linux et Windows
- Centreon : Mise en place des notifications par e-mail
- Centreon : Supervision d'un serveur VMware ESXi

👉 Cet article a été écrit en collaboration avec Elias Garach-Maluly.

## II. Contexte du tutoriel

Dans le cadre de ce tutoriel, nous utiliserons Centreon 24.10 et GLPI 10.0.18 sur des serveurs Almalinux 9.3. En complément, nous utiliserons un serveur de messagerie Microsoft Exchange 2019 sur un Windows Server 2022. Le serveur Exchange étant dans le domaine ITConnect.fr.

![Architecture de la solution Centreon + GLPI]( /images/centreon_glpi/1.png)

Notre tutoriel pour installer Microsoft Exchange : Tutoriel Microsoft Exchange

**Fonctionnement du processus :** Centreon envoie un mail (centreon-engine@itconnect) en utilisant Postfix vers l'adresse e-mail dédiée à GLPI (glpi@itconnect.fr) en passant par le relais SMTP d'Exchange (port 25). GLPI se connecte au serveur Exchange avec IMAP, récupère les derniers messages et les transforme en ticket. Vous pouvez utiliser autre chose qu'Exchange, mais vous devez adapter la configuration en conséquence.

> **Note :** Il existe une alternative avec Azure Entra si vous n'utilisez plus Exchange : GLPI - OAuth IMAP Entra.

## III. Configuration des services

Nous avons plusieurs éléments à configurer, d'abord dans le Centre d'Administration Exchange (EAC), puis côté GLPI et enfin côté Centreon.

### A. Centre d'Administration Exchange

Pour des raisons de sécurité, il est préférable de créer un compte de service avec sa propre adresse e-mail. Il va de soi de lui donner le moins de droits possibles.

Pour créer une adresse e-mail dédiée, connectez-vous sur :

```
https://mail.domaine.fr/ecp
```

Une fois sur la page d'Administration, vous allez créer une nouvelle adresse email à partir de l'onglet boîtes aux lettres :

![Aperçu du Centre d'Administration Exchange]( /images/centreon_glpi/2.png)

*AI-generated content may be incorrect.*

Un pop-up va apparaître et deux possibilités vont être présentées :

![Aperçu de la création d'une boîte aux lettres dans Exchange]( /images/centreon_glpi/3.png)

*AI-generated content may be incorrect.*

Soit, vous avez préalablement créé le compte dans votre annuaire Active Directory, soit vous pouvez créer un compte à la volée via l'interface. Ici, nous créons le compte glpi@itconnect.fr tel que mentionné précédemment.

Une fois le compte lié/crée, il apparaîtra ici :

![Aperçu du compte créé dans Exchange]( /images/centreon_glpi/4.png)

*AI-generated content may be incorrect.*

Vous pouvez faire une vérification que la boîte mail reçoit bien des courriels en envoyant un mail test.

### B. Côté Centreon

Précédemment, nous avons vu comment mettre en place les notifications sous Centreon à l'aide d'un tutoriel détaillé. Nous vous encourageons à lire ce tutoriel si ce n'est pas déjà fait, sauf si votre Centreon est déjà capable d'envoyer des e-mails !

Il va falloir désormais paramétrer que tous nos hôtes puissent envoyer des notifications par e-mail. Pour ceci, nous allons sélectionner nos hôtes et effectuer un Mass Change (modification en lot) qui permet d'appliquer les mêmes attributs de notifications pour nos hôtes et services.

Dans **Configuration -> Hôtes**, sélectionnez tous vos hôtes puis cliquez sur **Mass Change** :

![Mass Change]( /images/centreon_glpi/5.png)

Une fois dans l'onglet, nous allons dans les paramètres de **Notifications** afin de choisir nos préférences pour nos hôtes. Il suffit de choisir le contact lié qui recevra les e-mails, veillez à bien renseigner l'e-mail qui sera utilisé pour notre utilisateur GLPI.

Pour nos hôtes, nous avons choisi de recevoir une notification lorsque l'hôte était soit Down ou Unreacheable avec une notification toutes les 15 minutes en cas de dysfonctionnement.

![Notifications hôtes]( /images/centreon_glpi/6.png)

Désormais, nous allons effectuer la même opération pour nos services, dans **Configuration -> Services -> Services by host** : sélectionnez tous vos services et effectuez un **Mass Change** :

![Mass Change services]( /images/centreon_glpi/7.png)
Nous retrouvons le même type de paramètres que pour les hôtes. Dans notre cas, nous avons choisi d'activer les notifications pour les services lorsque l'état est sur Warning, Uknown ou Critical.

![Notifications services]( /images/centreon_glpi/8.png)

**Remarque :** En fonction de votre environnement et de la criticité des hôtes ou services, la fréquence d'envoi des notifications peut être ajustée.

### C. Côté GLPI

Afin de récupérer les e-mails, il faut créer un collecteur de mails dans GLPI. Vous pouvez aller dans **Configuration > Collecteurs**. Vous allez cliquer sur « Ajouter » et un menu s'ouvrira :

![Aperçu de la création d'un collecteur de mails dans GLPI]( /images/centreon_glpi/9.png)

*AI-generated content may be incorrect.*

Nous retrouvons plusieurs champs :

- **Nom (Email) :** le nom du récepteur doit être l'adresse e-mail complète.
- **Actif :** champ classique de GLPI, permet d'activer ou de désactiver le récepteur.
- **Serveur :** ce champ doit contenir le FQDN ou l'adresse IP de votre serveur.
- **Options de connexion :** ces différentes listes permettent de définir les paramètres de connexion à votre serveur (IMAP ou POP, SSL, TLS, validation du certificat, etc.).
- **Dossier de réception des courriels (optionnel, souvent INBOX) :** si vous souhaitez récupérer les messages dans un dossier spécifique de la boîte mail, indiquez-le ici.
- **Port (optionnel) :** indiquez ici si votre serveur de messagerie nécessite un port spécifique pour établir la connexion.
- **Chaîne de connexion :** ce champ n'est pas disponible lors de la création du récepteur.
- **Identifiant (Login) :** vous devez entrer ici l'identifiant de la boîte mail. Il s'agit souvent du préfixe de l'adresse e-mail (tout ce qui précède le @), mais il est préférable d'entrer l'adresse complète.
- **Mot de passe :** saisissez le mot de passe associé à la boîte mail concernée par le récepteur.
- **Dossier d'archivage des courriels acceptés (optionnel) :** champ optionnel permettant d'indiquer un dossier d'archivage pour les courriels acceptés par le récepteur.
- **Dossier d'archivage des courriels refusés (optionnel) :** champ optionnel permettant d'indiquer un dossier d'archivage pour les courriels refusés par le récepteur.
- **Taille maximale de chaque fichier importé par le récepteur de courriels :** ce champ permet de modifier la taille des fichiers importés. Il permet également de désactiver l'importation en plaçant le champ sur « Pas d'importation ».
- **Utiliser la date du courriel au lieu de celle de la collecte :** ce champ affecte la date de création des tickets ! Il est important de le prendre en compte, notamment pour la configuration des SLA.
- **Utiliser le champ "Reply-To" comme demandeur (lorsque disponible) :** ce champ permet de définir le demandeur du ticket généré à partir du champ « Répondre à » du courriel.
- **Ajouter les utilisateurs en copie (CC) comme observateurs :** ce champ permet d'ajouter les utilisateurs en copie (adresse e-mail) comme observateurs dans le ticket généré.
- **Ne collecter que les courriels non lus :** ce champ permet de générer des tickets uniquement à partir des courriels non lus.
- **Commentaires :** Champ de texte libre dans lequel vous pouvez entrer du contenu à titre informatif. Il n'influence pas la configuration.

Il s'agit d'informations sur les paramètres disponibles dans la documentation officielle (cette page).

Une fois les champs remplis en fonction de votre configuration, vous aurez une nouvelle ligne qui apparaîtra :

![Aperçu de la création d'un collecteur de mails dans GLPI]( /images/centreon_glpi/10.png)

*AI-generated content may be incorrect.*

Il est maintenant possible de forcer l'exécution afin de tester si les mails sont bien récupérés. Cliquez sur le nom de votre collecteur, puis rendez-vous dans **Actions > Récupérer les courriels maintenant**.

Vous devriez avoir une notification indiquant le nombre de mails récupérés.

![Aperçu de la récupération des mails dans GLPI]( /images/centreon_glpi/11.png)



**Note :** Pour que le mail ne soit pas refusé par le collecteur, il faut créer un compte utilisateur dans GLPI ayant pour email l'adresse source (ici centreon-engine@itconnect).

Une fois que cela est configuré, vous pouvez modifier les paramètres d'action automatique. Les actions automatiques sont des tâches exécutées automatiquement à intervalles réguliers. Elles servent notamment à récupérer automatiquement les mails.

Rendez-vous dans **Configuration > Actions automatiques > mailgate** :

![Aperçu des actions automatiques dans GLPI]( /images/centreon_glpi/12.png)



Ici, vous pourrez modifier :

- La fréquence d'exécution
- Le statut
- Le mode d'exécution
- La plage horaire d'exécution
- Le temps de conservation des journaux d'événements
- Le nombre de courriels à traiter à chaque exécution

Nous recommandons de mettre une fréquence d'exécution de 1 minute. Ainsi, vous aurez toujours les derniers mails de manière (presque) instantanée.

Voici le résultat obtenu sous forme de ticket GLPI :

![Ticket GLPI]( /images/centreon_glpi/13.png)

**Remarque importante sur le fonctionnement de ce processus :**

Centreon ne clôture pas les tickets, il se contente de mettre à jour leur état lorsque le service ou l'hôte repasse à OK. Or, Centreon n'est pas un outil de gestion de tickets : il ne conserve pas l'ID du ticket GLPI ni son statut dans sa base de données. Sans cette correspondance (le « mapping » entre alerte et ticket), Centreon est donc incapable d'indiquer à GLPI : « Le problème est résolu, ferme le ticket #72 automatiquement. » Ainsi, même si Centreon envoie une notification (par exemple, un e-mail) pour prévenir du retour à la normale, la fermeture du ticket dans GLPI nécessite toujours une intervention et une validation humaine.

## IV. Conclusion

En intégrant Centreon à GLPI via un collecteur d'e-mails, vous automatisez efficacement la gestion des alertes de supervision en les transformant directement en tickets d'incidents. Ce mécanisme permet non seulement de centraliser les notifications, mais aussi de fluidifier le traitement des incidents en les assignant automatiquement aux bons interlocuteurs selon vos règles de gestion.

Si vous avez des questions, vous pouvez commenter cet article.