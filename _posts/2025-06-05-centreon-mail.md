---
date: 2025-06-05 00:00:00 +0100
title: "Centreon : mise en place des notifications par e-mail pour votre supervision"
author: Matty
categories: [Centreon]
tags: [Linux, supervision, centreon, email, notifications, smtp]
render_with_liquid: false
image:
    path: /images/centreon_email/Centreon-configuration-notifications-par-e-mail.jpg.jpeg
---

## I. Présentation

Dans cet article, nous allons découvrir comment mettre en place les notifications par e-mails pour les services et hôtes supervisés sous Centreon. L'objectif est de vous guider pas à pas pour configurer correctement les alertes, afin que vous soyez informé en temps réel en cas de problème ou d'anomalie sur votre infrastructure. Que vous soyez administrateur système, technicien réseau ou simplement en charge de la supervision, cette fonctionnalité est essentielle pour garantir la disponibilité et la performance de vos équipements.

- Centreon : installation de votre serveur de supervision sous Linux
- Sécurisation Centreon : protection des utilisateurs, SELinux, pare-feu, etc.
- Sécurisation Centreon : comment configurer l'accès HTTPS pour l'interface Web ?
- Centreon : comment superviser des serveurs Linux avec SNMPv3 ?
- Centreon : comment superviser Windows Server ?
- Centreon : comment importer des hôtes via un fichier CSV et l'API CLAPI ?
- Centreon : supervision d'un serveur VMware ESXi avec le connecteur de Centreon

## II. Configurer l'envoi d'e-mails

Pour que Centreon puisse envoyer des e-mails de notification, il est nécessaire de configurer un serveur SMTP. Vous pouvez soit installer un serveur SMTP local (comme Postfix), soit utiliser un serveur SMTP existant (par exemple, celui de votre fournisseur de messagerie).

Sur certaines distributions Linux, Postfix peut être installé par défaut.

**Important :** les commandes de notification sont exécutées par le collecteur supervisant la ressource concernée. Il est donc indispensable de configurer le relais SMTP sur chaque collecteur.

Enfin, nous vous recommandons d'utiliser un compte de messagerie dédié à l'envoi des notifications afin de faciliter la gestion et le suivi des alertes.

### A. Configuration de Postfix

Avant de commencer, nous devons installer les paquets nécessaires :

- **postfix** : le serveur de mail (MTA) qui va envoyer vos e-mails.
- **cyrus-sasl-plain** : un module qui permet à Postfix de s'authentifier auprès d'un serveur SMTP en utilisant le mécanisme SASL (par exemple, si vous envoyez des mails via un relais externe comme Gmail ou un serveur de votre fournisseur).

```bash
dnf install postfix cyrus-sasl-plain
```

Redémarrez Postfix :

```bash
systemctl restart postfix
```

La commande ci-dessous va inscrire Postfix dans le système pour qu'il démarre automatiquement au prochain redémarrage de la machine. Sans ça, après un redémarrage, le service postfix ne serait pas actif sans intervention manuelle.

```bash
systemctl enable postfix
```

Le fichier `main.cf` contient les principales options de configuration de Postfix : domaine utilisé pour les mails, authentification, relais SMTP, restrictions, etc. Nous allons l'éditer.

```bash
nano /etc/postfix/main.cf
```

Il est possible de configurer Postfix pour envoyer des emails de deux manières différentes : avec ou sans authentification, avec ou sans TLS. Découvrez la configuration de ces deux méthodes dans la suite de l'article.

### B. Avec authentification et TLS (recommandé)

Dans cette configuration, Postfix utilise une connexion sécurisée (TLS) pour parler au serveur SMTP (par exemple, Gmail, Outlook, etc.) et s'authentifie avec un login et un mot de passe.

**Avantages :**

- La connexion est chiffrée (sécurisée).
- Le serveur SMTP authentifie le client (login + mot de passe).
- Les e-mails passent sans être bloqués ni considérés comme du spam.

Exemple de paramètres :

```
myhostname = hostname
relayhost = [smtp.isp.com]:port
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
```

