---
layout: post
title: Catch the bird
category: Sthack-2022
---

# STHACK 2022

## Épreuve "Catch the bird, a trip from web to IRL"

Le 20/05/2022



Cette épreuve indiquée dans la catégorie web et physique a intrigué notre équipe qui s'est penchée dessus durant plus de 5h.



En prémisse de cette épreuve, nous avons pour énoncé un scénario entre deux personnages ainsi qu'une carte postale avec un but précis, mettre la main sur une statuette d'oiseau:

![img](/assets/img/sthack2022/catch-the-bird/1.png){:class="img-responsive smallpict"}

### Ecoute radio

Les termes "Marconi" et "Guglielmo" étant soulignés, nous découvrons qu'ils font référence à l'un des précurseurs de la radio. Le numéro de rue "101.10" semble nous inviter à écouter sur cette fréquence FM. Nous devons donc nous munir d'un téléphone ainsi que d'écouteurs filaires pour écouter un message diffusé en boucle.

L'orateur utilise l'alphabet phonétique pour transmettre son message.


![image-20220602100444065](/assets/img/sthack2022/catch-the-bird/2.jpg){:class="img-responsive smallpict"}

Nous obtenons alors le lien suivant:

```
STHACKMDBIDWKYEBHKLNFNWMQHJ3OXNKE5INYLZRTWZENR752TN77OAD.ONION
```

### Accès au compte adminsitrateur

Via le navigateur Tor, nous arrivons sur ce qui semble être un marché du darknet, nous y retrouvons la statuette d'oiseau dans la liste des articles vendus:

![img](/assets/img/sthack2022/catch-the-bird/3.png){:class="img-responsive smallpict"}


Une page nous indique que nous devons trouver le moyen d'effectuer un achat afin d'obtenir des coordonnées GPS. Cependant, afin d'effectuer une commande nous devons être authentifiés, l'administrateur est indiqué comme étant très pointilleux.

Nous découvrons une interface d'authentification au compte d'administration. Nous cherchons donc à usurper l'identité de cet administrateur.

![img](/assets/img/sthack2022/catch-the-bird/4.png){:class="img-responsive smallpict"}

La page de login suivante divulgue un message d'erreur en base64 lorsque la tentative d'authentification n'est pas correcte.

Nous obtenons l'erreur suivante une fois le message déchiffré :

```php
An exception occured on line 31 of /www/src/Controller/LoginController.php --- Invalid password. ---

                return $this->render('login/index.html.twig', [
                    'controller_name' => 'LoginController',
                ]);
            }
            if ($request->request->get('password') === self::PASSWORD)
            {
                $session = new Session();
                $session->set('is_admin', true);
                return $this->redirectToRoute('admin');
            } else { sleep(rand(3, 5)); throw new Exception('Invalid password.'); }
        }
        #[Route('/logout', name: 'app_admin_logout')]
        public function logout (Request $request): Response
        {

            $session = new Session();
            $session->clear();
            return $this->redirectToRoute('app_admin_login');
```

Ici nous avons déduit qu'il fallait récupérer le contenu de la variable 'self::PASSWORD'.

Nous obtenons un autre morceau de code lié à une erreur au niveau de la page listant les articles:

```
An exception occured on line 86 of /www/vendor/symfony/expression-language/Lexer.php --- Unexpected character "'" around position 0 for expression ' OR 1=1# && true. ---

                } elseif (str_contains('.,?:', $expression[$cursor])) {
                    // punctuation
                    $tokens[] = new Token(Token::PUNCTUATION_TYPE, $expression[$cursor], $cursor + 1);
                    ++$cursor;
                } elseif (pregmatch('/[a-zA-Z\x7f-\xff][a-zA-Z0-9_\x7f-\xff]*/A', $expression, $match, 0, $cursor)) {
                    // names
                    $tokens[] = new Token(Token::NAME_TYPE, $match[0], $cursor + 1);
                    $cursor += \strlen($match[0]);
                } else {
                    // unlexable
                    throw new SyntaxError(sprintf('Unexpected character "%s".', $expression[$cursor]), $cursor, $expression);
                }
            }
            $tokens[] = new Token(Token::EOF_TYPE, null, $cursor + 1);
            if (!empty($brackets)) {
                [$expect, $cur] = array_pop($brackets);
                throw new SyntaxError(sprintf('Unclosed "%s".', $expect), $cur, $expression);
            }
```

