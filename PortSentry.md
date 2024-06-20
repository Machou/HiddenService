# PortSentry

- [Installation de PortSentry](#installation-de-portsentry)
- [Configuration de PortSentry](#configuration-de-portSentry)
- [Configuration de Fail2ban](#configuration-de-fail2ban)

Les tentatives d’intrusion sur les ordinateurs connectés à Internet sont fréquentes et variées, comprenant les accès SSH non autorisés, l’exploitation de failles de sécurité sur des services Web comme WordPress et phpMyAdmin, ainsi que sur des services avec des vulnérabilités connues tels que SMB, RDP, Docker, et MongoDB. Ces attaques sont souvent automatisées à partir de serveurs eux-mêmes vulnérables.

Pour se protéger, il est possible d’utiliser PortSentry, un logiciel open source sous Linux. PortSentry détecte les scans de ports et imite la présence de services réels. Lorsqu’un port est ouvert par un tiers, PortSentry exécute des commandes pour bloquer le trafic réseau de la machine attaquante, empêchant ainsi d’autres attaques.

En plus de protéger contre les intrusions, PortSentry peut aussi aider à se défendre contre des systèmes comme Shodan qui cataloguent les services exposés par les serveurs, informations pouvant être exploitées pour des attaques. Cependant, pour les attaques de type force brute, Fail2ban est recommandé. Fail2ban analyse les logs pour détecter des patterns d’attaque et applique des règles de blocage réseau, offrant une protection efficace contre ce type de menace.

## Installation de PortSentry

On installe PortSentry :

`sudo apt install portsentry`

## Configuration de PortSentry

On ajoute l’IP de notre serveur pour éviter de se faire bannir :

`sudo nano /etc/portsentry/portsentry.ignore.static`

à la fin du fichier, on y ajoute notre IP.

On configure PortSentry :

`sudo nano /etc/portsentry/portsentry.conf`

Ces options vous permettent d’activer les réponses automatiques pour les protocoles TCP / UDP. Cela est utile si vous souhaitez afficher des avertissements pour les connexions sur un protocole particulier (par exemple, bloquer TCP mais pas UDP). Pour éviter une attaque par déni de service (DoS) contre UDP et la détection de scan furtif pour TCP, il peut être préférable de désactiver le blocage tout en maintenant l’avertissement activé.

La troisième option permet d’exécuter uniquement une commande externe en cas de scan pour lancer un script de notification sans interrompre la route. Cela peut être utile pour les administrateurs qui veulent bloquer TCP tout en recevant seulement des alertes.

* 0 = ne pas bloquer les scans TCP / UDP
* 1 = bloquer les scans TCP / UDP
* 2 = exécuter uniquement la commande externe (KILL_RUN_CMD)

On change :

```sh
BLOCK_UDP="0"
BLOCK_TCP="0"
```

En :

```sh
BLOCK_UDP="1"
BLOCK_TCP="1"
```

On vérifie que ces options sont décommentées :

`KILL_ROUTE="/sbin/route add -host $TARGET$ reject"`

et

`KILL_HOSTS_DENY="ALL: $TARGET$ : DENY"`

*à chercher dans le fichier*

On change :

`#KILL_RUN_CMD_FIRST = "0"`

en

`KILL_RUN_CMD_FIRST = "1"`

Plus loin, on ajoute la commande :

`KILL_RUN_CMD="/sbin/iptables -I INPUT -s $TARGET$ -j DROP && /sbin/iptables -I INPUT -s $TARGET$ -m limit --limit 3/minute --limit-burst 5 -j LOG --log-level debug --log-prefix 'Portsentry: dropping: '"`

On changé également :

`SCAN_TRIGGER="0"`

en

`SCAN_TRIGGER="1"`

Il y a deux façons d’utiliser PortSentry.

* On définit la liste des ports TCP et UDP à écouter depuis **UDP_PORTS** et **TCP_PORTS** du fichier de configuration
* On laisse PortSentry ouvrir tous les ports disponibles avant le **port 1024**, port défini par défaut dans les paramètres **ADVANCED_PORTS_TCP** et **ADVANCED_PORTS_UDP** du fichier de configuration

On ouvre le fichier de configuration de PortSentry :

`sudo nano /etc/default/portsentry`

On change :

```sh
TCP_MODE="tcp"
UDP_MODE="udp"
```

en

```sh
TCP_MODE="atcp"
UDP_MODE="audp"
```

Avec ce changement PortSentry laissera ouvert tous les ports disponibles.

`sudo nano /etc/portsentry/portsentry.conf`

On modifie dans le fichier de configuration :

```sh
TCP_MODE="tcp"
UDP_MODE="udp"
```

On peut aussi détecter les *Stealth Scan* en modifiant les paramètres aux valeurs suivantes, utile pour détecter les scans de ports furtifs :

```sh
TCP_MODE="stcp"
UDP_MODE="sudp"
```

On redémarre PortSentry :

`sudo service portsentry restart`

On patiente quelques minutes / heures et on peut déjà constater que PortSentry fait correctement son travail en affichant le fichier des IP bloquées :

`sudo tail -f /etc/hosts.deny`

ou

`sudo tail -f /var/lib/portsentry/portsentry.history`

*on note que cette commande affiche la date, l’IP ainsi que le port*

Si vous rencontrez une erreur avec une IP qui a été bannie alors qu’elle ne devrait pas l’être, on peut la retirer :

`route del -host IP-PROBLEMATIQUE reject`

[Retour à l’accueil](README.md)