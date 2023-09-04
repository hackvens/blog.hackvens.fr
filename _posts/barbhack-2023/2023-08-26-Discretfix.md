---
layout: post
title: Discretfi
category: Barbhack-2023
---

# Discretexfi

**Catégorie :** Forensic

![enonce](/assets/img/barbhack2023/discretfi/enonce-discretexfi.png)

Dans ce challenge nous récupérons un fichier pcap comportant des paquets réseaux capturé à un instant donné. Nous observons dans un premier temps du traffic TCP en grande quantité :

![wireshark-full](/assets/img/barbhack2023/discretfi/wireshark-full.png)

En filtrant sur les requêtes de type `GET` nous restreignons l'affichage aux éléments qui nous intéresses. Pour ce faire nous utilisons le filtre `http.request.method=="GET"`:

![wireshark-filter](/assets/img/barbhack2023/discretfi/wireshark-filter.png)

Nous notons la présence de 3 URL différents :

- /newfile?f=secret.txt
- /bit
- /lastbits

Cependant nous ne retrouvons pas d'élément faisant référence au flag que nous cherchons. La seul différence notable est la version d'HTTP utilisé qui change à chaque requête. Nous isolons puis convertissons cette information avec la commande suivante :

```bash
tshark -r Discretexfi.pcap -Y 'http.request.method == "GET"' | awk -F ' ' '{print $10}' | cut -d '.' -f 2 | sed ':a;N;$!ba;s/\n//g; s/ //g' | perl -lpe '$_=pack"B*",$_'
```

Si nous décomposons cette commande nous obtenons :

- `tshark -r Discretexfi.pcap -Y 'http.request.method == "GET"'` permet d'extraire les informations du fichier de capture avec le filtre utilisé sur wireshark.

> 122702  12.714845  192.168.0.2 → 192.168.0.10 HTTP 217 GET /bit HTTP/1.1 
> 122712  12.715613  192.168.0.2 → 192.168.0.10 HTTP 217 GET /bit HTTP/1.1 
> 122722  12.716368  192.168.0.2 → 192.168.0.10 HTTP 217 GET /bit HTTP/1.1 
> 122732  12.717119  192.168.0.2 → 192.168.0.10 HTTP 217 GET /bit HTTP/1.0 
> 122742  12.717882  192.168.0.2 → 192.168.0.10 HTTP 222 GET /lastbits HTTP/1.0

- `awk -F ' ' '{print $10}'` permet de n'extraire que le dernier champs, à savoir **HTTP/1.1** ou **HTTP/1.0**.

> HTTP/1.1
> HTTP/1.1
> HTTP/1.1
> HTTP/1.0
> HTTP/1.0

- `cut -d '.' -f 2` permet de ne garder que le dernier chiffre de cette version, soit **1** ou **0**.

  > 1
  > 1
  > 1
  > 0
  > 0

- `sed ':a;N;$!ba;s/\n//g; s/ //g'` permet de retirer les retours à la ligne

> 11100

- `perl -lpe '$_=pack"B*",$_'` permet de convertir la suite binaire en ascii

Nous obtenons alors le résultat suivant :

```ascii
�0�9�6��4�2�9�22��2c�lices et des plaisirs gustatifs, se cache en moi une passion brûlante pour les bières. Derrière mon sourire et mes conversations animées, se trouve un amour profond et sincère pour cette boisson brassicole.
La bière va bien au-delà d'une simple boisson pour moi. C'est une véritable expérience sensorielle, un voyage palpitant à travers les saveurs, les arômes et les textures. Chaque gorgée est une découverte, une exploration des possibilités infinies qu'offre le monde brassicole.
J'adore me plonger dans la diversité des bières, partir à la recherche de nouvelles variétés, de brasseries artisanales et de créations audacieuses. Chaque bière est unique, racontant une histoire qui m'envoûte et me transporte vers de nouveaux horizons gustatifs.
brb{hTTp_vErSi0n_iS_th3_W44444y}
La dégustation de bière est un art que je chéris. Je savoure chaque gorgée, laissant la bière se répandre sur ma langue et révéler ses secrets. Les saveurs se déploient avec grâce, les notes fruitées, épicées ou maltées dansent dans ma bouche, m'invitant à explorer davantage.
Mais ma passion pour les bières ne s'arrête pas à la dégustation. Je suis captivé par l'histoire et la culture brassicole, par les traditions et les techniques de brassage. Je m'imprègne de chaque détail, chaque anecdote qui entoure cette boisson millénaire, nourrissant ma soif de connaissances brassif����\
```

Dans l'ensemble de ce texte nous retrouvons le flag : **`brb{hTTp_vErSi0n_iS_th3_W44444y}`**.

