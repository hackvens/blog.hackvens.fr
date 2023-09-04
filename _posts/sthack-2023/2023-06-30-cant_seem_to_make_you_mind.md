---
layout: post
title: Cant seem to make you mind
category: Sthack-2023
---

Voici un petit write up d'un challenge physique proposé de lors de la Sthack 2023  : Can't seem to make you mind 

Ce challenge nous mettait dans la peau d'un Hacktiviste cherchant à s'infiltrer dans les locaux de Evil Corp. Derrière une porte protégée par un Code PIN se trouve tous leurs sale petits secrets, et malheureusement pour eux : on a obtenu ce code et on va bientôt pouvoir les exposés à la face du monde. 

MAIS, car il y a toujours un mais (à l'image d'un épisode de Mr Robot, avoir le code ne suffit pas car le système est protégé par une double authentification : un OTP (One Time Password) est envoyé par SMS et il faut rentrer ce code dans les **3 minutes** et en moins de **10 tentatives**. Et bien évidemment : nous n'avons pas accès à ce téléphone.

Là vous vous dites : *"Mais comment contourner une MFA, cette sécurité spécifiquement conçu pour vérifier l'identité de la personne qui rentre un mot de passe ?*" 

Tout d'abord, pour resituer les choses, nous avions accès à un véritable boitier d'accès dont voici une reproduction à nu (qui ne rend pas hommage au soin apporté à la finition par le concepteur du chall mais que voulez vous, dans le feu de l'action à 5h du matin, on oublie de prendre des photos)
![c00414212c715b81be848b01f6a9d1a3.png](/assets/img/sthack2023/cantseemtomakeyoumind/c00414212c715b81be848b01f6a9d1a3.png)

Nous avions également accès à la doc explicative ainsi qu'au code de l'application et grâce à lui, on peut d'ores et déjà noter une première information depuis les directives include : 
Le boitier utilise un Raspberry Pico comme micro controleur :

![b74edf8351cf5356fea3368f5f965955.png](/assets/img/sthack2023/cantseemtomakeyoumind/b74edf8351cf5356fea3368f5f965955.png)

Tout d'abord, nous avons regardé du côté de l'envoi de SMS si y a pas possibilité de modifier le code pour recevoir le SMS, mais les valeurs sont écrite en dur dans le programme.

![25b261150b0e312f4290d7a7d7185753.png](/assets/img/sthack2023/cantseemtomakeyoumind/25b261150b0e312f4290d7a7d7185753.png)

Et puis en regardant comment l'OTP est généré, nous avons tout de suite remarqué une chose : 
La seed (la valeur initial qui est utilisée pour créer de l'aléatoire et donc éviter que le mot de passe soit prédictible) n'utilise pas la façon habituelle de faire des micro controleur : 

En effet, d'habitude pour créer une seed, les micro contoleur vont lire la valeur du voltage sur une de leur borne qui n'est relié à aucun composant : la valeur va ainsi dépendre du bruit ambient (onde radio, interférence des téléphones portables, fond diffus cosmologique etc...). Ce qui rend la prédiction quasiment impossible. 

Or ici, la seed est récupérée à partir de la valeur de la température relevé par un capteur interne : 

![fdbaf316187c94cc69cc39a57d53e806.png](/assets/img/sthack2023/cantseemtomakeyoumind/fdbaf316187c94cc69cc39a57d53e806.png)

Et une température :  ça peut se controler et si on connait la valeur que va renvoyer le capteur, on peut ainsi prévoir la valeur du code OTP.

A notre disposition, il y a en effet une bombe à froid dont la fiche technique spécifie la température exacte (En l'occurence la notre était à -35 C°) : 

![090fdf58368c7c63863d649655bb7a9a.png](/assets/img/sthack2023/cantseemtomakeyoumind/090fdf58368c7c63863d649655bb7a9a.png)

Désormais nous n'avions plus qu'à lire la datasheet technique du Raspberry Pico afin de trouver des informations sur comment le catpeur lit la température et surtout sous quel format la fonction *read_onboard_temperature()* renvoit la valeur.

![6291c0153bc28b1f135e90a9707d0e28.png](/assets/img/sthack2023/cantseemtomakeyoumind/6291c0153bc28b1f135e90a9707d0e28.png)

Alors nous avons recherché dans ce document de 639 pages https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf 
, l'information qu'il nous fallait et à la page 566 nous avons trouvé ceci :

![75b339ff373fc15088f967dd870fc823.png](/assets/img/sthack2023/cantseemtomakeyoumind/75b339ff373fc15088f967dd870fc823.png)

Super, plus qu'à faire une simple équation pour retrouver la valeur *raw* pour T= -35, super facile non ? ... non ?

Et c'est ici que malheureusement, nous avons échoué car oui cher lecteur ou lectrice : 
le flag on l'a pas eu. 

Je pourrais vous lister tout un tas d'excuse à base de manque de sommeil et de temps, de nos outils qui ont mal géré les nombres à virgules etc..  Bref ça n'a pas marché. 

Mais en reprenant à froid (et en discutant avec le créateur du challenge), voici le résultat qui aurait pu nous apporter le flag : 

Nous savons que la température est calculé comme ceci :

```
uint16_t raw = adc_read();
const float conversion_factor = 3.3f / (1<<12); 
float result = raw * conversion_factor;
float temp = 27 - (result -0.706)/0.001721;
```

Si on effectue l'opération inverse, nous pouvons définir l'expression suivante :

```
raw = ((27 - (-35)) * 0.001721 + 0.706) / conversion_factor
raw = 0.812702 / (3.3 / 4096)
raw = 1008,7355733333 = 1008 (vu qu'à l'origine c'est un type int)
```

On reprend le code de l'application qu'on modifie pour obtenir la valeur de l'OTP :

```
#define OTP_MIN 100000
#define OTP_MAX 999999

int main(void){
	int read_value_for_negative_35 = 1008;
	unsigned int seed = read_value_for_negative_35 + 324092;
	srand(seed);		 
	int otp = OTP_MIN + rand() % (OTP_MAX + 1 - OTP_MIN); 
	printf("l'OTP est : %d\n", otp);
	return(1);
	}
```

![8d1337025d434fbf4c0033890fc18baf.png](/assets/img/sthack2023/cantseemtomakeyoumind/8d1337025d434fbf4c0033890fc18baf.png)

Il nous aurait ensuite fallu refroidire le boitier jusqu'à une température de -35 C° puis entrée le code PIN initial puis l'OTP pour ouvrir la porte et accéder aux sombres secrets de Evil Corp.  


