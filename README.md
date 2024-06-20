**Sommaire**

- [Créer un Hidden Service sécurisé avec Tor sur Debian](#créer-un-hidden-service-sécurisé-avec-tor-sur-debian)
- [Liste des logiciels](#liste-des-logiciel)
- Qu’est-ce que Tor et un Hidden Service ?
  - [Quelques règles importantes](#quelques-règles-importantes)
- [Configuration du serveur](#-1)
- [Installation d’un serveur LAMP](#-3)
  - [Apache](#-4)
  - [PHP](#-5)
  - [MariaDB](#-6)
  - [Accéder aux tables SQL](#-7)
  - [Installation et configuration de Tor](#-8)
  - [Configuration du Hidden Service](#-9)
  - [Générer une adresse .onion personnalisée](#-10)
  - [FAQ Debug Tor](#faq-debug-tor)
- [Facultatif](#-12)
  - [Configuration de la langue](#configuration-de-la-langue)
  - [Configuration de la date et heure](#configuration-de-la-date-et-heure)
  - [Rediriger le trafic réseau du serveur vers Tor](#rediriger-le-trafic-réseau-du-serveur-vers-tor)
  - [Réécrire la RAM avant un arrêt / redémarrage du serveur](#réécrire-la-ram-avant-un-arrêt--redémarrage-du-serveur)
  - [On supprime ses traces](#on-supprime-ses-traces)
  - [Alias utiles (*~/.bashrc*)](#alias-utiles-pour-le-serveur)
  - [Désactiver le transfert IP et le multi-hébergement](#désactiver-le-transfert-ip-et-le-multi-hébergement)
  - [Désactiver IPv6](#désactiver-ipv6)
  - [Quelques tutoriels utiles](#quelques-tutoriels-utiles)

# Créer un Hidden Service sécurisé avec Tor sur Debian

## Liste des logiciels

Dans ce tutoriel, nous essayerons de garder les logiciels à jour avec leurs dernières versions. Voici la liste actuelle :

- [Debian](https://www.debian.org/) — [version 12.5, liste des changements](https://www.debian.org/News/2024/20240210)
- [Apache2](https://httpd.apache.org/) — [version 2.4.59, liste des changements](https://httpd.apache.org/security/vulnerabilities_24.html#2.4.59)
- [PHP](https://www.php.net/) — [version 8.3.7, liste des changements](https://www.php.net/ChangeLog-8.php#8.3.7)
- [MariaDB](https://mariadb.org/) — [version 10.11.6, liste des changements](https://mariadb.com/kb/en/mariadb-10-11-6-release-notes/)
- [Tor](https://www.torproject.org/) — [version 0.4.8.12, liste des changements](https://gitlab.torproject.org/tpo/core/tor/-/commits/tor-0.4.8.12)

*Dernière mise à jour le 7 juin 2024*

Nous allons configurer notre serveur, qui sera basé sur Apache2, PHP et SQL. Ce type de serveur est plus communément appelé **LAMP**. LAMP est un acronyme désignant un ensemble de logiciels libres permettant de construire des serveurs de sites web. L'acronyme original se réfère aux logiciels suivants :

- « **L**inux », le système d'exploitation ( GNU/Linux ) ;
- « **A**pache », le serveur Web ;
- « **M**ySQL ou **M**ariaDB », le serveur de base de données ;
- À l'origine, « **P**HP », « **P**erl » ou « **P**ython », les langages de script.

Bien sûr, nous pourrions utiliser une [image Docker LAMP](https://hub.docker.com/r/mattrayner/lamp), mais le but est de se familiariser avec l'environnement Linux. Aussi, il existe d'autres logiciels pour le serveur web, comme [nginx](https://nginx.org/) ou [Caddy](https://caddyserver.com/). Cependant, j'ai choisi d'utiliser Apache2, mais vous êtes libre de changer.

## ![Qu’est-ce que Tor et un Hidden Service ?](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Qu%27est-ce%20que%20Tor%20et%20un%20Hidden%20Service%20?)

Tor (acronyme de “ **The Onion Router** ”) est un réseau de communication anonyme. Il permet aux utilisateurs de naviguer sur Internet de manière anonyme en dirigeant le trafic à travers une série de serveurs (appelés nœuds) gérés par des bénévoles. Chaque **nœuds** ne connaît que les informations du nœud précédent et du nœud suivant, ce qui rend difficile pour quelqu’un de surveiller le chemin complet des données. Tor utilise un système de couches de chiffrage, d’où le terme « onion » (oignon) qui fait référence aux multiples couches de protection.

Un Hidden Service (ou service caché) est un service accessible uniquement via le réseau Tor. Ces services utilisent des adresses en « .onion » et permettent aux sites web, aux forums, aux messageries instantanées et à d’autres types de services de fonctionner de manière anonyme. Les utilisateurs peuvent accéder à ces services sans connaître l’emplacement physique du serveur, et le serveur ne connaît pas l’adresse IP des utilisateurs. Cela garantit une confidentialité et une sécurité accrues pour les deux parties. Les Hidden Services sont souvent utilisés pour protéger la vie privée des utilisateurs, mais ils peuvent aussi être utilisés à des fins malveillantes en raison de l’anonymat qu’ils offrent.

> Cartographie des utilisateurs Tor, par pays.

![](https://i.ibb.co/fY6VCzk/Geographies-of-Tor.png)

*[Statistiques officielles du projet Tor](https://metrics.torproject.org/)*

Tor est utilisé pour se protéger contre une certaine forme de surveillance sur Internet, connue sous le nom d’analyse de trafic. Cette analyse est utilisée pour déterminer qui communique avec qui sur un réseau public. Connaître la source et la destination de votre trafic peut permettre à des personnes de traquer votre comportement et vos intérêts. Cartogramme de l’usage de Tor à l’international.

Tor est aussi un outil de contournement de la censure sur Internet. Il permet aux personnes l’utilisant d’accéder à des sites, contenus ou services bloqués dans certaines zones du monde.

Tor fait circuler le trafic de ses utilisateurs et utilisatrices via une série de relais. Ce procédé permet de ne pas être tracé par les sites web consultés, d’accéder à des services, contenus ou sites bloqués par un FAI. Il est aussi possible pour chaque utilisateur de publier des contenus via les services *onion* de Tor, sans révéler la position de ces services.

Ces avantages peuvent être utiles pour chaque personne qui souhaite maîtriser ses traces laissées en ligne. Ils sont notamment mis en œuvre dans les échanges entre lanceurs d’alerte, journalistes, avocats, dissidents politiques, organisations non gouvernementales, pour échanger en maîtrisant la sécurité de leurs données, de leur connexion, de leurs destinataires et de leur position.

Pour faire simple, un *Hidden Service* va vous permettre d’avoir un serveur Jabber, SSH, d’héberger un site, etc. le tout en masquant complètement l’adresse IP et les informations du VPS / serveur aux utilisateurs, bots, aux gouvernements, etc.

Plusieurs choses :

- les utilisateurs Chinois bloqués par le [Grand Firewall de Chine](https://fr.wikipedia.org/wiki/Grand_Firewall_de_Chine) peuvent accéder à l’internet sans censure
- les utilisateurs Iraniens / Turques, etc. bloqués par leur gouvernement peuvent accéder à l’internet sans censure
- être anonyme à 99.99% pendant la navigation sur internet
- accéder aux sites cachés via les adresses en .onion
- discuter sur des messageries utilisant le proxy Tor
- etc.

Malheureusement, qui dit anonymisation des utilisateurs, dit criminalité en tout genre, vente de drogue, d’armes, trafic d’être humain, de fausse monnaie, etc. et j’en passe. Il a des côtés positifs et des côtés négatifs. Que ça soit dans la vraie vie ou sur internet, on sera toujours embêté par les vilains… Le but de ce tutoriel n’est pas de vous montrer comment acheter de la drogue mais d’apprendre le fonctionnement d’un serveur web utilisant le service de cryptage de Tor.

## Quelques règles importantes

*Ce tutoriel est un exemple, en aucun cas on pousse à faire des choses illégales. Vous pouvez adapter en fonction de vos besoins.*

- Toujours se connecter à vos services (SSH, sFTP, FTP, etc) via un proxy SOCK ⁵ de Tor
- Ne JAMAIS installer de logiciel / script (PHP, Python, Bash, etc.), dont vous n’êtes pas certain de la source
- Ne JAMAIS exécuter de logiciel / script / commande dont vous n’êtes pas certain de la source
- Ne JAMAIS réaliser des tâches dont vous n’êtes pas certain de la source
- Démarrer votre installation / configuration avec un VPS / serveur dont vous êtes certain de la source
- Démarrer votre installation / configuration sur un système d’exploitation fraîchement installé
- Utiliser un courriel anonyme pour le service sur lequel vous allez louer votre VPS / serveur
- *[liste de quelques services de courriels sécurisés](https://www.privacytools.io/privacy-email)*
- Payer votre VPS / serveur en [cryptomonnaie](https://fr.wikipedia.org/wiki/Cryptomonnaie) (Bitcoin, Monero, etc.)
- Ne JAMAIS fournir votre identité lorsque vous payez via Cryptomonnaie
- **Ne JAMAIS faire fonctionner un relais Tor sur le VPS / serveur, car ces adresses IP sont rendues publiques**
- Ne JAMAIS envoyer de courriel via le VPS / serveur (donc désactiver tous les logiciels / fonctions liées aux courriels)
- Ne JAMAIS autoriser l’envoie de fichier sur le VPS / serveur où va être hébergé votre site
- Ne JAMAIS autoriser l’ajout d’image distante (exemple, avec la balise *img src=""*)
- JavaScript est à BANNIR sur les applications Web que vous allez développer / héberger
- Désactiver toutes les fonctions Apache2, nginx, PHP, etc. qui sont susceptibles de renvoyer des erreurs aux visiteurs et peuvent afficher votre adresse IP (une liste non-exhaustive sera fournie)
- Ne pas inclure des fichiers distants via des CDNs, par exemple : jQuery, Bootstrap, etc.
- Effectuer un audit de vos applications Web pour éviter toute faille potentielle
- Effectuer un audit de vos scripts pour éviter toute faille potentielle
- Garder votre VPS / serveur à jour (amélioration des logiciels, correction de faille de sécurité, etc.)

*Si une règle vous semble incorrecte, si vous souhaitez proposer une amélioration, ajouter un oubli, n’hésitez pas à le proposer.*

**VPN > Tor, Tor > VPN > Proxy, Proxy > VPN > Tor, VPN > Tor ? Quelle est la meilleure solution ?**

Il n’y a pas de réponse universelle à cette question, car cela dépend de votre situation individuelle et de ce que vous cherchez à protéger.

Dans l’ensemble, l’utilisation de Tor est considérée comme la meilleure solution pour protéger son anonymat en ligne, car elle vous permet de masquer votre adresse IP et de rendre vos communications difficiles à tracer. Cependant, il est important de noter que l’utilisation de Tor ne garantit pas une protection à 100% et qu’il est possible que certaines attaques puissent encore vous identifier.

L’utilisation d’un VPN peut également être utile pour protéger votre anonymat en ligne, car elle permet de masquer votre adresse IP et de chiffrer votre trafic internet. Cependant, il est important de choisir un fournisseur VPN fiable qui ne conserve pas de journaux de connexion et qui utilise des protocoles de chiffrement solides pour éviter toute compromission.

L’utilisation d’un proxy peut également être utile pour masquer votre adresse IP, mais il est important de choisir un proxy fiable et sécurisé, car certains proxies peuvent être compromis ou surveillés par des tiers malveillants.

Dans l’ensemble, l’utilisation de Tor est considérée comme la solution la plus sûre et la plus robuste pour protéger son anonymat en ligne, mais l’utilisation d’un VPN ou d’un proxy peut également être utile en complément. Si vous choisissez d’utiliser un VPN ou un proxy en plus de Tor, il est important de comprendre les risques et les limitations de chaque solution, et de prendre les mesures de sécurité nécessaires pour éviter toute compromission.

**Ma recommandation :** VPN + Tor

**VPNs recommandés :** [Mullvad](https://mullvad.net/) ou [ProtonVPN](https://protonvpn.com/)

## ![Configuration du serveur](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Configuration%20du%20serveur)

On se connecte au serveur via SSH et on change directement le mot de passe root :

```sh
sudo su
passwd
```

----------

> **Solution n°1 : Clés SSH**

On génère les clés SSH avec l’algorithme de chiffrement [ed25519](https://fr.wikipedia.org/wiki/EdDSA) :

`ssh-keygen -t ed25519`

*Pourquoi ED25519 ? En résumé, ED25519 et RSA sont tous deux des algorithmes cryptographiques à clé publique populaires utilisés pour la transmission sécurisée de données. ED25519 est généralement considéré comme plus sécurisé et efficace que RSA, tandis que RSA offre un niveau de sécurité plus élevé en raison de la taille de sa clé plus grande. Le choix entre ces deux algorithmes dépend de l’application spécifique et du niveau de sécurité et d’efficacité requis.*

On laisse l’emplacement par défaut :

```sh
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/salameche/.ssh/id_ed25519):
```

On y met un mot de passe, pas obligatoire, mais fortement recommandé :

```sh
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

Nos clés sont générées :

```sh
Your identification has been saved in /home/salameche/.ssh/id_ed25519.
Your public key has been saved in /home/salameche/.ssh/id_ed25519.pub.
The key fingerprint is:
SHA256:4Yjb63lZzyRw+ADKaZ6nwZDA7jBrtorVR4mkgXRGWN0 salameche@my-pc
The key's randomart image is:
+--[ED25519 256]--+
|..+=. .          |
|o+o  ..E         |
|o..o.o ...       |
|o.o+=o ++..      |
|oo.=o.+ S=       |
|.+ .=+.   + .    |
|o o o+o  o =     |
|.o  .. oo   o    |
|+    .+.         |
+----[SHA256]-----
```

Une fois vos clés générer sur votre PC, on les place sur le serveur distant :

`ssh-copy-id <username>@<hostname>`

Remplacez **username** par le nom d’utilisateur et **hostname** par l’adresse IP ou le nom d’hôte du serveur.

Entrez le mot de passe de l’utilisateur.

C’est fait, la clé publique a bien été copiée dans le fichier **~/.ssh/authorized_keys** du serveur.

*Pour se connecter :*

Connectez-vous normalement en SSH via terminal par exemple :

`ssh <username>@<hostname>`

Entrez votre mot de passe de clé générée ci-dessus, voilà, vous êtes maintenant connecté à votre serveur !

Source : [LeCrabe.info](https://lecrabeinfo.net/se-connecter-en-ssh-par-echange-de-cles-ssh.html#etape-1-generer-des-cles-ssh)

*Vous y trouverez comment se connecter avec des clés SSH pour Windows, Mac et Linux*

----------

> **Solution n°2 : Mot de passe**

Je vous conseille de lire le début du [tutoriel pour apprendre à vous connecter à SSH]([https://mondedie.fr/d/11708).

On met à jour notre fichier *sources.list* :

`nano --backup /etc/apt/sources.list`

On change pour :

```sh
deb https://deb.debian.org/debian bookworm main
deb-src https://deb.debian.org/debian bookworm main

deb https://deb.debian.org/debian-security/ bookworm-security main
deb-src https://deb.debian.org/debian-security/ bookworm-security main

deb https://deb.debian.org/debian bookworm-updates main
deb-src https://deb.debian.org/debian bookworm-updates main
```

On met à jour les paquets et on installe [nano](https://doc.ubuntu-fr.org/nano) (éditeur de texte) et [sudo](https://doc.ubuntu-fr.org/sudo) (permet à un utilisateur normal d’exécuter des commandes en tant que super-utilisateur (ou « root »)).

*Ces deux logiciels ne sont pas installés par défaut sur certains VPS, cela dépend du fournisseur et de la distribution, donc pour éviter tout problème, on installe, s’ils sont déjà présents sur le serveur, ça ne changera rien.*

`apt update && apt install apt-transport-https lsb-release ca-certificates nano sudo wget`

On re-met à jour le serveur :

`apt update && apt upgrade -y`

On crée notre utilisateur principal :

`adduser salameche`

Pour plus de clarté dans ce tutoriel, j’utiliserai comme nom d’utilisateur : **salameche**

Une fois notre utilisateur créé, on l’ajoute au groupe « *sudo* », cela permettra d’exécuter les commandes « *root* », sans être « *root* », cela améliore grandement la sécurité et évitera de faire des bêtises :

`adduser salameche sudo`

Cela renverra :

```sh
Adding user `salameche' to group `sudo' ...
Adding user salameche to group sudo
Done.
```

Une fois l’utilisateur ajouté au groupe « *sudo* », on se connecte sur notre compte utilisateur « *salameche* » :

`su salameche`

On paramètre / sécurise SSH :

`sudo nano /etc/ssh/sshd_config`

On change le port SSH :

`Port _PORT_`

C’est une forme de sécurité simple, mais étonnamment efficace.

Les serveurs utilisent généralement le port 22 pour se connecter à SSH, donc il est moins susceptible d’être trouvé par des robots qui analysent les adresses IP à la recherche de mot de passe faible sur les comptes par défaut. Si vous numérisez tout le réseau, vous ne pouvez pas vous permettre de vérifier tous les ports possibles (65 535 ports disponibles) pour trouver le serveur SSH.

Cependant, si quelqu’un vous ciblera activement, cela ne fournit aucun bénéfice, car une simple analyse *nmap* unique révèlera le port sur lequel **SSH** fonctionne réellement (on utilisera [PortSentry](PortSentry.md) pour bloquer ces attaques, voir plus bas).

- **Le port doit être compris entre 0-65535**
- **Le port utiliser ne doit pas être déjà utilisé par une application**

On désactive la connexion root en SSH :

`PermitRootLogin no`

Nous n’utiliserons pas le protocole [FTP](https://fr.wikipedia.org/wiki/File_Transfer_Protocol), cela pour des raisons évidentes de sécurités, mais [sFTP](https://fr.wikipedia.org/wiki/SSH_File_Transfer_Protocol).

On commente et / ou supprime cette ligne :

`#Subsystem sftp /usr/lib/openssh/sftp-server`

On ajoute en dessous :

`Subsystem sftp internal-sftp`

**internal-sftp** est recommandé pour les scénarios où la sécurité et l’isolation sont importantes, comme le chrooting des utilisateurs vers un répertoire spécifique.*

À la fin de la page, on ajoute : (on vérifie que les paramètres ne sont pas présents pour éviter les doublons)

```sh
UseDNS no
UsePAM yes

DebianBanner no

AllowUsers salameche
```

- **UseDNS** : par défaut, le serveur cherche à établir la résolution DNS inverse depuis votre IP. Cette requête peut être assez longue, c’est pour cela que nous désactivons cette fonctionnalité, plutôt inutile
- **UsePAM** : PAM doit être désactivé si vous utilisez des clés d’authentifications, ce qui n’est pas notre cas, donc il doit être activé
- **DebianBanner** : permet d’éviter que le serveur SSH n’affiche la distribution Linux Ubuntu ou Debian
- **AllowUsers** : ajoute les utilisateurs autorisés à se connecter à SSH, pour notre cas, on ajoutera simplement « *salameche* »

On quitte et on redémarre SSH :

`sudo /etc/init.d/ssh restart`

On installe / désinstalle quelques logiciels pour la pratique et la sécurité, on recharge le cache de recherche et on met les liens symboliques à jour :

`sudo apt install ca-certificates curl gnupg locate && sudo apt purge ntp rsyslog exim* postfix* sendmail* samba* && sudo updatedb`

***Ajouts*** :

- [ca-certificates](https://packages.debian.org/fr/bookworm/ca-certificates) : ce paquet inclut les autorités de certifications livrées avec les navigateurs Mozilla afin de permettre aux applications basées sur SSL de vérifier l’authenticité des connexions SSL.
- [curl](https://packages.debian.org/fr/bookworm/curl) : curl est un outil en ligne de commande pour transférer des données avec une syntaxe URL qui prend en charge DICT, FILE, FTP, FTPS, GOPHER, HTTP, HTTPS, IMAP, IMAPS, LDAP, LDAPS, POP3, POP3S, RTMP, RTSP, SCP, SFTP, SMTP, SMTPS, TELNET et TFTP.
- [gnupg](https://packages.debian.org/fr/bookworm/gnupg) : GnuPG est l’outil GNU pour sécuriser les communications et le stockage de données. Il peut être utilisé pour chiffrer des données et créer des signatures numériques. Il inclut un mécanisme perfectionné de gestion de clés et est compatible avec la proposition de standard OpenPGP pour Internet tel que décrit dans la RFC 4880.
- [locate](https://packages.debian.org/fr/bookworm/locate) : updatedb génère un index de fichiers et répertoires. GNU locate peut être utilisé pour effectuer des requêtes rapides sur cet index.

***Suppressions*** :

- [ntp](https://packages.debian.org/fr/bookworm/ntp) : il s’agit d’un package de transition factice pour passer à NTPsec. Il peut être retiré en toute sécurité.
- [rsyslog](https://packages.debian.org/fr/bookworm/rsyslog) : rsyslog est une implémentation à unités d’exécution multiples de syslogd (un outil système qui fournit une journalisation de message).
- [exim4](https://packages.debian.org/fr/bookworm/exim4) : exim (version 4) est un agent de transport de courrier. Exim4 est le métapaquet sur lequel dépendent les composants essentiels d’une installation de base d’exim4.
- [postfix](https://packages.debian.org/fr/bookworm/postfix) : postfix est l’agent de transport de courriel de Wietse Venema qui a commencé son existence comme une alternative au très utilisé programme Sendmail. Postfix vise à être rapide, facile à administrer et sécuritaire, tout en restant assez compatible avec Sendmail pour ne pas frustrer ses utilisateurs. Ainsi, l’externe ressemble à Sendmail, alors que la structure interne est complètement différente.
- [sendmail](https://packages.debian.org/fr/bookworm/sendmail) : sendmail est un agent de transmission de courriels (MTA) alternatif pour Debian. Il est adapté pour le traitement des configurations de messagerie sophistiquées, quoique cela signifie aussi que sa configuration peut être complexe.
- [samba](https://packages.debian.org/fr/bookworm/samba) : samba est une implémentation du protocole SMB/CIFS pour les systèmes Unix, offrant la prise en charge du partage de fichiers et d’imprimantes multiplateforme avec Microsoft Windows, OS X et d’autres systèmes Unix. Samba peut également fonctionner comme un contrôleur de domaine de style Active Directory ou NT4 et peut s’intégrer aux domaines Active Directory ou aux domaines NT4 en tant que serveur membre.

## ![Le serveur est configuré](https://i.imgur.com/CyiIlQx.png)

## ![Installation d’un serveur LAMP](https://i.imgur.com/lcwfXBO.png)

## ![Installation et configuration d’Apache2](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Installation%20et%20configuration%20d’Apache2)

On installe Apache2 :

Avant toutes choses, on désinstalle Apache2 s’il est déjà installé, pour éviter tout conflit.

`sudo apt purge apache*`

Puis on réinstalle Apache2 :

`sudo apt install apache2 apache2-utils`

On masque quelques informations d’Apache2 :

`sudo nano /etc/apache2/apache2.conf`

On vérifie que ces lignes correspondent à :

```sh
<Directory />
	#Options FollowSymLinks
	AllowOverride None
	Require all denied
</Directory>

<Directory /usr/share>
	AllowOverride None
	#Require all granted
	Require all denied
</Directory>

<Directory /var/www/>
	Options FollowSymLinks
	AllowOverride None
	Require all granted
</Directory>
```

On ajoute ces 3 lignes à la fin de la page :

```sh
ServerSignature Off
ServerTokens Prod
TraceEnable Off
```

On sauvegarde et on ferme le fichier.

- [ServerSignature](https://httpd.apache.org/docs/2.4/fr/mod/core.html#ServerSignature)
- [ServerTokens](https://httpd.apache.org/docs/2.4/fr/mod/core.html#ServerTokens)
- [TraceEnable](https://httpd.apache.org/docs/2.4/fr/mod/core.html#TraceEnable)

Les fichiers :

- **/etc/apache2/apache2.conf** : le fichier de configuration principal d’Apache.
- **/etc/apache2/ports.conf** : configuration des ports.
- **/etc/apache2/sites-available/** : répertoire pour les fichiers de configuration des sites disponibles.
- **/etc/apache2/sites-enabled/** : répertoire pour les fichiers de configuration des sites activés.

On supprime le dossier **/var/www/html** :

`sudo rm -rf /var/www/html`

On applique les droits **Apache2** sur le dossier de notre futur Hidden Service :

```sh
sudo chown -R www-data:www-data /var/www
sudo chmod 775 /var/www
sudo usermod -aG www-data salameche
```

**Le VirtualHost**

> Un **VirtualHost** est une configuration qui permet à un serveur unique de répondre à des requêtes pour plusieurs noms de domaine. Cette fonctionnalité est particulièrement utile pour les serveurs web qui hébergent plusieurs sites web, permettant ainsi à chaque site d’avoir ses propres configurations et paramètres, tout en partageant les mêmes ressources serveur.

On désactive le [mod_autoindex](https://httpd.apache.org/docs/2.4/fr/mod/mod_autoindex.html) et le [mod_status](https://httpd.apache.org/docs/2.4/fr/mod/mod_status.html) :

`sudo a2dismod autoindex status`

- **mod_autoindex** : fournit une fonctionnalité de génération de listes de répertoires automatiques lorsque le serveur web reçoit une requête pour un répertoire qui ne contient pas de fichier index (comme index.html, index.php, etc.).

En d’autres termes, lorsque vous accédez à un répertoire sur un serveur web Apache et qu’aucun fichier d’index n’est présent dans ce répertoire, le module **mod_autoindex** génère automatiquement une liste des fichiers et répertoires contenus dans ce répertoire et la renvoie au client (le navigateur web) sous forme de page HTML. Cette page affiche généralement le nom, la taille et la date de modification des fichiers, ainsi que des liens pour naviguer dans les répertoires.

Le module **mod_autoindex** offre également des fonctionnalités de personnalisation permettant de modifier l’apparence et le comportement de ces listes de répertoires, comme la possibilité de masquer certains fichiers, d’ajouter des en-têtes et des pieds de page personnalisés, ou encore de définir des icônes pour différents types de fichiers.

En résumé, le module Apache **mod_autoindex** simplifie la gestion des répertoires sur un serveur web en générant automatiquement des listes de fichiers et de répertoires lorsque nécessaire, offrant ainsi une manière pratique de naviguer dans la structure des fichiers sur un site web.

- **mod_status** : Le module Apache **mod_status** est un module optionnel pour le serveur web Apache qui fournit des informations en temps réel sur la performance et l’utilisation du serveur. Il expose ces informations via une page HTML accessible via une URL spécifique.

Voici quelques-unes des informations que **mod_status** peut fournir :

- Statut du serveur : Il indique si le serveur est en cours d’exécution ou s’il est arrêté.
- Nombre de requêtes en cours : Combien de requêtes sont actuellement en cours de traitement par le serveur.
- Statistiques sur les requêtes : Nombre total de requêtes traitées depuis le démarrage du serveur, ainsi que des statistiques détaillées telles que le nombre de requêtes traitées par seconde.
- Statistiques sur les processus : Nombre de processus Apache en cours d’exécution, leur état (actif, en attente, etc.) et leur utilisation de la mémoire.
- Connexions : Informations sur les connexions actives et les connexions en attente.
- Détails sur les workers : Pour les configurations avec plusieurs workers (travailleurs), **mod_status** peut fournir des détails sur chaque worker, y compris leur état et leur utilisation des ressources.

Le module **mod_status** est souvent utilisé pour surveiller et diagnostiquer la performance du serveur Apache, ainsi que pour détecter tout problème éventuel. Il peut être particulièrement utile pour les administrateurs système chargés de gérer et de surveiller un serveur web Apache en temps réel.

Il convient de noter que, comme tout module Apache, **mod_status** doit être activé et configuré dans le fichier de configuration d’Apache pour être utilisé, et il est généralement recommandé de restreindre l’accès à la page de statut pour des raisons de sécurité.

On active différents modules utiles pour Apache2 :

`sudo a2enmod deflate headers rewrite`

On quitte et on redémarre Apache2 :

`sudo service apache2 restart`

- [Site officiel d’Apache2](https://httpd.apache.org/)
- [Dépôt GitHub officiel](https://github.com/apache/httpd)

## ![Installation et configuration de PHP](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Installation%20et%20configuration%20de%20PHP)

On va installer PHP 8 via le dépôt d’[Ondřej Surý](https://deb.sury.org/) car les dernières versions de PHP ne sont pas disponibles sur Debian.

Qui est **Ondřej Surý** ?

> Ondřej Surý est un développeur Debian depuis les années 2000, et il empaquetee PHP pour Debian depuis PHP 5, ce qui signifie que les paquets officiels dans Debian et Ubuntu sont soit son travail, soit ils sont basés sur son travail. Les paquets PHP de son Ubuntu PPA et Debian DPA correspondent aux paquets officiels de Debian.

- [Site officiel de PHP](https://www.php.net/)
- [Dépôt GitHub officiel](https://github.com/php/php-src)
- [Nouveautés dans PHP 8.3](https://kinsta.com/fr/blog/php-8-3/)

Avant toutes choses, on désinstalle PHP s’il est déjà installé, pour éviter tout conflit.

```sh
sudo systemctl stop php*
sudo apt autoremove --purge php*
sudo a2dismod php5 php7.0 php7.1 php7.2 php7.3 php7.4 php8.0 php8.1 php8.2 php8.3
```

*Des erreurs apparaitront si une version de PHP n’est pas installée.*

On ajoute la clé GPG & le dépôt :

```sh
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://packages.sury.org/php/apt.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/php.gpg
sudo chmod a+r /etc/apt/keyrings/php.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/php.gpg] https://packages.sury.org/php/ \
  $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/php8.list > /dev/null
```

On met à jour les paquets :

`sudo apt update`

On installe PHP et quelques dépendances utiles :

```sh
sudo apt install php8.3 \
php8.3-{bz2,cli,common,curl,intl,mbstring,mysql,opcache,xml,zip}
libapache2-mod-php8.3
```

On active PHP pour Apache2 :

`sudo a2enmod php8.3`

On configure PHP :

`sudo nano /etc/php/8.3/apache2/php.ini`

On remplace et / ou rajoute :

```ini
; https://www.php.net/manual/fr/ini.core.php#ini.short-open-tag
short_open_tag = Off

; https://www.php.net/manual/fr/ini.core.php#ini.open-basedir
open_basedir = /var/www

disable_functions = fonctions précédentes + ,passthru,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,highlight_file,show_source,mail,phpinfo,passthru,eval,proc_get_status,proc_nice,proc_open,proc_terminate,ftp_alloc,ftp_cdup,ftp_chdir,ftp_close,ftp_connect,ftp_delete,ftp_exec,ftp_fget,ftp_fput,ftp_get,ftp_put,ftp_nlist,imap_open

; https://www.php.net/manual/fr/info.configuration.php#ini.max-execution-time
max_execution_time = 60

; https://www.php.net/manual/fr/info.configuration.php#ini.max-input-time
max_input_time = -1

; https://www.php.net/manual/fr/errorfunc.configuration.php#ini.display-errors
display_errors = Off

; https://www.php.net/manual/fr/errorfunc.configuration.php#ini.display-startup-errors
display_startup_errors = Off

; https://www.php.net/manual/fr/errorfunc.configuration.php#ini.log-errors
log_errors = On

; https://www.php.net/manual/fr/ini.core.php#ini.post-max-size
post_max_size = 8M

; https://www.php.net/manual/fr/ini.core.php#ini.upload-max-filesize
upload_max_filesize = 8M

; https://www.php.net/manual/fr/ini.core.php#ini.max-file-uploads
max_file_uploads = 0

; https://www.php.net/manual/fr/ini.core.php#ini.file-uploads
file_uploads = Off

; https://www.php.net/manual/fr/datetime.configuration.php#ini.date.timezone
date.timezone = Australia/Melbourne

; https://www.php.net/manual/fr/ini.core.php#ini.expose-php
expose_php = Off

; https://www.php.net/manual/fr/filesystem.configuration.php#ini.allow-url-fopen
allow_url_fopen = Off

; https://www.php.net/manual/fr/filesystem.configuration.php#ini.allow-url-include
allow_url_include = Off

; https://www.php.net/manual/fr/mysqlnd.config.php#ini.mysqlnd.collect-statistics
mysqlnd.collect_statistics = Off

; https://www.php.net/manual/fr/mysqlnd.config.php#ini.mysqlnd.collect-memory-statistics
mysqlnd.collect_memory_statistics = Off
```

Ces fonctions sont à titre d’information, vous pouvez activer / désactiver celles que vous souhaitez.

On sauvegarde le fichier **php.ini** et on redémarre Apache2 :

`sudo service apache2 restart`

## ![Installation et configuration de MySQL](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Installation%20et%20configuration%20de%20MySQL)

MariaDB est un système de gestion de base de données édité sous licence GPL. Il s’agit d’un embranchement communautaire de MySQL : la gouvernance du projet est assurée par la fondation MariaDB, et sa maintenance par la société Monty Program AB, créateur du projet. Cette gouvernance confère au logiciel l’assurance de rester libre.

L’installation du méta-paquet **default-mysql-server** installera mariadb-server. Si les paquets mysql-server-* ou mysql-server-* sont installés, ils seront supprimés et remplacés par leur équivalent MariaDB. De la même façon, l’installation du méta-paquet default-mysql-client installera mariadb-client-*.

- [Site officiel de MariaDB](https://mariadb.org/)
- [Dépôt GitHub officiel](https://github.com/MariaDB/server)

Avant toutes choses, on désinstalle MariaDB s’il est déjà installé, pour éviter tout conflit.

```sh
sudo systemctl stop mysql* maria*
sudo apt autoremove --purge mysql* maria*
```

*Des erreurs apparaitront si une version de MySQL n’est pas installée.*

On installe le serveur et le client MariaDB :

`sudo apt install default-mysql-server default-mysql-client`

Ensuite, on sécurise l’installation, le script ci-dessous nous guidera à travers certaines procédures qui élimineront les valeurs par défaut qui ne sont pas adaptées à un environnement de production.

On lance la commande, pour configurer, sécuriser et finaliser l’installation :

`sudo mysql_secure_installation`

1. **Entrer** > pas de mot de passe requis (donc on laisse vide)
2. **Entrer** > on change vers **unix_socket authentication**
3. **Entrer** > on change le mot de passe root de MariaDB
3.1. **Visual-Voiture12-Select!** : sera le mot de passe root
4. on retire l’utilisateur anonyme
5. **Entrer** > on désactive la connexion de root à distance
6. **Entrer** > on retire les bases de données de test
7. **Entrer** > on recharge les tables

On redémarre MariaDB :

`sudo service mariadb restart`

On active MariaDB au démarrage :

`sudo systemctl enable mariadb`

On vérifie que MariaDB fonctionne correctement :

`sudo systemctl status mariadb`

On vérifie que la connexion fonctionne :

`sudo mariadb -u root -pVisual-Voiture12-Select!`

Une fois connecté, on peut afficher les bases de données :

`SHOW DATABASES;`

On redémarre le VPS !

`sudo reboot`

## ![Accéder aux tables SQL](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Accéder%20aux%20tables%20SQL)

Pour accéder à vos données enregistrer dans vos tables SQL, il va nous falloir un outil de gestion de base de données complet, simple et efficace, ce que nous retrouvons avec [AdminEvo](https://docs.adminerevo.org/).

Généralement, on utilise [phpMyAdmin](https://www.phpmyadmin.net/), il est complet, simple d’utilisation mais malheureusement, ce gestionnaire n’est pas adapté à notre configuration. Vous trouverez les autres [logiciels SGBD sur sql.sh](https://sql.sh/logiciels).

**Pourquoi pas phpMyAdmin ?**

- phpMyAdmin est lourd, donc pour Tor ce n’ pas recommandé
- **il utilise pleinement JavaScript**, qui je le rappelle, est à proscrire lorsque que l’on navigue sur le réseau Tor
- des failles de sécurités critiques peuvent être présentes
- phpMyAdmin est surtout utilisé pour gérer des hébergements mutualisés, ce n’est pas notre cas

Nous n’allons pas installer de gestionnaire de base de données à propremment parlé, c’est trop risqué, dans le même style on peut utiliser [AdminerEvo](https://docs.adminerevo.org/). Il vous suffira de télécharger le fichier lorsque vous aurez besoin d’accéder à votre base de données et de le supprimer une fois vos opérations terminées.

**Pourquoi AdminerEvo ?**

- léger (fichier unique)
- ne dépand pas de JavaScript
- peut s’installer / désinstaller comme on le souhaite

**Les pré-requis**

- fonctionne avec MySQL, MariaDB, PostgreSQL, SQLite, MS SQL, Oracle, Elasticsearch, MongoDB, SimpleDB (plugin), Firebird (plugin) et ClickHouse (plugin)
- support à partir de PHP 5
- disponible en Français, Thai, Tamil, Romanian, Korean, Ukrainian, Dutch, Persian, Polish, Lithuanian, Slovene, Czech, Finnish, Italian, Hebrew, Catalan, Bengali, Vietnamese, English, Portuguese, Serbian, Slovak, Chinese (Traditional), Estonian, Turkish, Indonesian, Norwegian, Brazilian Portuguese, Chinese (Simplified), Danish, Bosnian, German, Japanese, Spanish, Russian, Arabic, Hungarian
- des dizaines de plugins pour toute sorte d’utilisation
- gratuit (Apache License ou GPL 2)

Pour la configuration, rien de plus simple, il vous suffit de [télécharger AdminerEvo](https://download.adminerevo.org/4.8.4/adminer/adminer.zip) et de renommer le fichier, par exemple :

```sh
mkdir /var/www/admin && cd /var/www/admin
wget https://github.com/adminerevo/adminerevo/releases/download/v4.8.4/adminer-4.8.4.php -O "$(mktemp adminer-XXXXXXXXXXXXXXXXXXXX.php)"
```

On renomme logiquement le fichier aléatoirement, pour éviter, si vous oubliez de le supprimer, qu’un méchant robot attaque votre site. Dans tous les cas, je vous conseille de le supprimer une fois que vous avez terminé vos tâches avec MySQL.

*Note : AdminerEvo, comme son nom l’indique, est une nouvelle version de l’original qui n’était plus maintenu*

- [Site officiel d’AdminerEvo](https://docs.adminerevo.org/)
- [Dépôt GitHub officiel](https://github.com/adminerevo/adminerevo)

## ![Installation de Tor](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Installation%20de%20Tor)

Tor est un réseau informatique superposé mondial et décentralisé. Il se compose de serveurs, appelés nœuds du réseau et dont la liste est publique. Ce réseau permet d’anonymiser l’origine de connexions TCP. Cela peut entre autres servir à anonymiser la source d’une session de navigation Web ou de messagerie instantanée. Cependant, l’anonymisation du flux n’est pas totale, car l’application peut transmettre des informations annexes permettant d’identifier la personne, c’est pourquoi le projet Tor développe également un navigateur Web fondé sur Firefox, Tor Browser, ainsi que d’autres applications spécialement modifiées pour préserver l’anonymat de leurs usagers. L’implémentation de référence du protocole s’appelle « tor », c’est un logiciel libre sous licence BSD révisée.

Le projet Tor reçoit le prix du logiciel libre 2010, dans la catégorie « projet d’intérêt social ». Le nom « Tor » est à l’origine un acronyme pour « The Onion Router », littéralement « le routeur oignon », qui s’est lexicalisé comme nom propre.

- [Site officiel de Tor](https://www.torproject.org/)
- [Dépôt GitLab officiel](https://gitlab.torproject.org/tpo/core/tor)
- [Clé GPG officielle du projet Tor](https://support.torproject.org/tbb/how-to-verify-signature/)

Avant toutes choses, on désinstalle Tor s’il est déjà installé, pour éviter tout conflit.

```sh
sudo systemctl stop tor*
sudo apt autoremove --purge tor*
```

*Des erreurs apparaitront si une version de Tor n’est pas installée.*

On ajoute la clé GPG & le dépôt :

```sh
curl -fsSL https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | sudo gpg --dearmor -o /etc/apt/keyrings/tor-archive-keyring.gpg
sudo chmod a+r /etc/apt/keyrings/tor-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/tor-archive-keyring.gpg] https://deb.torproject.org/torproject.org \
  $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/tor.list > /dev/null
```

On met à jour les paquets :

`sudo apt update`

On installe Tor :

`sudo apt install tor deb.torproject.org-keyring`

On active Tor au démarrage :

`sudo systemctl enable tor`

## ![Configuration du Hidden Service](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Configuration%20du%20Hidden%20Service)

On configure Tor :

`sudo nano /etc/tor/torrc`

Choisir le dossier de Tor :

```sh
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:80
```

On limite l’accès à Tor SOCKS sur 127.0.0.1 :

On recherche et ajoute / dé-commente ces lignes :

```sh
SocksPolicy accept 192.168.0.0/16
SocksPolicy reject *
```

On redémarre Tor :

`sudo systemctl restart tor`

On vérifie que Tor fonctionne correctement :

`sudo systemctl status tor`

Redémarrer Tor va créer le répertoire **/var/lib/tor/hidden_service** ainsi que deux fichiers très importants :

**hostname** : le nom de votre hidden service : *ujgftyuiolrmez3lotccipshtkleegetolb73fuirgj7r4o4vfu7ozyd.onion*, par exemple

**private_key** : la clé privée de votre hidden service

*Notez le contenu de **hostname** quelque part, on l’utilisera plus tard !*

## ![Générer une adresse .onion personnalisée](https://fakeimg.pl/500x100/ffffff/2c96f3/?text=Générer%20une%20adresse%20.onion%20personnalisée)

Si vous souhaitez une adresse .onion personnalisée, lisez la suite, sinon on passe directement à la [Partie Apache2](#-11).

Les adresses en .onion sont générées aléatoirement par un algorithme intégré à Tor et n’a pas d’identité propre, en revanche vous pouvez personnaliser les premiers caractères de l’adresse .onion. Le nombre de caractères dépendra de votre puissance de calcul liée à votre carte graphique ou processeur. On va utiliser le logiciel [mkp224o](https://github.com/cathugger/mkp224o) pour générer les adresses.

Temps moyen pour générer un alias personnalisé sur une carte graphique *nVidia GeForce GTX 4070* :

5 lettres : 1 seconde
6 lettres : 10 secondes
7 lettres : 15 secondes
8 lettres : 1 minute

On installe les pré-requis :

`apt install gcc libc6-dev libsodium-dev make autoconf`

On clone le dépôt :

`git clone https://github.com/cathugger/mkp224o.git`

On configure **mkp224o** :

```sh
./autogen.sh
./configure
make
```

On créé notre adresse personnalisée qui commence par **machou** :

`./mkp224o machou`

Un répertoire a été créé :

`cd machou********************************.onion`

On copie ce répertoire dans le dossier du **Hidden Service**, sur le serveur :

`sudo cp -r machou********************************.onion /var/lib/tor/hidden_service`

On ajuste les droits :

```sh
sudo chown -R tor: /var/lib/tor/hidden_service
sudo chmod -R u+rwX,og-rwx /var/lib/tor/hidden_service
```

## FAQ Debug Tor

On vérifie l’état du service Tor :

`sudo systemctl status tor`

Si le service n’est pas actif, on le redémarre :

`sudo systemctl restart tor`

On vérifie la configuration du Hidden Service et est correcte dans le fichier `/etc/tor/torrc`. Voici un exemple de configuration :

```sh
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:8080
```

On vérifie que le répertoire spécifié (`/var/lib/tor/hidden_service/`) existe et que Tor a les permissions nécessaires pour y accéder :

`sudo ls -la /var/lib/tor/hidden_service/`

On s’assure que le propriétaire du répertoire et de ses fichiers est **tor** :

`sudo chown -R tor:tor /var/lib/tor/hidden_service`

On vérifie que votre service web fonctionne :

`curl http://127.0.0.1:8080`

*Cela doit renvoyer la page d’accueil de votre service web. Si ce n’est pas le cas, vérifiez que votre service web est démarré et configuré correctement.*

On vérifie les journaux de Tor :

`sudo journalctl -u tor`

*Les journaux de Tor peuvent fournir des informations supplémentaires sur ce qui ne va pas; on recherche des messages d’erreur ou des avertissements qui pourraient indiquer ce qui ne va pas.*

On vérifie les permissions du répertoire et des fichiers :

```sh
sudo chmod 700 /var/lib/tor/hidden_service
sudo chmod 600 /var/lib/tor/hidden_service/*
```

*On s’assure que la machine peut accéder à Internet et que les ports nécessaires sont ouverts et non bloqués par un pare-feu.*

Une fois toutes les vérifications effectuées, on redémarre Tor :

`sudo systemctl restart tor`

----------

Si vous avez cette erreur :

`chown: utilisateur incorrect: « tor:tor »`

On doit créer le groupe tor :

`sudo groupadd tor`

On créé l’utilisateur tor et on l’ajoute au groupe tor :

`sudo useradd -g tor -s /bin/false tor`

On définit les permissions du répertoire du Hidden Service :

```sh
sudo mkdir -p /var/lib/tor/hidden_service
sudo chown -R tor:tor /var/lib/tor/hidden_service
```

On redémarre Tor :

`sudo systemctl restart tor`

On vérifie que Tor fonctionne correctement :

`sudo systemctl status tor`

**Partie Apache2**

*On doit configurer Apache2 afin de se connecter au **Hidden Service**, cette opération a été mentionnée plus haut, donc si vous ne l’avez pas effectuée, faites-le maintenant !*

On modifie le site à héberger :

`sudo nano /etc/apache2/sites-enabled/000-default.conf`

On ajoute le nom du Hidden Service (Contenu dans **/var/lib/tor/hidden_service/hostname**) pour qu’Apache2 le reconnaisse :

`ServerName machou********************************.onion`

On modifie le répertoire par défaut du site :

`DocumentRoot /var/www/html`

en

`DocumentRoot /var/www`

On ajoute en dessous : (*optionnel*)

```sh
Header always append X-Frame-Options SAMEORIGIN
Header set X-XSS-Protection "1; mode=block"
Header always set Strict-Transport-Security "max-age=2592000; includeSubDomains; preload;"
```

On active le VirtualHost par défaut :

`sudo a2ensite 000-default`

On récupère l’adresse IP de notre serveur, `185.141.102.27`, et on bloque l’accès direct à l’IP du serveur :

On crée un fichier **direct.conf** :

`sudo nano /etc/apache2/sites-available/direct.conf`

On insère :

```sh
<VirtualHost *:80>
	ServerName 185.141.102.27
	Redirect 403
	DocumentRoot /dev/null
</VirtualHost>
```

On active le VirtualHost :

`sudo a2ensite direct`

On quitte et on redémarre Apache2 :

`sudo service apache2 restart`

On teste une page :

```sh
touch /var/www/index.html && nano /var/www/index.html
Bienvenue sur mon Onion !
```

Une fois que toutes ces opérations sont effectuées, on redémarre le serveur :

`sudo reboot`

Et voilà, c’est terminé !

Maintenant, lancez le [Navigateur Tor](https://www.torproject.org/download/) sur votre ordinateur et connectez-vous au Hidden Service que vous avez généré plus haut !

![](https://i.ibb.co/M267kK8/onion.png)

## ![Facultatif](https://fakeimg.pl/200x100/ffffff/2c96f3/?text=Facultatif)

### Configuration de la langue

On peut changer la langue pour brouiller un peu les pistes :

`sudo dpkg-reconfigure locales`

Dans, la liste, on peut choisir les deux langues principales *en*, *en_US.UTF-8*, *fr*, *fr_FR.UTF-8* :

Cela renverra :

```sh
Generating locales (this might take a while)...
	en_US.UTF-8... done
	fr_FR.UTF-8... done
Generation complete.
```

La langue française a été choisie, mais libre à vous de configurer celle que vous souhaitez, c’est d’ailleurs fortement recommandé de ne pas choisir votre langue maternelle, sinon laissez celle par défaut.

### Configuration de la date et heure

On peut changer la date et heure pour brouiller un peu les pisteurs :

`sudo dpkg-reconfigure tzdata`

Dans la liste, on choisit le fuseau horaire que l’on souhaite, pour nous ça sera Paris : *8 - Europe* > *37 - Paris*

Cela renverra :

```sh
Current default time zone: 'Europe/Paris'
Local time is now:      Mon Aug 22 17:11:16 CEST 2022.
Universal Time is now:  Mon Aug 22 15:11:16 UTC 2022.
```

Le fuseau horaire de Paris a été choisi, mais libre à vous de configurer celui que vous souhaitez, c’est d’ailleurs fortement recommandé de ne pas choisir celui où vous logez (pays ou ville), sinon laissez celui par défaut.

### Rediriger le trafic réseau du serveur vers Tor

**Nipe** est un script pour faire de Tor Network votre passerelle réseau par défaut.

On installe **Perl** :

`sudo apt install perl`

On télécharge et on installe le script :

```sh
git clone https://github.com/htrgouvea/nipe && cd nipe/
cpanm --installdeps .
# doit être en root
perl nipe.pl install
```

Si erreur, on installe les dépendances **Perl** manquantes via **cpan** :

`sudo cpan install Try::Tiny Config::Simple JSON`

Pour lancer Nipe :

`perl nipe.pl start`

Pour arrêter Nipe :

`perl nipe.pl stop`

Commandes Nipe :

```sh
COMMAND          FUNCTION
install          Install dependencies
start            Start routing
stop             Stop routing
restart          Restart the Nipe circuit
status           See status
```

Examples :

```sh
perl nipe.pl install
perl nipe.pl start
perl nipe.pl stop
perl nipe.pl restart
perl nipe.pl status
```

Démo : ![](https://heitorgouvea.me/images/projects/nipe/demo.gif)

- [Dépôt GitHub officiel du projet Nipe](https://github.com/GouveaHeitor/nipe)

### Réécrire la RAM avant un arrêt / redémarrage du serveur

On va réécrire la RAM avant chaque arrêt / redémarrage serveur, pour éviter les attaques type [attaque par démarrage à froid](https://fr.wikipedia.org/wiki/Attaque_par_d%C3%A9marrage_%C3%A0_froid).

On install **secure-delete** :

`sudo apt install secure-delete`

Pour le faire fonctionner à chaque arrêt, nous allons créer un fichier dans la partie **/etc/init** nommé **memory_wipe.conf** : (à copier / coller simplement dans le terminal)

```sh
echo '# upstart config - wipes memory on shutdown

description "Wipes memory on shutdown"

start on (runlevel [016] and stopped gdm)

script
	sdmem
end script' > sudo tee /etc/init/memory_wipe.conf
```

### On supprime ses traces

`sudo find /var/log -type f -print0 | sudo xargs -0 shred -fuzv -n 35`

Le logiciel [shred](https://doc.ubuntu-fr.org/shred) est un utilitaire de ligne de commande sous Linux et Unix qui permet de supprimer définitivement des fichiers en écrivant de manière aléatoire des données sur les emplacements de stockage correspondants. Il est généralement utilisé pour supprimer des fichiers sensibles ou confidentiels de manière sécurisée afin de s’assurer qu’ils ne peuvent pas être récupérés.

Il est important de noter que l’utilisation de shred ne garantit pas à 100% que les données sont totalement irrécupérables, car il est toujours possible qu’une partie des données soit récupérée à l’aide de techniques de récupération avancées. Cependant, shred offre une méthode simple et efficace pour supprimer de manière sécurisée des fichiers sur un système Linux ou Unix.

### Alias utiles pour le serveur

On édite notre fichier **~/.bashrc** :

`sudo nano ~/.bashrc`

*alias “ cleany ”*, il servira à nettoyer les connexions et les logs :

`alias cleany="sudo truncate -s 0 /var/run/utmp && sudo truncate -s 0 /var/log/btmp && sudo truncate -s 0 /var/log/wtmp && sudo truncate -s 0 /var/log/lastlog"`

*alias “ clog ”*, il servira à nettoyer tous les fichiers contenus dans le dossier /var/log :

`alias clog="find /var/log -type f -print0 | sudo xargs -0 shred -fuzv -n 35"`

*alias "update"*, il servira à mettre à jour et nettoyer votre serveur, mettre à jour le cache de recherche et la régénération des liens symboliques :

`alias update="sudo apt update && sudo apt upgrade && sudo apt full-upgrade && sudo apt dist-upgrade && sudo apt clean && sudo apt autoclean && sudo apt autoremove && sudo updatedb && sudo ldconfig && sudo chown www-data:www-data /var/www -R"`

On recharge le fichier **~/.bashrc** :

`source ~/.bashrc`

### Désactiver le transfert IP et le multi-hébergement

`sudo nano /etc/sysctl.conf`

On modifie :

```sh
net.ipv4.ip_forward=0
net.ipv6.conf.all.forwarding=0
```

On dé-commente :

```sh
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.rp_filter=1
```

On redémarre le serveur :

`sudo reboot`

### Désactiver IPv6

Si vous souhaitez désactiver l’IPv6, on procède comme cela :

Dans la console, on tape :

```sh
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.all.autoconf=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.autoconf=0
```

On modifie les options du kernel : (prudence !)

`sudo nano /etc/sysctl.conf`

On change comme suit :

```sh
# Désactivation de IPv6 pour toutes les interfaces
net.ipv6.conf.all.disable_ipv6 = 1

# Désactivation de l’auto configuration pour toutes les interfaces
net.ipv6.conf.all.autoconf = 0

# Désactivation de ipv6 pour les nouvelles interfaces (ex : si ajout de carte réseau)
net.ipv6.conf.default.disable_ipv6 = 1

# Désactivation de l’auto configuration pour les nouvelles interfaces
net.ipv6.conf.default.autoconf = 0
```

On charge les paramètres **sysctl** :

`sudo sysctl -p`

On modifie le fichier de l’interface réseau :

`sudo nano /etc/network/interfaces`

On commente ces lignes :

```sh
#iface eth0 inet6 static
#    address 2001:41D0:1:CD6b::1
#    netmask 128
#    post-up /sbin/ip -f inet6 route add _adresse_ip_v6_ dev eth0
```

Suppression de la ligne IPv6 dans /etc/hosts :

`sudo nano /etc/hosts`

On commente cette ligne :

`#::1 localhost ip6-localhost ip6-loopback`

On rejette tout le trafic IPv6 en éditant le fichier /etc/iptables/rules.v6, il doit contenir :

```sh
*filter

-A INPUT -j REJECT
-A FORWARD -j REJECT
-A OUTPUT -j REJECT

COMMIT
```

On applique :

`sudo ip6tables-restore < /etc/iptables/rules.v6`

Une fois que toutes ces opérations sont effectuées, on redémarre le serveur :

`sudo reboot`

### Quelques tutoriels utiles

Je vous recommande ces tutoriels :

- [Bien débuter sur Debian](https://mondedie.fr/d/5438) [ Tutoriel réalisé par [Meister](https://mondedie.fr/u/Meister) ] 🇫🇷
- [Installation et configuration de Fail2ban](https://mondedie.fr/d/5318/2) [ Tutoriel réalisé par [ex_rat](https://mondedie.fr/u/ex_rat) ] 🇫🇷
- [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) 🇺🇸
- [The Onion Diaries](https://github.com/alecmuffett/the-onion-diaries/tree/master) 🇺🇸

> À faire :
> - ajouter nginx pour créer un proxy inverser
> - heberger le site directement dans la ram
> - bloquer les ports inutiles
> - chiffrer le disque dur
> - stocker les clés dans le CPU plutôt que dans la RAM (qui peut être copiée) par exemple via TRESOR
> - SSH par clé avec accès selon IP ou par PortKnocking
> Concernant les logs, 2 solutions, sur des machines dites “ sécurisées ” :
> - soit de ne pas en émettre
> - soit de les traiter avec logrotate (et le paramètre shred — 3 max.)

Si vous avez des idées ou des améliorations à proposer, n’hésitez pas à postez un commentaire ci-dessous.