Nous remarquons la mention de "expression-language" dans le message de l'erreur ce qui nous pousse à consulter la documentation suivante :

https://symfony.com/doc/current/components/expression_language/syntax.html#component-expression-functions

La partie "Working with Functions" attire notre œil, car il semble possible de récupérer le contenu d'une variable via la fonction "constant()". De plus, il est possible d'influencer le message d'erreur en jouant sur un paramètre POST. Une injection semble donc possible.

Après quelques tentatives et à l'aide des messages d'erreurs, nous identifions le contexte lié à notre variable "self::PASSWORD".

![img](/assets/img/sthack2022/catch-the-bird/5.png){:class="img-responsive smallpict"}

Nous avons donc "**constant('App\Controller\LoginController::PASSWORD')**".

Néanmoins, après avoir récupéré le contenu du mot de passe administrateur nous devons trouver un moyen de l'afficher ou de le récupérer en clair.

Nous utilisons alors une autre fonction basique de symfony : **matches** afin de vérifier les caractères un à un.

```
constant('App\\Controller\\LoginController::PASSWORD') matches '/a*/'
```

Ce qui nous permet d'identifier manuellement le mot de passe comme étant : **Ybhr5vmjJD**

### Récupérer et déchiffrement d'un fichier

Nous  accédons ensuite à la liste des ventes effectuées, et obtenons les éléments tant attendus, à savoir les coordonnées GPS du point où se trouvent un cadenas et une clé secrète.

![img](/assets/img/sthack2022/catch-the-bird/6.png){:class="img-responsive smallpict"}

Le point GPS nous emmène dans les jardins de la mairie de Bordeaux, où nous découvrons un cadenas verrouillé auquel est attaché une clé USB.

![img](/assets/img/sthack2022/catch-the-bird/7.png){:class="img-responsive smallpict"}

L'usage de compétence de lock-picking est utile pour récupérer la clé USB, cependant dans notre cas nous manquions de temps et avons déplacé directement notre pc afin de brancher la clé USB sans avoir besoin d'ouvrir le cadenas.

Une fois le fichier récupéré, une simple commande permet de le déchiffrer à l'aide de la clé précédemment retrouvée lors de l'achat ci-dessus :

```
openssl enc -d -aes-256-cbc -in maltesefalcon.txt.aes256cbc -out file.dec
```

Nous obtenons alors un autre lien ONION:

```
The auction will take place at the address : maltapyzyfnwvgkl4se7tlhlrohule77cgnaguy2nouxyvosovjldbyd.onion

All bid will be made in $ only, you will be asked to provide an bank account address on auction start. You will only be able to bid up to this account available balance, so be sure to gather sufficients funds on this account before auction start.

Auction admittance will start at 2022-05-23 20:00:00 UTC and entree will remain open up to 2022-05-23 20:15:00 UTC. Past that, nobody will be accepted and auctions will start.

---------
Falcon
```

### Récupération du flag

La vente aux enchères semble commencer deux jours après la fin du CTF. En effet, un compte à rebours est présent sur la page d'accueil du nouveau site.

Nous changeons donc la date de notre pc afin de réduire ce délai à 0. Ceci n'était pas la méthode désirée par le créateur du challenge, mais une méthode qui fonctionne tout de même. C'est ainsi que nous avons obtenu le flag et validé l'épreuve :

![img](/assets/img/sthack2022/catch-the-bird/8.png){:class="img-responsive smallpict"}

![img](/assets/img/sthack2022/catch-the-bird/9.png){:class="img-responsive smallpict"}
