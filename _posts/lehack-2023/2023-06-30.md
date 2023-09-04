---
layout: post
title: 48 Meters underground
category: Lehack-2023
---

# 48 meters underground

Nous récupérons un fichier `firmware.bin` :

```bash
$ file firmware.bin 
firmware.bin: Linux kernel ARM boot executable zImage (big-endian)
```
Grâce à la commande `binwalk`, nous pouvons identifier que cette image contient un système de fichiers Squashfs. Nous extrayons alors cette image, toujours avec l'outil `binwalk` :

```bash
$ binwalk -e firmware.bin 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Linux kernel ARM boot executable zImage (big-endian)
14419         0x3853          xz compressed data
14640         0x3930          xz compressed data
538952        0x83948         Squashfs filesystem, little endian, version 4.0, compression:xz, size: 2068482 bytes, 995 inodes, blocksize: 262144 bytes, created: 2022-05-03 12:34:33
```

Le contenu est accessible dans le répertoire créé `_firmware.bin.extracted` :
```bash
$ ls -l        
-rw-r--r--  1 kali kali 16777216  2 juil. 01:06 firmware.bin
drwxr-xr-x 75 kali kali    36864 19 juil. 09:32 _firmware.bin.extracted
```

En parcourant ce répertoire, nous identifions la présence d'un sous-répertoire `squashsf_root`, qui correspond au système de fichiers :


```bash
$ ls -l _firmware.bin.extracted/

-rw-r--r--  1 kali kali      244  9 mars   2021  00-netstate
-rw-r--r--  1 kali kali      338  9 mars   2021  00_preinit.conf
-rw-r--r--  1 kali kali      236  9 mars   2021  00-sysctl
...
...
...
-rw-r--r--  1 kali kali     5756  9 mars   2021  slhc.ko
-rw-r--r--  1 kali kali       17  3 mai    2022  sort
drwxr-xr-x 16 kali kali     4096 19 juil. 09:32  squashfs-root
drwxr-xr-x  2 kali kali     4096 19 juil. 09:32  squashfs-root-0
-rw-r--r--  1 kali kali       16  3 mai    2022  ssh


$ ls -l _firmware.bin.extracted/squashfs-root/
total 56
drwxr-xr-x  2 kali kali 4096  9 mars   2021 bin
drwxr-xr-x  2 kali kali 4096 20 févr.  2017 dev
drwxr-xr-x 18 kali kali 4096 19 juil. 09:32 etc
drwxr-xr-x 11 kali kali 4096  9 mars   2021 lib
drwxr-xr-x  2 kali kali 4096 20 févr.  2017 mnt
drwxr-xr-x  2 kali kali 4096 20 févr.  2017 overlay
drwxr-xr-x  2 kali kali 4096 20 févr.  2017 proc
drwxr-xr-x  2 kali kali 4096 20 févr.  2017 rom
drwxr-xr-x  2 kali kali 4096 20 févr.  2017 root
drwxr-xr-x  2 kali kali 4096  9 mars   2021 sbin
drwxr-xr-x  2 kali kali 4096 20 févr.  2017 sys
drwxrwxrwt  2 kali kali 4096 20 févr.  2017 tmp
drwxr-xr-x  7 kali kali 4096 20 févr.  2017 usr
lrwxrwxrwx  1 kali kali    9 19 juil. 09:32 var -> /dev/null
drwxr-xr-x  4 kali kali 4096 20 févr.  2017 www
```
Nous parcourons les différents fichiers/répertoires à la recherche d'informations sensibles. Le dossier `scripts`, qui contient un script `telnetd.sh`, attire notre attention :

```bash
$ ls -l _firmware.bin.extracted/squashfs-root/etc/           
-rw-r--r-- 1 kali kali  441  9 mars   2021 banner
-rw-r--r-- 1 kali kali  408  9 mars   2021 banner.failsafe
drwxr-xr-x 2 kali kali 4096 20 févr.  2017 board.d
drwxr-xr-x 2 kali kali 4096  3 mai    2022 config
drwxr-xr-x 2 kali kali 4096 20 févr.  2017 crontabs
...
...
...
lrwxrwxrwx 1 kali kali    9 19 juil. 09:32 resolv.conf -> /dev/null
drwxr-xr-x 2 kali kali 4096 11 mars   2021 scripts
-rw-r--r-- 1 kali kali 3017  9 mars   2021 services
-rw------- 1 kali kali  140 11 mars   2021 shadow
-rw-r--r-- 1 kali kali    9  9 mars   2021 shells
-rw-r--r-- 1 kali kali  896  9 mars   2021 sysctl.conf
drwxr-xr-x 2 kali kali 4096 20 févr.  2017 sysctl.d
-rw-r--r-- 1 kali kali  128  9 mars   2021 sysupgrade.conf
lrwxrwxrwx 1 kali kali    9 19 juil. 09:32 TZ -> /dev/null
drwxr-xr-x 2 kali kali 4096 20 févr.  2017 uci-defaults


$ ls -l _firmware.bin.extracted/squashfs-root/etc/scripts 
-rw-r--r-- 1 kali kali 318 11 mars   2021 telnetd.sh
```
En analysant le script `telnetd.sh`, nous identifions que des identifiants de connexion sont présents via le paramètre `-u` de la commande `telnetd`, sous la forme `username:password`. Ici, les informations identifiées sont les suivantes :
* Nom d'utilisateur : `Device_Admin`
* Mot de passe : contenu du fichier `/etc/config/sign`

```bash
$ cat _firmware.bin.extracted/squashfs-root/etc/scripts/telnetd.sh        
#!/bin/sh
sign=`cat /etc/config/sign`
TELNETD=`rgdb
TELNETD=`rgdb -g /sys/telnetd`
if [ "$TELNETD" = "true" ]; then
        echo "Start telnetd ..." > /dev/console
        if [ -f "/usr/sbin/login" ]; then
                lf=`rgbd -i -g /runtime/layout/lanif`
                telnetd -l "/usr/sbin/login" -u Device_Admin:$sign      -i $lf &
        else
                telnetd &
        fi
fi
```

La lecture du fichier `etc/config/sign` nous permet alors d'obtenir le mot de passse du compte `Device_Admin`, et le flag du challenge :

```bash
$ cat _firmware.bin.extracted/squashfs-root/etc/config/sign 
w4ll_h1dd3n_p13c3_<-_->
```

