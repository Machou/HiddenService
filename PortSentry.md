# PortSentry

- [Qu’est ce que PortSentry ?](#)
- [Installation de PortSentry](#installation-de-portsentry)
- [Configuration de PortSentry](#configuration-de-portSentry)

## Qu’est ce que PortSentry ?

[PortSentry](https://github.com/Perdjesk/portsentry) est un outil de détection et de réponse aux scans de port conçu pour améliorer la sécurité des systèmes en surveillant les connexions réseau suspectes. Il peut identifier les tentatives de scan de port sur un système et réagir en bloquant les adresses IP des attaquants, empêchant ainsi les intrusions potentielles.

### Fonctionnalités principales de PortSentry

1. **Détection de scans de port** : PortSentry surveille les tentatives de connexion sur les ports ouverts ou fermés pour détecter les scans de port effectués par des attaquants
2. **Réponse active** : lorsqu’un scan de port est détecté, PortSentry peut réagir en bloquant l’adresse IP de l’attaquant en temps réel, souvent en ajoutant des règles de pare-feu ou en modifiant les fichiers de configuration réseau
3. **Modes de fonctionnement** : il peut fonctionner en plusieurs modes, notamment le mode "stealth" où il simule des ports ouverts pour attirer et détecter les scanners, et le mode "advanced" où il surveille des ports spécifiques
4. **Flexibilité et configuration** : il est hautement configurable, permettant aux administrateurs de définir les ports à surveiller, les actions à entreprendre en cas de détection de scan, et les niveaux de sensibilité

### Services et options

#### Surveillance des ports

- **Surveillance des ports ouverts** : PortSentry peut surveiller les ports ouverts légitimes pour détecter les tentatives de connexion suspectes
- **Surveillance des ports fermés** : il peut également surveiller les ports fermés pour détecter les scans de port, attirant les attaquants vers des ports qui ne devraient pas être accessibles

#### Actions de réponse

- **Blocage d’IP** : en réponse à un scan de port, PortSentry peut bloquer automatiquement l’adresse IP de l’attaquant en ajoutant des règles de pare-feu (comme iptables) ou en modifiant les fichiers de configuration des hôtes
- **Notifications** : il peut envoyer des notifications aux administrateurs en cas de détection d’activités suspectes, permettant une réponse rapide aux incidents

#### Modes de fonctionnement

- **Mode "attracteur" (Honey Pot)** : simule des ports ouverts pour attirer les attaquants et détecter les scans
- **Mode "détection avancée"** : surveille une liste spécifiée de ports pour des tentatives de connexion suspectes
- **Mode "stealth"** : fonctionne de manière discrète pour éviter d’être détecté par les scanners

#### Configuration

- **Sensibilité** : Les administrateurs peuvent configurer le niveau de sensibilité de PortSentry pour définir combien de tentatives de connexion sont nécessaires avant qu’une IP soit bloquée
- **Personnalisation des ports** : il est possible de spécifier quels ports doivent être surveillés en fonction des besoins spécifiques de sécurité
- **Options de journalisation** : il peut être configuré pour enregistrer les tentatives de connexion suspectes dans des fichiers de journal pour une analyse ultérieure

PortSentry est un outil essentiel pour renforcer la sécurité des systèmes en détectant et en réagissant aux scans de port, réduisant ainsi le risque d’intrusion par des attaquants potentiels.

## Installation de PortSentry

On installe PortSentry :

`sudo apt install portsentry`

## Configuration de PortSentry

On démarre le service :

`systemctl start portsentry`

On active le démarrage automatique :

`systemctl enable portsentry`

On vérifie que le service fonctionne :

`systemctl status portsentry`

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

`route del -host IP-PROBLÉMATIQUE reject`

<p align="center">
  [ <a href="README.md">Retour à l’accueil</a> ]
  [ <a href="#portsentry">Remonter la page</a> ]
</p>