# Sécuriser son serveur Linux Ubuntu 20.04

## 1 - Modifier son mot de passe root !

Première chose à faire, si comme moi tu as reçu ton mot de passe root par mail, considère le comme potentiellement intercepté. (Comme pour tous tes mots de passe que tu reçois par mail d’ailleurs).
Ton mot de passe root ne doit être connu que par toi et n’être accessible par personne d’autre, alors on le change tout de suite. Connectes toi en SSH sur ton serveur et change ton mot de passe avec la commande :

```bash
passwd root
```

## 2 - Root c’est fini, bonjour admin
Comme je l’ai dit, la plupart des attaques sont des attaques où des robots vont tenter de se connecter en SSH sur ton serveur avec l’utilisateur root en tentant des combinaisons différentes de mot de passe. Une bonne pratique de sécurité consiste donc à bloquer root et de le remplacer par un utilisateur de ton choix qui aura les privilèges root.

Commences par ajouter un utilisateur, ici, je l’appelle « rp » parce que c’est mes initiales, mais ne sois pas benêt et mets les tiennes à la place 😉 :

```bash
adduser rp
```

Puis on ajoute l’utilisateur aux groupes lui permettant d’avoir les privilèges root (j’ai rajouté le groupe adm pour pouvoir consulter les logs sans devoir utiliser sudo) :

```bash
usermod -aG root,sudo,adm rp
```

On modifie maintenant la configuration SSH pour interdire la connexion en tant que root. Tu peux aussi modifier ton port de connexion SSH si tu le souhaites, ça t’évitera 99.99% des attaques par brute force, mais c’est un peu pénible pour la suite de devoir toujours indiquer ton numéro de port …
```bash
sudo nano /etc/ssh/sshd_config
``` 
```bash
...
# Interdire à root de se connecter en SSH
PermitRootLogin no
# Facultatif : Modifier le port de connexion pour le SSH
Port 4854
# Facultatif : Seul admin peux se connecter en SSH
AllowUsers rp
# Facultatif : Seul les utilisateurs du groupe sshusers peuvent se connecter en SSH
AllowGroups sshusers
...
``` 
Attention ! Avant de redémarrer ton service SSH, avec une mauvaise configuration, tu risques de perdre complètement l’accès à ton serveur ! Penses à garder ta session ssh root active le temps de vérifier que tout va bien et que tu disposes bien des privilèges root avec le nouvel utilisateur.

On redémarre le service SSH :
```bash
service ssh restart
``` 

On se connecte en SSH avec le nouvel utilisateur et on teste que la commande sudo fonctionne bien :
```bash
sudo ls -l ./
``` 

Si ça ne fonctionne pas, ne quittes surtout par ta session root et fais ce qu’il faut pour que ça marche.

Le principe de sudo est simple, tout ce que tu exécutes en utilisant le préfixe sudo va exécuter la commande avec les droits root. Tu peux même prendre la main sur l’utilisateur root avec la commande (mais à éviter) :
```bash
sudo su
``` 

## 3 – Un firewall strict = maîtrise des portes d’entrées / sorties

Sur ton serveur, plusieurs services sont en cours d’exécution et certains sont à l’écoute sur des ports ouverts ce qui peut présenter un risque si le logiciel derrière le port ouvert contient une faille. Il est important de configurer un firewall qui bloquera tous les ports sauf ceux dont nous avons besoin et dont le logiciel qui l’utilise est sérieux, réputé pour être sécurisé et dont on a confiance.

Aller hop ! On installe l’incontournable paquet iptable :
```bash
sudo apt-get install iptables
``` 

