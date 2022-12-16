
# Networks and Systems Admin.

Petite description du projet 


## Tech

**Gateway:** OpenBSD 7.2 

**Server:** FreeBSD 13.1, use *nginx*, *php74* and *mysql*

        
## VMs Installation

Installer OpenBSD 7.2 sur la VM1, et FreeBSD 13.1 sur la VM2, choix libre pour les VM 3 et 4.

*Si l'installation se fait en continue, il suffit simplement de supprimer l'iso d'install dans Configuration > Stockage sur l'interface Virtual Box*

## VMs Configuration

```
gateway		    > 1 Bridge + 3 Internal Network ( lan-1,2,3 )
server		    > 1 Internal Network ( lan-2 )
admin-client	> 1 Internal Network ( lan-1 )
employee-client	> 1 Internal Network ( lan-3 )
```


## Key files
    Gateway ( OpenBSD )
    /etc/pf.conf                        > Configuration des routes du réseau (firewall, NAT ...)
    /etc/hostname."nom interface"       > Configuration des Networks Cards
    /etc/dhcpd.conf                     > Configuration des IP et des lan

    Server ( FreeBSD )
    /etc/rc.conf                        > Configuration des startup scripts (Php, nginx, mysql)
    /usr/local/etc/nginx/nginx.conf     > Configuration du server nginx
    /usr/local/etc/php-fpm.d/www.conf   > Configuration du user www
    /                                   > Configuration pour mysql


# Gateway Configuration

## Configuration des Networks Cards
##### *file location : /etc/hostname."nom interface"*

### Paramétrer les fichiers :
    hostname.em0
	    - inet autoconf

	hostname.em1,2,3
		- inet 192.168.42."à modifier selon l'interface" 255.255.255.192

### Lancer les network cards : 

	sh /etc/netstart ("nom interface")

## Configuration DHCP
##### *file location : /etc/dhcpd.conf*

### Paramétrer le fichier :
##### Création des lan et de l'adresse fixe du server
![Domain name](/Screen%20Projet%20R%C3%A9seau/Configuration%20NAT.png)
![fichier dhcp](/Screen%20Projet%20R%C3%A9seau/Param%C3%A9trage%20addresse%20fixe%20server.png)
	
### Lancer/Stopper le processus dhcp : 
    rcctl start/stop dhcpd
	
	dhcpd

*En cas de problème d'IP avec le server ( S'il a été modifié, recréé ... et qu'il n'a plus la bonne adresse IP), supprimer les anciennes données du fichier dhpcd.leases peut permettre de résoudre le problème.*

	echo "" > /var/db/dhcpd.leases

## Configuration NAT et Firewall
##### *file location : /etc/pf.conf*

### Paramétrage du fichier pf.conf
![fichier pf.conf 1](/Screen%20Projet%20R%C3%A9seau/pf.conf%20part%201.png)
![fichier pf.conf 2](/Screen%20Projet%20R%C3%A9seau/pf.conf%20part%202.png)
![fichier pf.conf 3](/Screen%20Projet%20R%C3%A9seau/pf.conf%20part%203.png)

### Activer le forwarding :
	sysctl net.inet.ip.forwarding=1
			or
	echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf


    => Permet à la gateway de rediriger le message vers
	l'extérieur, et n'est plus bloqué dans son interface de base.
### Actualiser la config :
	pfctl -f pf.conf

