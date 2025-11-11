---
date: 2025-05-07 00:00:00 +0100
title: "Centreon : installation de votre serveur de supervision sous Linux"
author: Matty
categories: [Centreon]
tags: [Linux, supervision, centreon, installation]
render_with_liquid: false
image:
    path: /images/centreon_linux/installation-de-Centreon-sous-Linux.jpg.jpeg
---

## I. Présentation

Dans ce tutoriel, nous allons découvrir pourquoi la supervision réseau est un pilier essentiel de toute infrastructure informatique, et comment Centreon, un outil open source puissant, peut vous aider à surveiller en temps réel l’état de santé de vos équipements, services et applications.

Aujourd’hui, impossible d’imaginer un DSI sans supervision : que ce soit pour détecter les pannes, anticiper les problèmes de performance ou simplement garantir la disponibilité des services critiques, la supervision joue un rôle clé. Elle permet d’avoir une vision centralisée, d’envoyer des alertes en cas d’incident et d’obtenir des rapports précis pour analyser l’activité du système.

C’est ici qu’intervient Centreon, une solution de supervision open source qui s’est imposée comme une référence dans le domaine. Conçue pour s’adapter à tous types d’environnements, Centreon permet de surveiller aussi bien les réseaux locaux que les infrastructures cloud ou hybrides.

![Image](/images/centreon_linux/Apercu-de-linterface-Centreon-800x413.png.png)

Centreon propose plusieurs éditions adaptées à différents besoins. Dans le cadre de ce tutoriel, nous allons utiliser la version IT 100, idéale pour découvrir l’outil dans un environnement de test ou avec des besoins limités (TPE, PME). En effet, cette édition gratuite permet de superviser jusqu’à 100 hôtes, tout en bénéficiant des principales fonctionnalités de la version commerciale.

Voici quelques exemples de ce que vous pouvez monitorer avec Centreon :

- Surveiller l’utilisation CPU, RAM et disque d’un serveur
- Vérifier si un service comme Apache ou MySQL fonctionne correctement
- Contrôler l’état d’un lien réseau ou d’un VPN
- Être alerté en cas de coupure Internet
- Suivre la température ou l’état matériel d’un équipement réseau

Dans le cadre de ce tutoriel, nous verrons l’installation de Centreon sur Linux. Pour ma part, je vais l’installer sous un serveur AlmaLinux 9, mais vous pouvez tout à fait utiliser Debian 11 ou une autre distribution prise en charge par Centreon.

La liste complète des distributions compatibles est disponible dans la documentation officielle de Centreon. Nous allons ici effectuer une installation manuelle, bien qu’il existe des images prêtes à l’emploi si vous préférez une solution plus rapide à déployer.

> Note : il existe d’autres solutions de supervision, concurrentes de Centreon, et accessibles par l’intermédiaire de versions gratuites. On peut citer, par exemple, Zabbix et Nagios.

---

## II. Installation de Centreon sur Linux

### A. Prérequis

Pour la version IT 100 de Centreon (limitée à 100 hôtes), les besoins en ressources sont modestes. Une machine virtuelle AlmaLinux 9 légère mais bien configurée suffira.

Dimensionnement recommandé (pour ≤ 100 hôtes) :

- CPU : 2 vCPU
- RAM : 2 à 3 Go
- Stockage total : environ 60 à 80 Go

### B. Pré-installation

Après avoir installé votre serveur, mettez à jour votre système :

```bash
dnf update
```

Pendant l’installation, SELinux doit être désactivé. Éditez `/etc/selinux/config` et remplacez `enforcing` par `disabled`, ou exécutez :

```bash
sed -i 's/^SELINUX=.*$/SELINUX=disabled/' /etc/selinux/config
```

Redémarrez le système :

```bash
reboot
```

Après le démarrage, vérifiez l'état de SELinux :

```bash
getenforce
```

Vous devriez obtenir :

```
Disabled
```

Si le pare-feu est actif, il est recommandé de le désactiver temporairement pendant l'installation :

```bash
systemctl stop firewalld
systemctl disable firewalld
```

### C. Installer les dépôts

