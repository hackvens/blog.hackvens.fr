---
layout: post
title: Videosurveillance
category: Sthack-2022
---

# Vidéosurveillance
Pour cette édition de la Sthack, nous avons eu le droit à un petit challenge de forensic assez varié. Le but de celui était de retrouver un flux vidéo à partir d'une écoute réseau.

## Analyse de l'écoute
Nous commençons d'abord par analyser l'écoute réseau, nous nous rendons compte rapidement de la présence de flux RTP. Ce protocole est utilisé pour le transport de flux média, comme les appels téléphoniques.

![échange RTP](/assets/img/sthack2022/videosurveillance/echange_RTP.png)

Wireshark nous permet facilement d'analyser cet échange RTP via le menu "Telephonie > RTP > Flux RTP" :

![flux RTP](/assets/img/sthack2022/videosurveillance/RTP.png)

Et bingo ! Nous obtenons bien un appel téléphonique, une personne semble laisser un message : "Salut, j'ai vu dans les logs du **Pfsense** que tu n'arrives pas à te connecter en SSH. Alors euh, j'ai vu dans les logs que le numéro de port il est bon, je te redonne le mot de passe, c'est **sofitel2011**. S O F I T E L, en minuscule 2011".

## A la recherche du Pfsense

Nous essayons donc de retrouver un serveur SSH sur lequel l'une des deux adresses IP aurait pu tenter de se connecter. Pour cela, nous essayons le filtre ssh avec wireshark, sans succès... Nous nous disons que si la connexion SSH n'a pas été interceptée, l'IP du Pfsense doit quand même se trouver dans la liste des IP contactées. Afin de gagner du temps, nous scannons uniquement les 1000 premiers ports des IP identifiées pendant l'analyse réseau et nous finissons par trouver un serveur avec une fausse interface web Pfsense.

![Reconstitution de l'interface web Pfsense](/assets/img/sthack2022/videosurveillance/pfsenseweb.png)

Nous réalisons ensuite un scan de l'ensemble des ports de l'IP du pfsense 212.47.234.75.Après quelques minutes d'attentes, nous trouvons un service SSH sur le port 32712. A la suite de plusieurs tentatives, nous arrivons à nous connecter au service avec l'utilisateur root et le mot de passe donné dans l'appel téléphonique.

## Découverte de sous-réseau

Avec un accès en ssh sur une machine, nous tentons de rebondir sur un autre réseau pour vérifier notre théorie. Pour cela, nous examinons d'abord les différentes intefaces de la machine :

![ip a](/assets/img/sthack2022/videosurveillance/ipa.png)

Nous remarquons tout de suite qu'une interface est connectée au réseau 172.16.0.10/24. Pour avoir des informations sur les différentes connexions actives que la machine pourrait avoir nous utilisons la commande netstat -leaputn.

![Rebond](/assets/img/sthack2022/videosurveillance/netstat.png)

Nous voyons plusieurs connexions ssh sur la machine 172.16.0.142, nous essayons de nous connecter en ssh avec les mêmes identifiants et ça fonctionne :)

![Nouveau serveur](/assets/img/sthack2022/videosurveillance/ssh2.png)

## Récupération de la vidéo

Nous remarquons que dans le répertoire /root il y a un fichier index.html. Nous essayons de récupérer le fichier via scp, mais on constate assez vite que ce n'est pas possible... Nous utilisons donc la old fashioned way : base64.

![base64](/assets/img/sthack2022/videosurveillance/base64.png)

Nous récupèrons le fichier, nous l'ouvrons avec VLC et le flag est à nous !

![FLAG](/assets/img/sthack2022/videosurveillance/mp4.png)

