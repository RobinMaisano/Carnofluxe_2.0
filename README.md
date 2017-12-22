# Procédures de l'installation de l'infrastructure

Ce document explique comment a été mise en place l'infrastructure du projet et comment la re-créer.

Dans notre cas de projet, cette infrastructure a été virtualisée. Il se peut donc qu'on rencontre des soucis lors du déploiement de cette infrastructure sur un réseau physique.

Le but de ce projet est de mettre en place un système d’information pour accueillir à terme un site de e-commerce et mettre en place des outils de supervision de ce site.
L'infrastructure doit permettre : 
- la mise en place d’un service de résolution de nom en interne sur le domaine carnofluxe.local. L’objectif est de permettre une gestion des noms d’hôtes sans adresse IP publique dans le réseau privé de l’entreprise
- La gestion des sauvegardes et des remontées d’informations pour assurer un support et une continuité de service de qualité qui sera faite grâce à des scripts bash sur les serveurs sous système d’exploitation GNU/Linux
- La mise en place d’un serveur HTTP et la mise en ligne de 2 sites WEB : un site vitrine (représentant le site de e-commerce) qui sera accessible depuis les 2 interfaces réseaux et un site de supervision accessible uniquement depuis le réseau interne qui intégrera des données.

Pour se faire, nous avons créé 3 serveurs :
- "dns01" : 
			Paquets installés : DNS maitre (Bind9) et DHCP (isc-dhcp-server)
			ip:  192.168.88.5/24
- "dns02" :
			Paquets installés :  DNS esclave (Bind9)
			ip:  192.168.88.6/24
- "srv_apache" :
			Paquets installés :  HTTPd (Apache2)
			ip:  192.168.88.10/24

## Pré-requis pour les 3 machines :

Avant de se lancer dans l'installation et de la configuration des services, il faut commencer par créer la configuration de base de chaques Debian.
Pour se faire il faut commencer par définir pour chaques machines leurs noms :

				nano /etc/hosts 

A la ligne 127.0.1.1, retirer "debian" et le remplacer par son nom machine avec son nom de domaine ex : "dns01.carnofluxe.local"

				nano /etc/hostname 

Modifier "debian" par son nom de machine.
Il faut ensuite rebooter la machine pour que les modifications soient prises en compte:

			reboot

Il faut maintenant configurer l'adresse ip de la machine afin qu'elle puisse échanger avec les autres machines du réseau. Il faut donc ouvrir le fichier de configuration de l'adresse ip:

			nano /etc/network/interfaces 

Puis modifier la configuration pour avoir les paramètres suivants :

			allow-hotplug eth0
			iface eth0 inet static
       		 address 192.168.88.10
       		 netmask 255.255.255.0
       		 gateway 192.168.88.2

Ne pas oublier de changer l'adresse ip par l'adresse ip voulue.

On modifie ensuite la configuration du DNS avec:

			 nano /etc/resolv.conf
 
Il faut ensuite modifier le fichier pour avoir les informations suivantes :

			domain carnofluxe.local
			search carnofluxe.local
			nameserver 192.168.88.5
			nameserver 192.168.88.6

Pour finir la configuration de l'adresse ip il faut relancer le service apache :

			service apache2 restart

Il faut mettre a jour les dépôts APT et les paquets:

			apt-get update && apt-get update && apt-get dist-upgrade

On peut a présent installer SSH pour pouvoir administer les serveurs a distance:

			apt-get install openssh-server

On modifie ensuite le fichier de configuration SSH :

			nano /etc/ssh/sshd_config