Installez les plugins DNF, ajoutez EPEL et activez CRB :

```bash
dnf install dnf-plugins-core
dnf install epel-release
dnf config-manager --set-enabled crb
```

Activez PHP 8.2 :

```bash
dnf module reset php
dnf module install php:8.2
```

Pour MySQL 8.0, disponible dans les dépôts officiels, installez d'abord le dépôt Centreon puis mettez à jour :

```bash
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://packages.centreon.com/rpm-standard/24.10/el9/centreon-24.10.repo
dnf clean all --enablerepo=*
dnf update
```

Installation de MySQL Server et des paquets Centreon :

```bash
dnf install -y mysql-server mysql
dnf install -y centreon-mysql centreon
systemctl enable --now mysqld
```

Configurez l'authentification par défaut pour utiliser `mysql_native_password` :

```bash
echo "default-authentication-plugin=mysql_native_password" >> /etc/my.cnf.d/mysql-server.cnf
```

Ajustez la limite de fichiers ouverts (exemple) et rechargez systemd :

```bash
sed -Ei 's/LimitNOFILE\s*=\s*[0-9]+/LimitNOFILE = 32000/' /usr/lib/systemd/system/$mysql_service_name.service
systemctl daemon-reload
systemctl restart mysqld
```

Activez les services nécessaires au démarrage :

```bash
systemctl enable php-fpm httpd centreon cbd centengine gorgoned snmptrapd centreontrapd snmpd
systemctl enable crond
systemctl start crond
systemctl enable mysqld
systemctl restart mysqld
```

### D. Sécurisation de la base de données

Sécurisez l'accès root MySQL :

```bash
mysql_secure_installation
```

Répondez "oui" à toutes les questions sauf à "Disallow root login remotely ?" (selon vos besoins). Définissez un mot de passe pour root ; il vous sera demandé lors de l’installation web.

### E. Installation de Centreon par le Web

Démarrez Apache :

```bash
systemctl start httpd
```

Ouvrez l'assistant web :

http://<IP>/centreon

L'assistant de configuration de Centreon s'affiche. Cliquez sur Next.

L'assistant de configuration de Centreon s'affiche. Cliquez sur Next.

![IMAGE 1](/images/centreon_linux/Centreon-Installation-1.png.png)

Les modules et les prérequis nécessaires sont vérifiés. Ils doivent tous être satisfaits. Cliquez sur Refresh lorsque les actions correctrices nécessaires ont été effectuées.  

> Astuce : en cas de problème, vérifiez que tous les modules PHP requis sont bien installés (php-mysqlnd, php-intl, php-gd, etc.). Un simple oubli peut bloquer la validation.  
Puis cliquez sur Next.

![IMAGE 2](/images/centreon_linux/Centreon-Installation-2.png.png)

Définissez les chemins utilisés par le moteur de supervision. Je vous recommande d'utiliser ceux par défaut.  
Puis cliquez sur Next.

![IMAGE 3](/images/centreon_linux/Centreon-Installation-3.png.png)

Définissez les chemins utilisés par Centreon Broker, le composant chargé de transporter les données de supervision (états des hôtes, services, métriques, etc.) depuis les moteurs de collecte vers la base de données. Il agit comme un multiplexeur, optimisant la gestion et la fiabilité des flux de données. Conserver les chemins par défaut est généralement recommandé, sauf configuration avancée.  
Puis cliquez sur Next.

![IMAGE 4](/images/centreon_linux/Centreon-Installation-4.png.png)

Définissez les informations nécessaires pour la création de l'utilisateur par défaut, à savoir le compte admin. Vous utiliserez ce compte pour vous connecter à Centreon la première fois. Le mot de passe doit être conforme à la politique de sécurité : 12 caractères minimum, lettres minuscules et majuscules, chiffres et caractères spéciaux. Vous pourrez changer cette politique par la suite.  
Ensuite, cliquez sur Next.

![IMAGE 5](/images/centreon_linux/Centreon-Installation-5.png.png)