- Le paramètre `myhostname` est le nom d'hôte du serveur Centreon, ici la valeur associée est `hostname`.
- Le paramètre `relayhost` correspond au serveur de messagerie du compte qui enverra les e-mails.
- Le paramètre `smtp_sasl_password_maps` sert à déclarer le fichier qui contient l'identifiant et le mot de passe (la création de ce fichier est évoquée dans la suite de ce tutoriel)

Dans notre cas suivant, Centreon utilisera un compte Gmail pour envoyer les notifications :

```
myhostname = centreon-central
relayhost = [smtp.gmail.com]:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
```

### C. Sans authentification et sans TLS (non recommandé)

Dans cette configuration, Postfix envoie les emails sans chiffrement et sans s'identifier. C'est uniquement possible si le serveur SMTP accepte les envois anonymes, par exemple, sur un relais SMTP interne à une entreprise ou un serveur local.

**Avantages :**

- Plus, simple à configurer.
- Pas besoin de login ni de mot de passe.

**Inconvénients :**

- Aucune sécurité (les données circulent en clair sur le réseau).
- Refusé par la plupart des serveurs SMTP publics.
- Risque d'être marqué comme spam ou bloqué.

Exemple de paramètres :

```
myhostname = centreon-central
relayhost = [smtp.gmail.com]:587

smtp_use_tls = no

smtp_sasl_auth_enable = no
```

### D. Configurer les identifiants du compte qui enverra les emails

Revenons sur la méthode basée sur une authentification pour voir comment stocker les identifiants. Créez un fichier appelé `sasl_passwd` qui servira à stocker les identifiants (login/mot de passe) du compte email qui enverra les notifications.

Ce fichier sera utilisé par Postfix pour s'authentifier auprès du serveur SMTP.

```bash
touch /etc/postfix/sasl_passwd
```

Dans le fichier `/etc/postfix/sasl_passwd`, ajoutez cette ligne :

```
[smtp.gmail.com]:587 username@gmail.com:XXXXXXXX
```

ou

```
[smtp.fai.com]:port identifiant:motdepasse
```

**Note :** veillez à remplacer les valeurs par celles correspondantes à votre fournisseur de messagerie ainsi que les bons identifiants. Pour utiliser un compte GMAIL, vous devez configurer un mot de passe d'application (« app password ») suivez la procédure disponible sur cette page.

La commande ci-dessous transforme le fichier texte `sasl_passwd` en un fichier de type "hash" (`sasl_passwd.db`) que Postfix peut lire rapidement. Postfix ne lit pas directement les fichiers texte pour des raisons de performance et de sécurité. C'est ce fichier `.db` qui est utilisé réellement par le service.

```bash
/etc/postfix/sasl_passwd.db
```

Désormais, changez le propriétaire (root) et le groupe (postfix) du fichier `sasl_passwd` ainsi que de son équivalent `.db`. Cela garantit que seuls l'administrateur système et Postfix peuvent accéder à ces fichiers sensibles.

```bash
chown root:postfix /etc/postfix/sasl_passwd*
```

Cette commande définit les permissions sur les fichiers :

- Le propriétaire (root) peut lire et écrire.
- Le groupe (postfix) peut uniquement lire.
- Les autres utilisateurs n'ont aucun accès.

C'est une mesure essentielle pour éviter toute fuite d'identifiants SMTP.

```bash
chmod 640 /etc/postfix/sasl_passwd*
```

Maintenant, rechargez la configuration de Postfix sans redémarrer le service. Elle prend en compte les modifications, notamment les nouveaux paramètres d'authentification SMTP.

```bash
systemctl reload postfix
```

Vérifiez que le service Postfix est bien démarré, vous devriez avoir un résultat semblable à celui-ci :

```bash
systemctl status postfix
```

![Statut du service Postfix]( /images/centreon_email/1.png)
**Debug :** si l'envoie de mail ne se fait pas, vérifiez les logs avec la commande suivante :

```bash
tail -f /var/log/maillog
```

### E. Centreon : notification des hôtes et services

Pour permettre à un contact de recevoir des notifications dans Centreon, il est nécessaire de suivre plusieurs étapes de configuration.