On définit les règles du firewall, en fonction des services que tu utilises, commentes, dé-commentes ou ajoute des lignes :
```bash
sudo nano /etc/init.d/firewall
``` 
```bash
#! /bin/sh
### BEGIN INIT INFO
# Provides:          PersonalFirewall
# Required-Start:    $remote_fs $syslog
# Required-Stop:     $remote_fs $syslog
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Personal Firewall
# Description:       Init the firewall rules
### END INIT INFO

# programme iptables IPV4 et IPV6
IPT=/sbin/iptables                                                                                                                                                                 
IP6T=/sbin/ip6tables
# Les IPs
#IP_TRUSTED=xx.xx.xx.xx

do_start() {                                                                                                                                                                       
# Efface toutes les regles en cours. -F toutes. -X utilisateurs                                                                                                                
$IPT -t filter -F                                                                                                                                                              
$IPT -t filter -X                                                                                                                                                              
$IPT -t nat -F                                                                                                                                                                 
$IPT -t nat -X                                                                                                                                                                 
$IPT -t mangle -F                                                                                                                                                              
$IPT -t mangle -X                                                                                                                                                              
$IP6T -t filter -F                                                                                                                                                             
$IP6T -t filter -X                                                                                                                                                              
$IP6T -t mangle -F                                                                                                                                                             
$IP6T -t mangle -X                                                                                                                                                             
# strategie (-P) par defaut : bloc tout l'entrant le forward et autorise le sortant                                                                                            
$IPT -t filter -P INPUT DROP                                                                                                                                                   
$IPT -t filter -P FORWARD DROP                                                                                                                                                 
$IPT -t filter -P OUTPUT ACCEPT                                                                                                                                                
$IP6T -t filter -P INPUT DROP                                                                                                                                                  
$IP6T -t filter -P FORWARD DROP                                                                                                                                                
$IP6T -t filter -P OUTPUT ACCEPT                                                                                                                                               
# Loopback                                                                                                                                                                     
$IPT -t filter -A INPUT -i lo -j ACCEPT                                                                                                                                        
$IPT -t filter -A OUTPUT -o lo -j ACCEPT                                                                                                                                       
$IP6T -t filter -A INPUT -i lo -j ACCEPT                                                                                                                                       
$IP6T -t filter -A OUTPUT -o lo -j ACCEPT                                                                                                                                      
# Permettre a une connexion ouverte de recevoir du trafic en entree                                                                                                            
$IPT -t filter -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT                                                                                                         
$IP6T -t filter -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT                                                                                                        
# ICMP                                                                                                                                                                         
$IPT -t filter -A INPUT -p icmp -j ACCEPT                                                                                                                                      
# DNS:53                                                                                                                                                                       
$IPT -t filter -A INPUT -p tcp --dport 53 -j ACCEPT                                                                                                                            
$IPT -t filter -A INPUT -p udp --dport 53 -j ACCEPT                                                                                                                            
$IP6T -t filter -A INPUT -p tcp --dport 53 -j ACCEPT                                                                                                                           
$IP6T -t filter -A INPUT -p udp --dport 53 -j ACCEPT                                                                                                                           
# SSH:22
# ATTENTION, indiques bien ton port personnalisé si tu l'as changé                                                                                                                                                                     
$IPT -t filter -A INPUT -p tcp --dport 22 -j ACCEPT                                                                                                                            
$IP6T -t filter -A INPUT -p tcp --dport 22 -j ACCEPT                                                                                                                           
# HTTP:80                                                                                                                                                                     
$IPT -t filter -A INPUT -p tcp --dport 80 -j ACCEPT                                                                                                                            
$IP6T -t filter -A INPUT -p tcp --dport 80 -j ACCEPT                                                                                                                           
# HTTPS:443                                                                                                                                                                        
$IPT -t filter -A INPUT -p tcp --dport 443 -j ACCEPT                                                                                                                           
$IP6T -t filter -A INPUT -p tcp --dport 443 -j ACCEPT                                                                                                                          
# SMTP:25/587/465                                                                                                                                                                         
$IPT -t filter -A INPUT -p tcp --dport 25 -j ACCEPT                                                                                                                            
$IP6T -t filter -A INPUT -p tcp --dport 25 -j ACCEPT                                                                                                                           
$IPT -t filter -A INPUT -p tcp --dport 587 -j ACCEPT                                                                                                                           
$IP6T -t filter -A INPUT -p tcp --dport 587 -j ACCEPT
#    $IPT -t filter -A INPUT -p tcp --dport 465 -j ACCEPT
#    $IP6T -t filter -A INPUT -p tcp --dport 465 -j ACCEPT
    # POP3:110/995                                                                                                                                                                         
#    $IPT -t filter -A INPUT -p tcp --dport 110 -j ACCEPT
#    $IP6T -t filter -A INPUT -p tcp --dport 110 -j ACCEPT
#    $IPT -t filter -A INPUT -p tcp --dport 995 -j ACCEPT
#    $IP6T -t filter -A INPUT -p tcp --dport 995 -j ACCEPT
    # IMAP / SIEVE : 143/993/4190                                                                                                                                                                 
#    $IPT -t filter -A INPUT -p tcp --dport 143 -j ACCEPT
#    $IP6T -t filter -A INPUT -p tcp --dport 143 -j ACCEPT
    $IPT -t filter -A INPUT -p tcp --dport 993 -j ACCEPT                                                                                                                           
    $IP6T -t filter -A INPUT -p tcp --dport 993 -j ACCEPT                                                                                                                          
    $IPT -t filter -A INPUT -p tcp --dport 4190 -j ACCEPT                                                                                                                          
    $IP6T -t filter -A INPUT -p tcp --dport 4190 -j ACCEPT                                                                                                                         
    # accepte tout d'une ip en TCP                                                                                                                                                 
#    $IPT -t filter -A INPUT -p tcp -s $IP_TRUSTED -j ACCEPT
    echo "firewall started [OK]"                                                                                                                                                   
}
# fonction qui arrete le firewall
do_stop() {                                                                                                                                                                        
# Efface toutes les regles                                                                                                                                                     
$IPT -t filter -F                                                                                                                                                              
$IPT -t filter -X                                                                                                                                                              
$IPT -t nat -F                                                                                                                                                                 
$IPT -t nat -X                                                                                                                                                                 
$IPT -t mangle -F                                                                                                                                                              
$IPT -t mangle -X
$IP6T -t filter -F
$IP6T -t filter -X
$IP6T -t mangle -F
$IP6T -t mangle -X
# remet la strategie
$IPT -t filter -P INPUT ACCEPT
$IPT -t filter -P OUTPUT ACCEPT
$IPT -t filter -P FORWARD ACCEPT
$IP6T -t filter -P INPUT ACCEPT
$IP6T -t filter -P OUTPUT ACCEPT
$IP6T -t filter -P FORWARD ACCEPT
#
echo "firewall stopped [OK]"
}
# fonction status firewall
do_status() {
# affiche les regles en cours
clear
echo Status IPV4
echo -----------------------------------------------
$IPT -L -n -v
echo
echo -----------------------------------------------
echo
echo status IPV6
echo -----------------------------------------------
$IP6T -L -n -v
echo
}
case "$1" in
start)
do_start
exit 0
;;
stop)
do_stop
exit 0
;;
restart)
do_stop
do_start
exit 0
;;
status)
do_status
exit 0
;;
*)
echo "Usage: /etc/init.d/firewall {start|stop|restart|status}"
exit 1
;;
esac
``` 