### Explications fichier pf.conf :
------Explication générale------
    
    Le fichier se lit de haut en bas, les dernières règles qui seront lus sont celles se trouvant au plus bas du fichier 
    ( à part si mot clé quick mais non utilisé dans notre cas )

    Une règle se trouvant en bas du fichier peut donc override une se trouvant au dessus,

    Exemple : 
	    - pass out on em0 from 192.168.42.0/26 to any nat-to em0
	    - pass out on any proto icmp from { em1:network, em2:network, em3:network, em0 }

    Ici on ajoute une règle pour le ping ( icmp ), any vise en effet toutes les interfaces réseaux, 
    cependant cette règle s'applique aussi pour la sortie de em0 et override la première règle en enlevant le "nat-to em0" 
    ( qui n'est pas précisé dans la seconde règle ), ce qui fait qu'on aurait plus de nat et donc plus de connection internet avec les machines clientes.

    C'est pour cela qu'on a du préciser les interfaces en omettant em0 dans la règle pour les pings :
    pass out on { em1, em2, em3 } proto icmp from { em1:network, em2:network, em3:network, em0 }
  
------Explication commande------

    - pass = règle autorisant le traffic

    - block = règle bloquant le traffic

    - out = en sortie de l'interface

    - in = en entrée de l'interface 

    - on = pour telle interface

    - from = provenance du packet

    - to = destination du packet

    - nat-to <adresse ip> = Translation d'adresse vers telle adresse ip, nécessaire pour la connection internet des machines clientes

    - proto = protocole de transport ( attention ) utilisé, c'est à dire soit tcp soit udp soit icmp

    - <interface>:network = toutes les adresses ip correspondant au réseau de l'interface, 
    Exemple em1:network va prendre toutes les adresses ip se trouvant sur le même réseau que em1 ( 192.168.42.0/26 dans notre cas )

    - to ! <adresse_ip> = à destination de toutes les adresses ip sauf celle précisée, le ! s'applique à presque toutes les règles ( from, on, etc .. )
 
    - table <variablequetuveux> const { adresse_ip1, adresse_ip2 } = table permet de stocker une liste d'adresse ip dans une variable, 
    const ici indique qu'on ne peut pas modifier cette variable, 
    enfin on met les adresses ip entre accolades. Le mot clé self  
    quant à lui indique qu'il faut prendre toutes les adresses ip 
    des interfaces de notre firewall ( ici gateway )



# Server Configuration

## Installation des packages 
### Nginx
    pkg install nginx
### PHP 7.4
    pkg install php74
### MySQL
    pkg install mysql80-server mysql80-client

## Activer les startup scripts
#### *file location : /etc/rc.conf*

*Tips : Commande pour trouver les rcvar de nos services permettant de les enable*
    
    grep rcvar /usr/local/etc/rc.d/*
![Example commande rcvar](/Screen%20Projet%20R%C3%A9seau/Rcvar%20variables.png)
Nginx : 
    
    sysrc nginx_enable=YES
			or 
	echo 'nginx_enable="YES"' >> /etc/rc.conf
	    ---> /usr/local/etc/rc.d/nginx start

Php74 : 
    
    sysrc php_fpm_enable=YES
			or 
    echo 'php_fpm_enable="YES"' >> /etc/rc.conf
	    ---> /usr/local/etc/rc.d/php-fpm start

Mysql : 
    
    sysrc mysql_enable=YES
			or 
	echo 'mysql_enable="YES"' >> /etc/rc.conf
	    ---> /usr/local/etc/rc.d/mysql start
Fichier rc.conf
![fichier rc.conf](/Screen%20Projet%20R%C3%A9seau/Enable.png)

## Configuration php-fpm

### 1. www.conf
##### *file location :  /usr/local/etc/php-fpm.d/www.conf*
	
	a. Remplacer le paramètre :
	    Listen = /var/run/php-fpm.sock

	b. Enable listens configuration :
		Uncomment les ; pour : 
			listen.owner = www
			listen.group = www
			listen.mode = 0660

### 2. php.ini
	
	a. Copy the php.ini-production to php.ini :
		> cp php.ini-production php.ini

##### *file location : /usr/local/etc/php.ini*

	b. Find : "cgi.fix_pathinfo=1" :
		> Uncomment ( remove ; )
		> Set 1 to 0
( This will prevent PHP from trying to execute parts of the path if the file that was passed in to process is not found )

	c. Start PHP-FPM
		> service php-fpm start


## MySQL Configuration


## Nginx Web Server Configuration
##### *file location : /usr/local/etc/nginx/nginx.conf*

    a. Lancer le service nginx :
		
    service nginx start
	
	a_00. ligns location : Premières lignes

		Uncomment : "user ;" ( Première ligne )
		worker_processes "nombre de CPU"; ( sysctl hw.ncpu pour check le nombre de cpu )
	
	a_01. ligns location : Block server

		Uncomment and adjust the "location ~\.php$ {}" part

![Location / php](/Screen%20Projet%20R%C3%A9seau/Server%20block.png)
Restart le server :
	    
        service nginx restart
## Authors

- [@David Grondin](https://github.com/divatchyanoo)
- [@Enki Fouque](https://github.com/Enkiiii)