Fournissez les informations de connexion à l'instance de base de données.  
- Base locale : laissez « Database Host Address » vide (localhost).  
- Root user/password : compte utilisé pour créer les bases (mot de passe défini lors de mysql_secure_installation).  
- Database user name/password : identifiants qui seront créés pour l'utilisation quotidienne (évitez de réutiliser des mots de passe identiques).  
Dès que les informations sont fournies, l'assistant crée les fichiers de configuration et les bases de données.  
Lorsque le processus est terminé, cliquez sur Next.

![IMAGE 6](/images/centreon_linux/Centreon-Installation-6.png.png)

Sélectionnez les modules et widgets disponibles à l'installation.  
Une fois les modules choisis, cliquez sur Next.

![IMAGE 7](/images/centreon_linux/Centreon-Installation-7.png.png)

Lorsque l’installation est terminée, la page finale s’affiche : connectez-vous avec le compte `admin` et le mot de passe que vous avez défini.

![IMAGE 8](/images/centreon_linux/Premiere-connexion-a-Centreon.jpg.jpeg)

---

## III. Initialisation de la supervision

Dans l'interface web : Pollers -> Export Configuration -> Export and Reload. Le message de confirmation "Configuration exported and reloaded" indique que tout est bon.

![IMAGE 9](/images/centreon_linux/Centreon-Supervision-1.png.png)
![IMAGE 10](/images/centreon_linux/Centreon-Supervision-2.png.png)

Côté serveur, redémarrez les processus de collecte :

```bash
systemctl restart cbd centengine
systemctl restart gorgoned
systemctl start snmptrapd centreontrapd snmpd
```

La supervision doit maintenant être opérationnelle.

---

## IV. Ajout de la licence IT 100

Avec cette licence, voici ce que vous pouvez obtenir :

Vous pourrez installer jusqu'à 3 serveurs centraux, et monitorer jusqu'à 100 hôtes.
Vous aurez accès à la fonctionnalité de découverte automatique des hôtes et des services, et à la totalité de la bibliothèque de connecteurs de supervision Centreon.
Votre plateforme Centreon doit être connectée à internet pour que la licence IT-100 puisse fonctionner.

Voyons désormais comment ajouter la licence à Centreon.

Pour demander votre licence, rendez-vous sur le site internet à la page Centreon IT100 et remplissez le formulaire.

Vous recevrez un email contenant votre jeton permettant d'utiliser Centreon IT Edition.

Désormais, afin de lier votre jeton de licence et votre plateforme, il suffit de vous rendre dans : Administration -> Extensions -> Manager.

![IMAGE 11](/images/centreon_linux/Ajouter-la-licence-a-Centreon.jpg.jpeg)

Tous les modules installés sur notre plateforme ont un bouton vert avec une coche blanche dedans. Les modules nécessitant une licence ont un bandeau coloré en bas (rouge si vous n'avez pas de licence valide, vert si vous en avez une).

Désormais, il faut renseigner notre jeton en cliquant sur le bouton « Add Token ».
![IMAGE 12](/images/centreon_linux/Centreon-Licence2-800x331.png.png)

Une fois rendu dessus, renseignez votre jeton reçu par email sous le format suivant LKEY.xxxxxxx.

![IMAGE 13](/images/centreon_linux/Centreon-Licence3.png.png)

Une fois la licence activée, nous pouvons voir que nos licences sont désormais valides et que nous avons un abonnement Centreon IT 100.

![IMAGE 14](/images/centreon_linux/Centreon-Licence-gratuit-IT100.jpg.jpeg)

---

## V. Conclusion

En suivant ces quelques étapes, vous êtes maintenant prêt à monitorer votre infrastructure et à utiliser Centreon de manière efficace. Cet outil saura vous offrir une vision claire, centralisée et réactive de l’état de vos équipements. Mais, cela n'est que le début !

Dans les prochains articles, nous verrons comment sécuriser votre serveur Centreon, superviser des hôtes et des services sur différents systèmes (Linux, Windows, équipements réseau, etc.), configurer des notifications, et affiner votre supervision.

Pour aller plus loin sans plus attendre, vous pouvez consulter la documentation officielle de Centreon.