On définit les utilisateurs qui peuvent accéder à la connexion à distance. IL faut ajouter la ligne suivante (remplacer 'monUtilisateur" par les utilisateurs voulus) :

			AllowUsers monUtilisateur

Voici pour la configuration de base des 3 machines. On peut donc maintenant passer à l'installation des services voulus.

## Le serveur WEB (srv_apache)

Il faut tout d'abord installer le service Apache :

			apt-get install apache2

On crée ensuite les répertoires qui permettront de contenir les fichiers des VHosts:

			mkdir /var/www/supervision/
			mkdir /var/www/carnofluxe/

On place dans chacun des répertoires un fichier index.html qui nous permettra de voir si on accède bien au site web lors des tests 

			nano /var/www/supervision/index.html
			nano /var/www/carnofluxe/index.html

Dans chacun des index.html il ajouter la page html suivante (penser à remplacer "monSite" ) :

			<!DOCTYPE html>
						<html>
						    <head>
						        <meta charset="utf-8" />
			        <title>Bienvenue</title>
			    </head>
			    <body>    
					Bienvenue sur le site "monSite"
			    </body>
			</html>

Il faut après créer les Vhosts :

			nano /etc/apache2/sites-enabled/carnofluxe.fr.conf
			nano /etc/apache2/sites-enabled/carnofluxe.local.conf
			nano /etc/apache2/sites-enabled/supervision.carnofluxe.local.conf

Pour chaque fichier de configuration il faut ajouter le code suivant (en pensant bien à remplacer les informations):

			<VirtualHost monSite.fr:80>

 			        ServerName  monSite.fr
 			        ServerAlias  monSite.fr
  			       DocumentRoot /var/www/monSite.fr/
					 DirectoryIndex index.html
		 
 			        <Directory /var/www/monSite/>
  			               Require all granted
  			               AllowOverride All
			         </Directory>
			</VirtualHost>

Pour rendre les sites accessibles il faut utiliser la commande suivante:

			a2ensite carnofluxe.fr
			a2ensite carnofluxe.local
			a2ensite supervision.carnofluxe.local

A présent notre serveur web est fonctionnel. Il nous manque plus qu'a installer des outils complémentaire pour les futures scripts.

Pour un script il faut installer "csv2html" et curl. Voici comment faire  :

			apt-get install curl python-setuptools git
			git clone https://github.com/dbohdan/csv2html.git
			python setup.py install

On peut ensuite créer les fichiers csv qui ne seront utiles par la suite:

			touch /srv/log.csv
			touch /srv/state.csv

Pour finir la configuration du serveur web il faut configurer Crontab :

			crontab -e

Crontab va nous permettre de lancer les scripts à des moments bien précis. Pour cela il faut, dans l'interface de configuration, définir quand se lancera le script puis indiquer là ou il se trouve (chemin d'accès).

## Le serveur DNS maitre et DHCP (dns01)

###DHCP :
On commence par installer le service DHCP:

			apt-get install isc-dhcp-server

On modifie ensuite la configuration du DHCP:

			nano /etc/dhcp/dhcpd.conf

Il faut ajouter les lignes suivantes :

			subnet 192.168.88.0 netmask 255.255.255.0{
						range 192.168.88.1 192.168.88.254;
						option domain-name "carnofluxe.local";
						option routers 192.168.88.2;
						option broadcast-address 192.168.88.255;
						option domain-name-servers 192.168.88.5, 192.168.88.6;
						default-lease-time 600;
						max-lease-time 7200;
			}

Dans le cas ou le réseau ne possède pas la même range d'adresse, il faut modifier les informations du fichier de configuration.

On relance ensuite le service DHCP:

			/etc/init.d/isc-dhcp-server restart

On peut vérifier si le dhcp est en marche avec les 2 commandes suivantes :

			tail /var/log/syslog     (observation de la fin du log système)
			ps ax | grep dhcpd      (recherche du processus dhcp )

### DNS :

On commence par installer le DNS :

			apt-get install bind9

On modifie ensuite les fichiers de configurations pour y placer les lignes suivantes (encore une fois les informations contenues dans les lignes sont propre à mon réseau):

			nano /etc/bind/named.conf.local
			
			zone "carnofluxe.local" {
				type master;
				also-notify {192.168.88.6;};
				allow-transfer {192.168.88.6;};
				allow-update { none; };
				allow-query { any; };
				notify no;
				file "/etc/bind/db.carnofluxe.local";
			};

			zone "carnofluxe.fr" {
				type master;
				also-notify {192.168.88.6;};
				allow-transfer {192.168.88.6;};
				allow-update { none; };
				allow-query { any; };
				notify no;
				file "/etc/bind/db.carnofluxe.fr";
			};

			zone "88.168.192.in-addr.arpa" {
				type master;
				also-notify { 192.168.88.6; };
				allow-transfer {192.168.88.6;};
				allow-update { none; };
				allow-query { any; };
				notify no;
				file "/etc/bind/db.88.168.192.in-addr.arpa";
			};
