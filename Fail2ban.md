# Fail2ban

- [Qu’est ce que Fail2ban ?](#quest-ce-que-fail2ban-)
- [Installation de Fail2ban](#installation-de-fail2ban)
- [Configuration de Fail2ban](#configuration-de-fail2ban-pour-les-services-actifs)
- [Configuration avancée](#configuration-avancée)

## Qu’est ce que Fail2ban ?

<p align="center">
	![]()
</p>

![](https://i.ibb.co/NmgL9QN/Fail2ban-screenshot.png)

[Fail2ban](https://github.com/fail2ban/fail2ban) est un logiciel de sécurité destiné à prévenir les attaques par force brute en bloquant temporairement les adresses IP suspectes. Il analyse les journaux des services pour détecter des motifs d’échecs d’authentification répétés et autres comportements suspects, puis utilise des règles iptables pour bannir temporairement ces adresses IP.

### Fonctionnalités principales de Fail2ban

1. **Surveillance des journaux** : Fail2ban surveille les fichiers de journaux de divers services (SSH, FTP, HTTP, etc.) pour détecter des tentatives de connexion échouées répétées
2. **Bannissement temporaire** : lorsque Fail2ban détecte un nombre prédéfini de tentatives échouées depuis une même IP, il bannit cette IP pour une période déterminée
3. **Flexibilité et extensibilité** : Fail2ban est hautement configurable et peut être étendu pour surveiller presque n’importe quel service à travers des fichiers de configuration personnalisés et des filtres regex
4. **Actions de bannissement** : Fail2ban peut utiliser différentes actions de bannissement, telles que la mise à jour des règles iptables, la modification des règles hosts.deny, ou l’envoi de notifications par email

## Installation de Fail2ban

#### Méthode N°1

On installe Fail2ban via les paquets officiels :

`sudo apt install fail2ban`

----------

#### Méthode N°2 : via les sources

On installe Fail2ban via les sources du dépôt GitHub :

```sh
git clone https://github.com/fail2ban/fail2ban.git
cd fail2ban
sudo python setup.py install
```

On vérifie que Fail2ban s’est correctement installé :

`fail2ban-client -h`

### Configuration de Fail2ban pour les services actifs

On démarre le service :

`systemctl start fail2ban`

On active le démarrage automatique :

`systemctl enable fail2ban`

On vérifie que le service fonctionne :

`systemctl status fail2ban`

Nous allons créer et modifier les fichiers de configuration de Fail2ban, mais comme indiqué dans le fichier `/etc/fail2ban/fail2ban.conf`, *It will probably be overwritten or improved in a distribution update.*, donc ce fichier sera probablement remplacé ou amélioré lors d’une mise à jour de la distribution. De ce fait, nous allons créé des « prisons » dans les fichiers placés dans le dossier `/etc/fail2ban/jail.d`.

Le fichier `/etc/fail2ban/jail.conf` peut servir de documentation et les paramètres par défaut sont écrits à l’intérieur.

`sudo nano /etc/fail2ban/jail.d/prisonspersos.conf`

On va créer un filtre pour gérer la configuration globale :

```sh
[DEFAULT]
ignoreip = 127.0.0.1 ip-du-serveur mon-ip
findtime = 10m
bantime = 24h
maxretry = 3
```

* **ignoreip** : liste des adresses ip ignorées
  * Fail2ban prends en charge les plages IP, pour le filtre **ignoreip** : `ignoreip = 192.168.1.0/24` ou `ignoreip = 192.168.1.0/24 10.0.0.1 172.16.0.0/16`
* **findtime** : spécifie la fenêtre de temps pendant laquelle Fail2ban recherche des tentatives de connexion répétées
* **bantime** : définit la durée pendant laquelle une adresse IP est bannie après avoir dépassé le nombre de tentatives de connexion échouées
* **maxretry** : définit le nombre maximal de tentatives de connexion échouées autorisées dans la période définie par `findtime` avant que Fail2ban ne déclenche un bannissement

On va prendre pour exemple le filtre pour le service *SSH*.

`sudo nano /etc/fail2ban/jail.d/prisonspersos.conf`

On y ajoute :

```sh
[sshd]
enabled = true
port = 22
logpath = /var/log/auth.log
maxretry = 3
```

* **enabled** : *true* = activé | *false* = désactivé
* **port** : spécifie le port du service
* **logpath** : cherche dans le fichier log du service
* **maxretry** : définit le nombre maximal de tentatives de connexion échouées autorisées

On redémarre Fail2ban pour activer les modifications :

`sudo systemctl restart fail2ban`

On vérifie que le filtre a été prit en compte :

`sudo fail2ban-client status`

La sortie renverra :

```sh
Status
|- Number of jail: 1
`- Jail list:   sshd
```

Vous allez probablement créer différents filtres, si vous souhaitez en activer / désactiver certains, on peut utiliser :

`sudo fail2ban-client start|stop|status sshd`

Fail2ban est un outil indispensable pour renforcer la sécurité de votre serveur. Facile à configurer et hautement personnalisable, Fail2ban vous offre une tranquillité d’esprit en sécurisant vos applications web, SSH, et bien plus encore. Adoptez Fail2ban pour une défense proactive et efficace contre les cybermenaces.

## Configuration avancée

Si vous souhaitez être averti par courriel lorsque Fail2ban banni une IP, vous pouvez !

On modifie la configuration globale :

`sudo nano /etc/fail2ban/jail.d/prisonspersos.conf`

On y ajoute, dans la partie *[DEFAULT]* :

```sh
action = %(action_mw)s
destemail = fail2ban@monsite.fr
```

Pour envoyer un courriel plus détaillé (whois + logs), on chnage :

`action = %(action_mwl)s`

On redémarre Fail2ban :

`sudo systemctl restart fail2ban`

La configuration actuelle est globale, mais on peut spécifier l’envoie de courriel pour chaque filtre séparément.

Fail2ban gère aussi les expressions régulières (regex), qui sont définies par la directive `failregex`.

**Utilisateur avancé** : pour ceux qui souhaitent créer des filtres personnalisés, ou pour testé un filtre sur un fichier log, on utilisera `fail2ban-regex`.

Fail2ban met à disposition des filtres déjà écrits. Pour tester le filtre qui gére les « mauvais robots » qui se balade sur votre serveur, on peut tester le script :

`fail2ban-regex /var/log/apache2/access.log /etc/fail2ban/filter.d/apache-badbots.conf`

Pour afficher les prisons actives :

`sudo fail2ban-client status`

La sortie renverra :

```sh
Status
|- Number of jail: 1
`- Jail list:   sshd
```

Pour afficher le nombre de tentative, le nombre d’IP bannie ainsi que la liste des IP bannies :

`sudo fail2ban-client status sshd`

La sortie renverra :

```sh
Status for the jail: sshd
|- Filter
|  |- Currently failed: 6
|  |- Total failed:     485
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 70
   |- Total banned:     70
   `- Banned IP list:   37.156.*.* 183.81.*.* 121.233.*.* 45.148.*.* …etc
```

*Note : cette prison est active depuis environ 24 heures, 70 IP ont déjà été bannies.*

Si votre adresse IP se retrouve dans la liste des IP bannies, pas de panique, nous pouvons la retirer !

On tape :

`sudo fail2ban-client set nom-du-service  mon-ip`

On peut aussi bannir une IP manuellement :

`sudo fail2ban-client set nom-du-service banip mon-ip`

Merci au [Wiki officiel d’ubuntu-fr.org](https://doc.ubuntu-fr.org/fail2ban) et au [dépôt GitHub officiel de Fail2ban](https://github.com/Sean-Der/fail2web).

<p align="center">
  [ <a href="README.md">Retour à l’accueil</a> ]
  [ <a href="#fail2ban">Remonter la page</a> ]
</p>
