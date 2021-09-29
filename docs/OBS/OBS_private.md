---
title: "OBS private instance"
date: 2021-09-15
draft: true
---

# OBS private instance Installation

## Installation / Mise à jour

A faire tout de suite après l'installation

- créer un user + clé ssh
- installer sudo + configuration NOPASSWD pour le user créé
- désactiver la connexion du user root (ssh)
- activer le service ssh
- changer les versions des repositories + zypper dup

on reboot pour valider tous les changements.

## Filesystems

```
nuc:~ # df -hP
Filesystem              Size  Used Avail Use% Mounted on
devtmpfs                4.0M     0  4.0M   0% /dev
tmpfs                    16G     0   16G   0% /dev/shm
tmpfs                   6.3G  9.3M  6.3G   1% /run
tmpfs                   4.0M     0  4.0M   0% /sys/fs/cgroup
/dev/nvme0n1p1           98G  2.8G   91G   3% /
/dev/mapper/OBS-server   49G  2.6G   44G   6% /srv/obs
/dev/mapper/OBS-cache    25G   45M   24G   1% /var/cache/obs/worker
tmpfs                   3.2G     0  3.2G   0% /run/user/1000
```

LVM pour les workers, ici 4. Les LVs des workers seront montés pour être utilisé pour les builds.

```
nuc:~ # lvs -a -o +devices
  LV            VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Devices
  cache         OBS -wi-ao----  25.00g                                                     /dev/sda1(40416)
  server        OBS -wi-ao----  50.00g                                                     /dev/sda1(0)
  worker_root_1 OBS -wi-a-----  26.47g                                                     /dev/sda1(12800)
  worker_root_2 OBS -wi-a-----  26.47g                                                     /dev/sda1(19704)
  worker_root_3 OBS -wi-a-----  26.47g                                                     /dev/sda1(26608)
  worker_root_4 OBS -wi-a-----  26.47g                                                     /dev/sda1(33512)
  worker_swap_1 OBS -wi-a----- 512.00m                                                     /dev/sda1(19576)
  worker_swap_2 OBS -wi-a----- 512.00m                                                     /dev/sda1(26480)
  worker_swap_3 OBS -wi-a----- 512.00m                                                     /dev/sda1(33384)
  worker_swap_4 OBS -wi-a----- 512.00m                                                     /dev/sda1(40288)
```

## Users management

### Création d'utilisateurs

Créer un user via l'interface web https://nuc.mon-lab.org

#### Donner le droit d'administration OBS

Pour donner les droits d'admin a un user :

```
nuc:~ # cd /srv/www/obs/api
nuc:/srv/www/obs/api # bundle exec rake user:give_admin_rights mfatrez RAILS_ENV=production
Making user 'mfatrez' an admin
```

## Ajouter une distribution

récupérer le fichier [obs_mirror_project](https://raw.githubusercontent.com/openSUSE/open-build-service/master/dist/obs_mirror_project)

un exemple de obs_mirror_project est présent dans le doc "Build Service Concept CrossDevelopment"


à lancer avec le user root :

```
nuc:~ # obs_mirror_project -p "openSUSE:Leap:15.3" -r "standard" -a "x86_64" -d "/srv/obs/build"
```

/!\
peut être il faudrait changer les droits des fichiers dans /srv/obs/build. De mémoire, il faut mettre obsrun:obsrun pour les fichiers/répertoires.
/!\

Tester la commande

```
/usr/lib/obs/server/bs_admin --rescan-repository "openSUSE:Leap:15.3" "standard" "x86_64"
```

```
mfatrez@nuc:~> osc -A https://nuc.mon-lab.org api /distributions -e
<status code="ok">
  <summary>Ok</summary>
</status>
mfatrez@nuc:~> osc -A https://nuc.mon-lab.org api /distributions
<distributions>
  <distribution vendor="openSUSE" version="15.3" id="1">
    <name>openSUSE Leap 15.3</name>
    <project>openSUSE:Leap:15.3</project>
    <reponame>15.3</reponame>
    <repository>standard</repository>
    <link>http://www.opensuse.org/</link>
    <icon url="https://static.opensuse.org/distributions/logos/opensuse.png" width="8" height="8"/>
    <icon url="https://static.opensuse.org/distributions/logos/opensuse.png" width="16" height="16"/>
    <architecture>x86_64</architecture>
  </distribution>
</distributions>
```
