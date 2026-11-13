# Compte rendu TP2 : Préparer la VM pour compiler un noyau Linux

Dans ce TP on prépare l'environnement pour compiler un noyau Linux 6.17 dans une VM Debian 13 sous VirtualBox. L'idée est de mettre tout ce qui est « build » sur un **deuxième disque** monté sur `/build`, pour ne pas remplir le disque système. Ce compte rendu couvre les étapes 1 à 5 du TP (jusqu'au lancement de `menuconfig`).

## 1. Importer la VM et se connecter en SSH

J'ai importé `Debian13-base.ova` (environ 970 Mo, fournie par le prof) dans VirtualBox.

La VM est en réseau **NAT** : elle n'est pas joignable directement depuis ma machine. Pour s'y connecter en SSH, on ajoute une **règle de redirection de port** (Configuration > Réseau > Avancé > Redirection de ports) :

| Port hôte | Port invité |
|---|---|
| 2424 | 22 |

Ça veut dire : quand un paquet arrive sur le port 2424 de ma machine, il est redirigé vers le port 22 (SSH) de la VM. Ensuite, depuis un terminal **de ma machine** (PowerShell sous Windows, pas dans la fenêtre de la VM) :

```
ssh -p 2424 user@127.0.0.1
```

`127.0.0.1` c'est ma propre machine, et c'est la redirection qui envoie la connexion vers la VM.

> Petit souci au début : j'avais noté `eleve` comme nom d'utilisateur (celui du TP1), mais sur cette VM le compte s'appelle `user` (j'ai vérifié avec `whoami` dans la fenêtre de la VM). Le mot de passe est `azerty`.

## 2. Ajouter un deuxième disque

Il faut ajouter le disque **VM éteinte** : on ne peut pas brancher un disque à chaud dans une machine en exécution.

Dans Configuration > Stockage, sur le contrôleur SATA, j'ai fait *Add Hard Disk* puis *Create* : format VDI, allocation dynamique, **60 Go**. Le fichier `.vdi` représente le disque dur, donc la VM a maintenant 2 disques. Pour l'instant il est inutilisable : il faut démarrer la VM, puis le partitionner et le formater.

## 3. Repérer le bon disque

C'est l'étape où il ne faut pas se tromper, parce qu'on va partitionner et formater un disque.

```
user@debian:~$ lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
NAME    SIZE TYPE MOUNTPOINT
sda      30G disk
├─sda1 27.9G part /
└─sda2  2.1G part [SWAP]
sdb      60G disk
sr0    1024M rom
```

Le disque système (avec `/` et `[SWAP]`) est `sda`, et le nouveau disque (60G, sans partition ni point de montage) est `sdb`. Le sujet du TP écrit `/dev/sdb` en dur, mais l'ordre des noms dépend de la VM : pendant le cours, sur la VM du prof, c'était l'inverse. Recopier les commandes sans regarder aurait pu formater le disque système. La règle que j'ai suivie : **le disque à formater est celui sans partition et sans point de montage**.

## 4. Partitionner, formater, monter

On installe `parted`, on crée une table de partitions GPT, puis une partition qui commence à 1 MiB et prend tout le disque :

```
sudo apt-get install -y parted
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart build ext4 1MiB 100%
```

Résultat :

```
NAME    SIZE TYPE MOUNTPOINT
sda      30G disk
├─sda1 27.9G part /
└─sda2  2.1G part [SWAP]
sdb      60G disk
└─sdb1   60G part
sr0    1024M rom
```

Ensuite on crée le système de fichiers ext4 (avec le label `KBUILD`), on crée le dossier `/build` et on y monte la partition (le montage raccroche un chemin à la partition) :

```
sudo mkfs.ext4 -L KBUILD /dev/sdb1
sudo mkdir /build
sudo mount /dev/sdb1 /build
df -h /build
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        59G  2.1M   56G   1% /build
```

On voit bien le point de montage. J'ai fait à la main ce que fait l'installateur dans le TP1.

## 5. Montage automatique avec `/etc/fstab`

Pour que `/build` soit monté à chaque démarrage, il faut l'ajouter dans `/etc/fstab` (qui dit quoi monter et où). On récupère l'UUID de la partition :

```
user@debian:~$ sudo blkid /dev/sdb1
/dev/sdb1: LABEL="KBUILD" UUID="86a4816f-e217-4ff0-b368-bd0493fb120c" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="build" PARTUUID="b7b7b48b-d8ff-4d35-9719-43f308fce36f"
```

On utilise l'**UUID** plutôt que `/dev/sdb1` parce que les noms `sda`/`sdb` peuvent changer si on déplace ou ajoute des disques, alors que l'UUID est donné à la création de la partition et ne bouge plus. Ligne ajoutée :