La première étape consiste à accéder à la fiche du contact. Pour cela, connectez-vous à l'interface Centreon, puis rendez-vous dans le menu **Configuration > Utilisateurs > Contacts/Utilisateurs**. Une fois sur cette page, cliquez sur le nom du contact pour lequel vous souhaitez activer les notifications.

Dans la fiche du contact, ouvrez l'onglet intitulé **Informations générales**. Vous y trouverez une section nommée **Notification**. Dans cette section, vous devez régler l'option **Activer les notifications** sur **Oui**. Il est important de noter que si cette option est laissée sur **Défaut**, Centreon cherchera la valeur définie dans le modèle parent le plus proche. Si aucun modèle parent ne spécifie cette valeur, le mode **Défaut** équivaut à **Non**, à moins que le contact ait été spécifiquement configuré pour recevoir des notifications au niveau de l'hôte.

Dans notre cas, nous avons paramétré les options de notification de la manière suivante, pour assurer un bon niveau de supervision tout en évitant les alertes superflues.

**Notifications liées aux hôtes**

Nous allons activer les notifications pour les événements suivants :

- **Down** : si l'hôte est considéré comme hors service.
- **Unreachable** : si l'hôte devient injoignable, ce qui peut indiquer un problème réseau.
- **Recovery** : lorsqu'un hôte revient à un état normal après une panne.

Les autres événements, tels que **Flapping** (état instable) ou **Downtime Scheduled** (période d'indisponibilité planifiée), n'ont pas été cochés. Ainsi, aucune notification ne sera envoyée dans ces situations, ce qui permet de limiter les alertes inutiles.

Pour la période de notification, nous avons choisi **24x7**, ce qui signifie que le contact recevra des alertes à n'importe quel moment de la journée ou de la semaine, sans restriction horaire. Concernant le mode d'envoi, la commande sélectionnée est **host-notify-by-email**, ce qui implique que les notifications d'hôte seront transmises par e-mail.

**Notifications liées aux services**

Pour les services, la configuration est un peu plus précise. Nous allons activer les alertes pour les états suivants :

- **Warning** : le service fonctionne, mais présente une anomalie mineure.
- **Unknown** : l'état du service ne peut pas être déterminé.
- **Critical** : le service rencontre une panne ou un problème majeur.

Les notifications de **Recovery**, de **Flapping** ou de **Downtime Scheduled** ne sont pas activées ici non plus, pour éviter un excès d'e-mails non critiques.

Là encore, la période de notification est **24x7**, garantissant une surveillance continue. La commande utilisée est **service-notify-by-email**, ce qui assurera l'envoi des alertes par courrier électronique.

Ce qui donne la configuration globale suivante :

![Configuration des notifications par e-mail dans Centreon]( /images/centreon_email/2.png)

Voici un exemple d'email que vous recevrez si un de vos services ou hôte est Critical ou Down, par exemple :

![Exemple d'email de notification Centreon]( /images/centreon_email/3.png)

> Il est possible de désactiver les notifications pour un service en parparticulier, vous ne souhaitez pas recevoir de notifications. Pour cela, rendez-vous dans la section de votre Service et dans l'onglet **Notifications** désactivez l'option pour ne pas être alerté.

## III. Conclusion

La mise en place des notifications par e-mail dans Centreon est une étape indispensable pour garantir une supervision proactive et efficace de votre infrastructure. Grâce à une configuration de Postfix et des paramètres de notification dans l'interface Centreon, vous pouvez être alerté immédiatement en cas de dysfonctionnement, ce qui vous permet d'intervenir rapidement pour rétablir la situation.

Que ce soit pour les hôtes ou les services, il est important d'adapter les types de notifications à vos besoins afin d'éviter une surcharge d'alertes. En maîtrisant ces paramètres, vous optimisez la réactivité de votre équipe et la fiabilité de votre système d'information.

Dans un prochain tutoriel, nous verrons comment personnaliser davantage ces notifications, notamment en les redirigeant vers des outils de collaboration comme Microsoft Teams, Slack, ou encore via des API web. Cela vous permettra d'intégrer encore plus efficacement Centreon dans vos processus de gestion des incidents.

Pour aller plus loin dès maintenant, vous pouvez consulter la documentation officielle de Centreon.