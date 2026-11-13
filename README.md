# Compte rendu TP 2 : installer Nginx à partir des sources

Le but de ce TP est d'installer le serveur web Nginx en le compilant nous-mêmes à partir de son code source, au lieu de passer par le paquet Debian. Compiler soi-même permet de choisir exactement ce qu'on installe : le dossier d'installation, l'utilisateur sous lequel tourne le serveur et les modules qu'on garde ou qu'on enlève.

## Mise à jour du système

On commence par mettre à jour tout le système, y compris le noyau, avec :

```
apt-get dist-upgrade
```

Contrairement à `apt-get upgrade`, qui met à jour les paquets installés sauf le noyau, `dist-upgrade` met tout à jour, y compris le noyau. On a maintenant le nouveau paquet linux-image. On vérifie la version du noyau avec `uname -a` :

```
eleve@master-tp:~$ uname -a
Linux master-tp 6.12.115+deb13.1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.115-2 (2026-11-01) x86_64 GNU/Linux
```

On voit le nom de la machine (master-tp), la version du noyau (6.12.115) et l'architecture (x86_64).

## 2. Téléchargement de Nginx

Ensuite on télécharge les sources de Nginx depuis le site officiel :

```
wget https://nginx.org/download/nginx-1.30.6.tar.gz
```

On récupère une archive `.tar.gz` d'environ 1,3 Mo, qui contient tout le code source de Nginx.

## 3. Décompression de l'archive

Puis on désarchive et on décompresse l'archive :

```
tar -xzf nginx-1.30.6.tar.gz
```

Le `.tar.gz` est en fait deux choses : une archive tar qui regroupe tous les fichiers en un seul, puis une compression gzip. Dans la commande, `-x` sert à extraire, `-z` à décompresser le gzip et `-f` à indiquer le fichier. On obtient un dossier `nginx-1.30.6` avec les sources.

## 4. Installation de build-essential

Pour compiler, il faut gcc et make. On les installe via le paquet build-essential, qui contient gcc, make, etc. :

```
apt install build-essential
```

gcc est le compilateur qui transforme le code C de Nginx en exécutable, et make lance la compilation en suivant les instructions du Makefile.

## 5. Dossier des sources

On se place dans le dossier des sources :

```
cd nginx-1.30.6
```

## 6. Options de compilation

On affiche toutes les options de compilation disponibles :

```
root@master-tp:~/nginx-1.30.6# ./configure --help
```

La liste est longue, mais on s'intéresse surtout à trois options :

- `--prefix` : le dossier où Nginx sera installé ;
- `--user` : l'utilisateur non privilégié sous lequel tournent les processus worker ;
- `--group` : le groupe non privilégié de ces mêmes processus.

L'intérêt de `--user` et `--group` est que les processus qui traitent les requêtes ne tournent pas en root. Si quelqu'un exploite une faille dans Nginx, il n'aura que les droits de cet utilisateur limité.

## 7. Génération du Makefile

On configure le Makefile avec le script configure :

```
./configure --prefix=/opt/nginx-1.30.6 --user=nginx --group=nginx --without-http_rewrite_module --without-http_gzip_module
```

Le script vérifie que tout ce qu'il faut est présent sur la machine, puis génère le Makefile avec nos options. Ici, Nginx sera installé dans `/opt/nginx-1.30.6` et tournera sous l'utilisateur et le groupe nginx. Les options `--without-...` désactivent deux modules : le module rewrite (réécriture d'URL) et le module gzip (compression des réponses). Ils ne sont pas utiles pour ce TP, et ça évite d'installer les bibliothèques dont ils ont besoin.

## 8. Compilation

Maintenant on compile tout le code avec `make` :

```
make
```

make lit le Makefile généré à l'étape précédente et compile tous les fichiers sources un par un pour obtenir l'exécutable nginx.

## 9. Installation

Puis on l'installe avec `make install` :

```
make install
```

Cette commande copie l'exécutable et les fichiers de configuration dans le dossier choisi avec `--prefix`, c'est-à-dire `/opt/nginx-1.30.6`.