------------------------------
			nano /etc/bind/named.conf.options
			
			auth-nxdomain no;
			recursion no;
			allow-query-cache { none; };
			version none;
			additional-from-cache no;
			allow-transfert { 192.168.88.6; };
			
--------------------------------
			nano /etc/bind/db.carnoflux.local
			
			$TTL	604800
			$ORIGIN carnofluxe.local.
			@	IN	SOA	dns01.carnofluxe.local. root.carnofluxe.local. (
						15		; Serial
						604800		; Refresh
						86400		; Retry
						2419200		; Expire
						604800 )	; Negative Cache TTL
			;
			@	IN	NS	dns01.carnofluxe.local.
			@	IN A	192.168.88.10
			dns01	IN	A	192.168.88.5
			www	IN	A	192.168.88.10
			supervision	IN	A	192.168.88.10
			
----------------
			nano /etc/bind/db.carnoflux.fr
			
			$TTL	604800
			$ORIGIN carnofluxe.fr.
			@	IN	SOA	dns01.carnofluxe.fr. root.carnofluxe.fr. (
						17		; Serial
						604800		; Refresh
						86400		; Retry
						2419200		; Expire
						604800 )	; Negative Cache TTL
			;
			@	IN	NS	dns01.carnofluxe.fr.
			dns01	IN	A	192.168.88.5
			www	IN	A	192.168.88.10

----------------------------
			nano /etc/bind/db.88.168.192.in-addr.arpa
			
			$TTL	604800
			$ORIGIN carnofluxe.fr.
			@	IN	SOA	dns01.carnofluxe.fr. root.carnofluxe.fr. (
						17		; Serial
						604800		; Refresh
						86400		; Retry
						2419200		; Expire
						604800 )	; Negative Cache TTL
			;
			@	IN	NS	dns01.carnofluxe.local.
			5	IN	PTR	dns01.carnofluxe.local.
			

Lorsque tous les fichiers ont été complétés on check que l'on ait pas fait d'erreur :

			named-checkconf -z

Si aucune n'est détectée on peut relancer le service et passer à la configuration de l'autre DNS:

			 service bind9 restart

## Le serveur DNS esclave (dns02)

###DNS :

Encore une fois, on installe bind9 :

			apt-get install bind9

Tout comme l'autre DNS, on doit modifier la configuration et ajouter les lignes suivantes :

			nano /etc/bind/named.conf.local

			zone "carnofluxe.local" {
				type slave;
				masters { 192.168.88.5; };
				file "/var/lib/bind/db.carnofluxe.local";
			};

			zone "carnofluxe.fr" {
				type slave;
				masters { 192.168.88.5; };
				file "/var/lib/bind/db.carnofluxe.local";
			};

			zone "88.168.192.in-addr.arpa" {
				type slave;
				masters { 192.168.88.5; };
				file "/var/lib/bind/db.88.168.192.in-addr.arpa";
			};
--------------------
			nano /etc/bind/named.conf.options
			
			auth-nxdomain no;
			recursion no;
			allow-query-cache { none; };
			version none;
			additional-from-cache no;
			allow-transfert { 192.168.88.6; };
			

On vérifie qu'il n'y ait pas d'erreur :

			named-checkconf -z

Puis on redémarre le service :

			service bind9 restart

Si tout s'est bien passé, les fichiers db. ont été créés dans le dossier :

			/var/lib/bind/



## Ressources:
https://www.supinfo.com/articles/single/1714-mise-place-serveurs-dns-maitre-esclave-avec-bind9
https://wiki.debian.org/fr/DHCP_Server