Il faut donner les droits d’exécution au script :
```bash
sudo chmod +x /etc/init.d/firewall
```
Attention ! Si tu définis mal les règles, tu peux te bloquer toi même et ne plus pouvoir accéder au serveur. Je te conseille fortement d’exécuter le script puis essaye de te ré-authentifier en SSH. Si après exécution du script, tout est bloqué, tu as perdu ta session et impossible de te reconnecter, redémarre le serveur pour réinitialiser le firewall.

C’est partit ! On lance le script :
```bash
sudo /etc/init.d/firewall start
```

Si tout va bien et que tu peux toujours te connecter en SSH, alors on peut indiquer au serveur que tu veux que le script soit exécuté au démarrage pour que jamais le firewall ne soit corrompu :
```bash
sudo update-rc.d firewall defaults
```
Si tu changes d’avis, tu peux le retirer avec la commande :
```bash
sudo update-rc.d -f firewall remove
```
## 4 – Fail2Ban, surveillance des logs et bannissement par IP

Fail2Ban permet de bannir les machines qui tentent de brute forcer votre serveur.

On installe le paquet fail2ban :
```bash
sudo apt-get install fail2ban
```

Créer le fichier de configuration
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
```bash
sudo nano /etc/fail2ban/jail.local
```
```bash
...
# Durée du bannissement en seconde
bantime  = 600
# Plage de temps de surveillance des logs
findtime = 600
# Nombre de tentatives avant bannissement
maxretry = 3
# Adresse mail des notifications
destemail = moi@mondomain.fr
# Nom du sender dans les notifications mail
sendername = Fail2ban
...
```

