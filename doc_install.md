# Projet UF Réseau
Ynov Informatique B2A - Repository du projet UF Réseaux - Thomas Le Naour / Alex Boisseau

## Mise en place et sécurisation du VPS

* Mettre à jour le système `apt-get update` && `apt-get upgrade`.
* Changer le port (pour éviter les attaques "en masse" sur le port de base qui est le port 22) : décommenter la ligne dans le fichier `/etc/ssh/sshd_config` et changer le port.
* Redémarrer le service ssh `/etc/init.d/ssh restart`.  
* Pour se connecter en ssh au VPS il faudra rajouter une option où on spécifie le port choisi (`ssh root@IPDUSERVEUR -p PORT`). 
* Changer le mot de passe de l'utilisateur root (le seul pour le moment) -> `passwd root`. (Ne jamais se connecter avec l'utilisateur `root` après avoir créé son premier utilisateur.)
* Créer le nouvel utilisateur `adduser NOUVELUTILISATEUR`. Rentrer les données qui sont demandées et choisir un mot de passe(mélanger les caractères etc...)
* Empecher l'utilisateur `root` de se connecter au serveur via protocole ssh. Pour cela, on se rend dans le fichier `/etc/ssh/sshd_config`. A la ligne `PermitRootLogin` et mettre `no`.
* Redémarrer le service ssh `/etc/init.d/ssh restart`. 
* Installer le logiciel `Fail2Ban`. Ce logiciel permet de "stocker" les adresses IPs autorisées à acceder au serveur. Pour l'installer -> `apt-get install fail2ban`.
* Configurer `fail2ban` : se rendre ans le fichier (normalement vide) `/etc/fail2ban/jail.d/defaults-debian.conf` puis y ajouter

```
[sshd]
enabled = true
maxretry = 5
findtime = 180
bantime = 3600
```

* Ajouter un utilisateur au groupe `sudo` (permet à cet utilisateur d'utiliser sudo.) -> `usermod -a -G sudo NOMUTILISATEUR`.  
* On va maintenant générer une paire de clé ssh pour sécuriser les connexions. Nous devons donc créer une paire sur notre ordinateur `ssh-keygen` puis mettre notre clé public sur le serveur `ssh-copy-id user@ip_server`.
* A partir de ce moment les connexion via paires de clés (privés/public) sont possibles. On enlève la possibilité de se connecter au serveur via login/mdp dans le fichier /etc/ssh/sshd_config en passant la ligne `PasswordAuthentification` à `no`.

## Mise en place du Reverse Proxy

Le reverse Proxy que nous allons mettre en place va permettre d'optimiser les performances de notre serveur en placant un autre serveur "intermediaire" entre le client et notre serveur. Ce serveur intermediaire sera (configuré sous Nginx) et pourra stocker des fichiers statiques en cache ce qui permettra à notre serveur (configuré sous apache) d'économiser ses ressources. Le serveur intermediaire sera accessible à partir du port 80 sur notre adresse IP tandis que notre serveur sera uniquement accessible en local sur le port 8080. 


### Etape 1 : Installation et configuration de Apache

* Installer Apache : `apt-get install apache2 -y`
* Démarrer Apache et l'activer : `systemctl start apache2` et `systemctl enable apache2`.
* Par défault, Apache écoute sur le port 80, on va le configurer pour qu'il écoute sur le port 8080 :    
    * `vi /etc/apache2/ports.conf`
    * Trouver la ligne `Listen 80`
    * La remplacer par `Listen 127.0.0.1:8080`
* Redémarrer Apache : `systemctl restart apache2`
* Pour vérifier que les modifications ont bien été prises en compte on éxecute la commande `sudo ss -ltpn`. Si apache écoute sur `127.0.0.1:8080`, les modifications ont bien été prises en compte.
* Lors de l'ajout d'un site, on commence par se rendre dans la conf d'apache `/etc/apache2/sites-available` pour y ajouter le fichier `monsite.com.conf`.
    * On commence par créer un `virtualhost`qui écoute sur l'IP `127.0.0.1` et sur le port `8080`.
    * On ajoute ensuite le `ServerName` : `www.monsite.com`
    * Puis le `ServerAlias` : `monsite.com`
    * Le chemin du contenu : `/var/www/monsite.com`
    * Les logs d'Apache

    ```
    <VirtualHost 127.0.0.1:8080>
        ServerName www.alexboisseau.com
        ServerAlias alexboisseau.com
        DocumentRoot /var/www/boiss_bseau/alexboisseau.com
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```
    * Activer ce site `sudo a2ensite monsite.com.conf` puis reload apache `sudo systemctl reload apache2`.
    * (Penser à reload Apache à chaque modification !)




* Créer un dossier par utilisateur dans le dossier `/var/www/`. Ces dossiers contiendront les sites des utilisateurs.


### Etape 2 : Installation et configuration de Nginx

* Installer NGINX : `sudo apt-get install nginx`
* Démarrer NGINX et l'activer : `systemctl start nginx` et `systemctl enable nginx`.
* A l'ajout d'un site on va se rendre dans le dossier `/etc/nginx/sites-available` pour y créer le fichier `monsite.com.conf`.
    * Au sein de ce fichier on va ajouter un block `server` qui doit contenir les infos suivantes : 
    ``` 
    server {
    server_name alexboisseau.com www.alexboisseau.com;

        location / {
            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
    * Créer un lien symbolique dans le dossier `sites-enabled` avec la commande `sudo ln -s /etc/nginx/sites-available/monsite.com.conf /etc/nginx/sites-enabled`.
    * Une fois fait, ajouter la certification SSL avec la commande `sudo certbot --nginx -d monsite.com -d www.monsite.com`.

### Etape 3 : Installation de MariaDB et de PHP pour pouvoir héberger des sites dynamiques

* Pour pouvoir faire fonctionner des sites dynamiques on va avoir besoin d'installer php et mariaDB pour récuperer de la data à la volée et l'inserer dans des modèles HTML prédéfinies.

#### Installation de PHP : 

* Exécuter la commande : `sudo apt-get install php libapache2-mod-php`

#### Installation de MariaDB : 

* Exécuter la commande : `sudo apt install mariadb-server`
* Création de 2 utilisateurs avec tous les droits (dans MariaDb): 
```
CREATE USER 'newuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON * . * TO 'newuser'@'localhost';
FLUSH PRIVILEGES;
exit; 
```
### Etape 4 : Mise en place de la back-up (Borg)

* Installation du paquet `sudo apt install borgbackup`
* Installation de PIP pour Borgmatic `sudo apt install python3-pip python3-setuptools`
* Installation de Wheel pour Borgmatic `sudo pip3 install wheel`
* Installation de Borgamatic `pip3 install --user --upgrade borgmatic`
* On génère la configuration de base qu'on va modifier avec `sudo env "PATH=$PATH" generate-borgmatic-config`
* Création d'un repository sur BorgBase, associé à une paire de clef SSH qu'on a générer sur le VPS
* Initilisation de la backup sur BorgBase `sudo env "PATH=$PATH" borgmatic init --encryption repokey-blake2`
* Création d'une backup : `sudo env "PATH=$PATH" borgmatic --verbosity 1`
* Initilisation d'une crontab