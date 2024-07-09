# Fail2ban

- [Qu’est ce que Fail2ban ?](#quest-ce-que-fail2ban-)
- [Installation de Fail2ban](#installation-de-fail2ban)
- [Configuration de Fail2ban](#configuration-de-fail2ban-pour-les-services-actifs)
- [Configuration avancée](#configuration-avancée)

## Qu’est ce que Fail2ban ?

![](https://i.ibb.co/NmgL9QN/Fail2ban-screenshot.png)

[Fail2ban](https://github.com/fail2ban/fail2ban) est un logiciel de sécurité destiné à prévenir les attaques par force brute en bloquant temporairement les adresses IP suspectes. Il analyse les journaux des services pour détecter des motifs d’échecs d’authentification répétés et autres comportements suspects, puis utilise des règles iptables pour bannir temporairement ces adresses IP.

### Fonctionnalités principales de Fail2ban

1. **Surveillance des journaux** : Fail2ban surveille les fichiers de journaux de divers services (SSH, FTP, HTTP, etc.) pour détecter des tentatives de connexion échouées répétées
2. **Bannissement temporaire** : lorsque Fail2ban détecte un nombre prédéfini de tentatives échouées depuis une même IP, il bannit cette IP pour une période déterminée
3. **Flexibilité et extensibilité** : Fail2ban est hautement configurable et peut être étendu pour surveiller presque n’importe quel service à travers des fichiers de configuration personnalisés et des filtres regex
4. **Actions de bannissement** : Fail2ban peut utiliser différentes actions de bannissement, telles que la mise à jour des règles iptables, la modification des règles hosts.deny, ou l’envoi de notifications par email

## Installation de Fail2ban

#### Méthode N°1

On installe **Fail2ban** via les paquets officiels :

`sudo apt install fail2ban`

----------

#### Méthode N°2 : via les sources

On installe **Fail2ban** via les sources du dépôt GitHub :

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

Nous allons créer et modifier les fichiers de configuration de Fail2ban, mais comme indiqué dans le fichier `/etc/fail2ban/fail2ban.conf`, ce fichier sera probablement remplacé ou amélioré lors d’une mise à jour. De ce fait, nous allons créé des « prisons » dans les fichiers placés dans le dossier `/etc/fail2ban/jail.d`.

Le fichier `/etc/fail2ban/jail.conf` peut servir de documentation et les paramètres par défaut sont écrits à l’intérieur.

On va créer un filtre pour gérer la configuration par défaut de tous les filtres actifs et à venir :

`sudo nano /etc/fail2ban/jail.d/prisons.conf`

On y ajoute :

```sh
[DEFAULT]
ignoreip = 127.0.0.1 192.168.1.0/24 ip-du-serveur mon-ip
findtime = 10m
bantime = 72h
maxretry = 3
```

* **ignoreip** : liste des adresses IP ignorées
  * **ignoreip** : plages IP ; exemples : `ignoreip = 192.168.1.0/24` ou `ignoreip = 192.168.1.0/24 10.0.0.1 172.16.0.0/16`
* **findtime** : spécifie la fenêtre de temps pendant laquelle Fail2ban recherche des tentatives de connexion répétées
* **bantime** : définit la durée pendant laquelle une adresse IP est bannie après avoir dépassé le nombre de tentatives de connexion échouées
* **maxretry** : définit le nombre maximal de tentatives de connexion échouées autorisées dans la période définie par `findtime` avant que Fail2ban ne déclenche un bannissement

On y ajoute le filtre pour le service *SSH* :

```sh
[sshd]
enabled = true
filter = sshd
port = 22
logpath = /var/log/auth.log
```

* **enabled** : *true* = activé | *false* = désactivé
* **filter** : ensemble de règles et de motifs (patterns) utilisés pour identifier les tentatives de connexion infructueuses ou malveillantes
  * **filter** de **sshd** renverra vers `/etc/fail2ban/filter.d/sshd.conf`
* **port** : spécifie le port du service (si vous avez modifié votre port d’origine, pensez à le faire ici aussi)
* **logpath** : cherche dans le fichier log du service

Si le fichier `/var/log/auth.log` n’existe pas, on vérifie si **rsyslog** est installé et en cours d’exécution :

`sudo systemctl status rsyslog`

Si **rsyslog** n’est pas installé, on l’installe :

```sh
sudo apt update
sudo apt install rsyslog
```

On vérifie la configuration de **rsyslog** :

`sudo nano /etc/rsyslog.conf`

On s’assure que l ligne suivante n’est pas commentée (pas de # au début) :

`auth,authpriv.*                 /var/log/auth.log`

On redémarre **rsyslog** :

`sudo systemctl restart rsyslog`

Debian utilise **systemd** et ses logs peuvent être consultés via **journald**. On peut afficher les logs SSH avec la commande suivante :

`sudo journalctl -u ssh`

ou

`sudo cat /var/log/auth.log`

On redémarre Fail2ban pour appliquer les modifications :

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

Quelques exemples de filtres, à ajouter dans le fichier :

`sudo nano /etc/fail2ban/jail.d/prisons.conf`

On y ajoute :

```sh
[apache-auth]
enabled = true
bantime = 744h
maxretry = 1

[apache-badbots]
enabled = true
bantime = 744h
maxretry = 1

[apache-fakegooglebot]
enabled = true
bantime = 744h
maxretry = 1

[apache-overflows]
enabled = true
bantime = 744h
maxretry = 1
```

- **apache-auth** : détecte les tentatives de connexion échouées à des zones protégées par mot de passe sur un serveur web Apache. Cela inclut les échecs d’authentification HTTP basique ou digest
- **apache-badbots** : détecte les accès par des robots malveillants connus sur un serveur web Apache. Les « mauvais robots »sont des scripts automatisés qui tentent d’exploiter des vulnérabilités ou de surcharger le serveur
- **apache-fakegooglebot** : détecte les faux robots se faisant passer pour les Robots de Google (Googlebot) mais qui ne proviennent pas des plages d’adresses IP légitimes de Google
- **apache-overflows** : détecte les tentatives d’attaque par débordement de tampon (buffer overflow) sur un serveur web Apache. Ces attaques exploitent des failles de sécurité en envoyant des données excessivement longues pour causer des erreurs de dépassement de mémoire

## Configuration avancée

Pour être averti par courriel lorsque Fail2ban banni une IP, on modifie la configuration globale :

`sudo nano /etc/fail2ban/jail.d/prisons.conf`

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

Si votre adresse IP se retrouve dans la liste des IP bannies, pas de panique, nous pouvons la retirer !

On tape :

`sudo fail2ban-client set nom-du-service mon-ip`

On peut aussi bannir une IP manuellement :

`sudo fail2ban-client set nom-du-service banip mon-ip`

Merci au [Wiki officiel d’ubuntu-fr.org](https://doc.ubuntu-fr.org/fail2ban) et au [dépôt GitHub officiel de Fail2ban](https://github.com/Sean-Der/fail2web).

<p align="center">
  [ <a href="README.md">Retour à l’accueil</a> ]
  [ <a href="#fail2ban">Remonter la page</a> ]
</p>