Dans ce fichier, sont configurés différents filtres [ssh], [apache-auth]... On va indiquer au serveur lesquels doivent être surveillés.
```bash
sudo nano /etc/fail2ban/jail.d/custom.conf
```
```bash
[sshd]
enabled = true
[sshd-ddos]
enabled = true
[postfix]
enabled = true
[dovecot]
enabled = true
#[sieve]
#enabled = true
[recidive]
enabled = true
```

C’est un exemple, il faut indiquer les services que tu utilises toi.

Depuis une dernière version de fail2ban, pour recevoir les mails de notification, il faut indiquer votre mail dans les fichiers :
```bash
sudo nano /etc/fail2ban/action.d/sendmail-common.conf
sudo nano /etc/fail2ban/action.d/mail.conf
sudo nano /etc/fail2ban/action.d/mail-whois.conf
```

Je m’arrête 2 secondes sur le filtre [recidive]. Il permet de bannir plus longtemps les IPs qui ont déjà été bannie plusieurs fois. Par exemple, on va pouvoir dire à fail2ban de bannir pendant 1 an les IPs qui ont déjà été bannies 3 fois dans la journée.

Une fois que tout est configuré, on redémarre le service :
```bash
sudo service fail2ban restart
```

Puis on vérifie que les prisons sont bien en service :
```bash
sudo fail2ban-client status
```

## 5 – Toujours à jour avec cron-apt
Bon, c’est bien beau, mon serveur est sécurisé mais que se passe il le jour où une faille dans un logiciel est découverte pendant que je suis en vacance ? Un pirate aura tout son temps pour en profiter.

Un serveur fait les choses de façon automatique non ? Hé bien qu’il se mette à jour tout seul ! Aller on installe le paquet cron-apt :
```bash
sudo apt-get install cron-apt
```

On veux faire les mises à jour de sécurité uniquement, donc on crée un nouveau fichier de source :
```bash
grep security /etc/apt/sources.list > /etc/apt/security.sources.list
```

Un peu de configuration :
```bash
sudo nano /etc/cron-apt/config
```
```bash
APTCOMMAND=/usr/bin/apt-get
OPTIONS="-o quiet=1 -o Dir::Etc::SourceList=/etc/apt/security.sources.list"
MAILTO="moi@mondomain.fr"
MAILON="upgrade"
```

On indique qu’on veut que cron-apt installe les mises à jour quand il en trouve :
```bash
sudo nano /etc/cron-apt/action.d/3-download
```
```bash
dist-upgrade -y -o APT::Get::Show-Upgraded=true
```

Cron-apt se lance par défaut toute les nuits à 4h du matin, si tu veux modifier ça, c’est ici : 
```bash
sudo nano /etc/cron.d/cron-apt
```

Ok, on est pas trop mal là, c’est plutôt bien sécure ! Mais nous on est des dingues, on va encore plus loin ! Il est toujours possible de protéger plus son serveur, mais plus l’on protège plus on ajoute de contraintes. La protection ultime n’existe pas, du moment que la machine est connectée à internet, la moindre faille peut-être exploitée de façon plus ou moins ingénieuse.

# Sécurité avancée
## 1 – Rkhunter : Détecter les intrusions
Rhkunter calcule les empreintes MD5 des logiciels installés sur ton serveur. Si un pirate arrivait à pénétrer ton serveur, il est fort probable qu’il ajoute une backdoor afin de pouvoir revenir facilement par la suite. Rkhunter détecte donc ces backdoors et t’avertis.