```
echo 'UUID=86a4816f-e217-4ff0-b368-bd0493fb120c  /build  ext4  defaults,noatime  0  2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown -R $USER:$USER /build
```

Le `chown` sert à ce que la compilation se fasse en simple utilisateur, pas en root. Après un `sudo reboot`, `/build` était bien remonté tout seul :

```
user@debian:~$ df -h /build
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        59G  2.1M   56G   1% /build
```

> Problème au premier redémarrage : la VM est restée bloquée sur un écran **iPXE** (démarrage par le réseau), ce qui veut dire qu'elle n'avait pas trouvé de disque sur lequel démarrer. J'avais enregistré l'état de la machine, donc elle reprenait au même endroit. J'ai éteint la VM, vérifié que le disque système était bien sur le port SATA 0 et le nouveau sur le port 1, mis **Disque dur** en premier dans l'ordre d'amorçage (Système > Carte mère), puis relancé. Ça a démarré normalement.

## 6. Mettre à jour le système et installer les prérequis

La commande d'installation des outils de compilation a d'abord échoué avec des erreurs 404 :

```
Err:5 http://debian.obspm.fr/debian trixie/main amd64 rsync amd64 3.4.1+ds1-5
  404  Not Found [IP: 145.238.186.56 80]
```

La liste des paquets de la VM était périmée : les versions qu'elle connaissait n'existaient plus sur les serveurs. Il fallait d'abord mettre à jour (c'est la commande du TP1 : `update` puis `upgrade`) :

```
sudo apt-get update
sudo apt-get -y upgrade
```

Pendant l'upgrade, un écran bleu de configuration de GRUB propose une liste de disques. J'ai coché **`/dev/sda`** : GRUB s'installe dans le MBR du disque entier du système (pas dans la partition `sda1`), et surtout pas `sdb` qui est mon disque de build.

Ensuite, l'installation des prérequis est passée (73 nouveaux paquets, sans erreur) :

```
sudo apt-get -y install bc bison flex libelf-dev libssl-dev libncurses-dev dwarves fakeroot rsync ccache build-essential
```

Puis on active `ccache` (un cache de compilation) en le mettant en tête du `PATH` :

```
echo 'export PATH="/usr/lib/ccache:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

```
user@debian:/build$ echo $PATH
/usr/lib/ccache:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

## 7. L'archive `linux-build.tar.gz` du prof

Compiler tout le noyau prendrait plusieurs heures, trop pour la séance. Le prof nous a donc distribué `linux-build.tar.gz` (environ 7,5 Go), qui contient un dossier de build **déjà compilé**.

L'archive vient de Linux, donc je ne l'ai pas extraite sous Windows (pas de droits ni de liens symboliques Linux). Je l'ai copiée dans la VM avec `scp`, qui copie des fichiers en s'appuyant sur SSH (`scp source destination`). Depuis PowerShell, dans le dossier de l'archive :

```
scp -P 2424 linux-build.tar.gz user@127.0.0.1:/build/
```

(`-P` en majuscule pour `scp`, alors que `ssh` utilise `-p`.) Dans la VM, je l'ai extraite dans `/build`, puis supprimé l'archive pour récupérer 7 Go :

```
cd /build
tar -xzf linux-build.tar.gz
rm /build/linux-build.tar.gz
```

Vérification (`tar -tzf` avait montré que tous les chemins commencent par `linux-build/`, donc on obtient bien `/build/linux-build`) :

```
user@debian:/build$ ls /build/linux-build | head
arch
block
built-in.a
certs
crypto
drivers
fs
include
init
io_uring
user@debian:/build$ du -sh /build/linux-build
24G     /build/linux-build
```

## 8. Télécharger les sources du noyau

```
mkdir -p ~/src && cd ~/src
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.17.tar.xz
tar -xf linux-6.17.tar.xz
cd linux-6.17
```

```
user@debian:~/src/linux-6.17$ ls | head
arch
block
certs
COPYING
CREDITS
crypto
Documentation
drivers
fs
include
user@debian:~/src/linux-6.17$ pwd
/home/user/src/linux-6.17
```

Les **sources** sont dans `~/src` et les **fichiers de compilation** dans `/build/linux-build` : c'est la séparation voulue par le TP.

## 9. Préparation de la configuration (début de l'étape 5)

```
mkdir -p /build/linux-build /build/tmp
export TMPDIR=/build/tmp
make O=/build/linux-build menuconfig
```

- `O=...` : tous les fichiers générés vont dans ce dossier, ça évite de polluer le dossier des sources.
- `TMPDIR` : les fichiers temporaires vont aussi sur le disque de build. Cette variable ne dure que le temps de la session, donc il faut la refaire à chaque nouvelle connexion SSH.
- `menuconfig` : une interface dans le terminal pour choisir ce qu'on compile (la suite est dans le TP3).