On installe le paquet rkhunter :
```bash
sudo apt install rkhunter
```

Puis on édite la configuration :
```bash
sudo nano /etc/default/rkhunter
```

### Pour effectuer une vérification chaque jour
```bash
CRON_DAILY_RUN="yes"
REPORT_EMAIL="moi@mondomaine.fr"
```

Rkhunter va parfois t’avertir de modifications sur un logiciel alors que tu en es à l’origine (suite à une mise à jour par exemple), dans ce cas, il faut mettre la base d’empreintes à jour avec la commande :
```bash
sudo rkhunter --propupd
```

## 2 – Logwatch : Surveillez l’activité de votre serveur
Logwatch est un utilitaire qui analyse vos logs du serveur pour te faire un récapitulatif tous les jours sous forme de mail. Delà te permet de détecter les anomalies. Logwatch va par exemple te faire un récapitulatif des mails sortants de votre serveur. Tu pourras ainsi voir si ton serveur ne sert pas de passerelle pour le spam. Il va t’indiquer les attaques bloquées par fail2ban, etc …

On installe le paquet logwatch :
```bash
sudo apt install logwatch
```

On le configure :
```bash
sudo cp /usr/share/logwatch/default.conf/logwatch.conf /etc/logwatch/conf/logwatch.conf
sudo nano /etc/logwatch/conf/logwatch.conf
```

```bash
# Pour recevoir les mails au format html, c'est plus agréable à lire
Format = html
# Adresse sur laquelle recevoir les mails
MailTo = moi@mondomaine.fr
# Niveau de détail des logs
Detail = Med
```

Et si tu veux recevoir un rapport là tout de suite :
```bash
sudo logwatch --mailto moi@mondomaine.fr --output html
```

## 3 – Analyser de trafic en temps réel
Les deux logiciels suivants analysent le trafic en temps réel pour empêcher les attaques. Ils se lancent donc à chaque requête vers le serveur pour détecter les attaques et consomment donc des ressources, ce qui ralentit les échanges. Dans le cas d’un serveur d’hébergement web, ce n’est pas très pertinent si tu souhaites des sites qui répondent du tac au tac. Après, c’est toi qui vois en fonction de tes besoins en sécurité. Je ne vais pas m’attarder dessus dans cet article.

Portsentry : Détection des scans de ports très puissante. Pour l’utiliser c’est ici.
Snort : Système de détection d’intrusion. Pour l’utiliser c’est ici.

## 4 – Une alarme SSH
Une autre chose très simple que tu peux mettre en place est une alerte mail dès qu’une authentification ssh a lieu sur ton serveur.

Cette méthode est plutôt contraignante, car normalement, tous les mails reçus sont des faux positifs …

Édites le fichier .bashrc de l’utilisateur à protéger (dans le homedir) et ajoutes à la fin du fichier :
```bash
sudo nano ~/.bashrc
```
```bash
echo "Objet : Alerte connexion SSH.
Serveur : `hostname`
Date : `date`
`who` " | mail -s "`hostname` connexion ssh de : `who | cut -d"(" -f2 | cut -d")" -f1`" moi@mondomaine.fr
```

Si tu veux le mettre en place pour l’utilisateur root, le fichier est /root/.bashrc car le homedir de root est /root/

## 5- Restrictions par IP
Cette méthode est très efficace, car seul l’IP de ton choix peut communiquer avec le serveur sur le protocole choisi. Par contre, les offres d’accès internet permettent rarement d’avoir une IP fixe. Et puis tu ne pourras plus accéder à ton serveur de n’importe où (sauf VPN).

Il existe plusieurs méthodes pour restreindre par IPs. Je te présente celle qui utilise le firewall. Si tu as suivi ce tuto depuis le début, nous avons créé un fichier /etc/init.d/firewall.

Il suffit de rajouter les options `-s 100:100:100:100, 101:101:101:101` sur les protocoles à restreindre (avec tes IPs bien sûr …).

Exemple pour restreindre le protocole SSH sur mon IP seulement :
```bash
sudo nano /etc/init./firewall
```
```bash
$IPT -t filter -A INPUT -p tcp --dport 22 -s 100:100:100:100 -j ACCEPT
$IP6T -t filter -A INPUT -p tcp --dport 22 -s 100:100:100:100 -j ACCEPT
```

Tu peux faire de même sur tous les ports que tu veux. Si tu veux le faire temporairement, par exemple pour autorisé un ami à se connecter de chez lui :

```bash
iptables -t filter -A INPUT -p tcp --dport 22 -s 100:100:100:100 -j ACCEPT
ip6tables -t filter -A INPUT -p tcp --dport 22 -s 100:100:100:100 -j ACCEPT
```

Quand il a fini, on réinitialise, on applique la tolérance zéro !
```bash
iptables -t filter -A INPUT -p tcp --dport 22 -s 100:100:100:100 -j DROP
ip6tables -t filter -A INPUT -p tcp --dport 22 -s 100:100:100:100 -j DROP
```

Tu peux aussi redémarrer le firewall pour lui retirer les droits (cela réinitialise toutes les règles) :
```bash
sudo /etc/init.d/firewall restart
```

## 6 – Authentification SSH par clé privée
L’authentification par clé privée augmente considérablement la difficulté de brute forcer le serveur en SSH. Il faut à la fois un fichier et une clé pour décrypter ce fichier avant de pouvoir se connecter au serveur.

J’ai fait un article expliquant la mise en place d’une authentification SSH par clé privée : c’est ici !

Une fois ta clé publique insérée sur le serveur, configure ton service SSH pour bloquer l’authentification par mot de passe simple :
```bash
sudo nano /etc/ssh/sshd_config
```
```bash
...
RSAAuthentication yes
# Autoriser l'authentification par clé privée
PubkeyAuthentication yes
# Bloquer l'authentification avec mot de passe simple
PasswordAuthentication no    
...
```

On relance le service SSH :
```bash
sudo service ssh restart
```

Attention ! Encore une fois, assure-toi que ça marche bien en essayant de t’authentifier avec ta clé avant de fermer ta session sinon tu vas perdre accès à ton serveur …

Si tu gères plusieurs serveurs, il est plus agréable de se connecter avec la même clé privée. Ainsi, aucun mot de passe ne t’es demandé pour te connecter une fois que ta clé privée est déverrouillée. Il y a donc aussi des avantages à utiliser cette méthode. tu peux aussi permettre la connexion via clé privée et via mot de passe (au choix). Dans ce cas, tu ne gagne pas en sécurité, mais tu gagne en simplicité d’utilisation.

# Le mot de la fin
La sécurité d’un serveur est un sujet délicat, plus tu as de services qui tournent sur le serveur, plus le nombre de failles potentielles est important. Je te recommande de diviser pour mieux régner, sépares tes services (mails, hébergement, cloud, etc …) sur des serveurs différents. Si l’un des services tombe, les autres subsistent et c’est l’un des avantages des serveurs virtuels.

Autre chose, plus tu as d’utilisateurs sur un serveur, plus le risque de mot de passe « dans la nature » est grand. Les utilisateurs « légaux » de ton serveur sont la plupart du temps les éléments déclencheurs des problèmes. Penses donc à attribuer le strict minimum de droits à tes utilisateurs sur le serveur et formes-les. Mets en place des restrictions sur tes services (exemple : maximum d’envois de mails par heure par utilisateur pour éviter que l’utilisateur puisse servir de relais spam en étant piraté). Obliges tes utilisateurs à utiliser des mots de passe complexes.

Mets à jour tes logiciels le plus souvent possibles !

Tiens-toi informé sur les logiciels que tu utilises pour être au courant des dernières failles, abandon de projet, etc...

Forces l’utilisation de protocoles sécurisés utilisant le cryptage SSL ou TLS (ssh, https, imaps, smtps, etc …)

Tentes de te pirater toi-même, cherches les failles dans ton système, fais une veille sur la sécurité.
