# Décisions prises, justifications et indicateurs chiffrés

## Demi-journée 1 — Étape 2 : comprendre le jeu Olist

Tout le travail d'exploration est dans le notebook
[notebooks/01_comprendre_olist.ipynb](notebooks/01_comprendre_olist.ipynb). Ici, on raconte ce qu'on a
regardé, ce qu'on a trouvé et ce qu'on en retient pour la suite.

### Premier contact avec les fichiers

On a commencé simplement : ouvrir les 9 CSV avec `pd.read_csv` et regarder leur taille, leurs colonnes
et quelques lignes. Petite précaution dès le chargement : on lit les codes postaux en texte et pas en
nombre. Ce sont des identifiants, et en entier on perdrait le zéro du début (`01037` deviendrait `1037`).

Ce qui ressort tout de suite, c'est que `orders` est le centre de tout. Les autres tables tournent
autour :

| Fichier | Lignes | Ce qu'on y trouve |
|---|---:|---|
| orders | 99 441 | une ligne par commande, avec son statut et 5 dates |
| customers | 99 441 | le client de chaque commande |
| order_items | 112 650 | les articles de chaque commande (prix, frais de port) |
| order_payments | 103 886 | le ou les paiements d'une commande |
| order_reviews | 99 224 | l'avis laissé par le client (note de 1 à 5, commentaire) |
| products | 32 951 | le catalogue : catégorie, poids, dimensions |
| sellers | 3 095 | les vendeurs et leur localisation |
| geolocation | 1 000 163 | des points GPS par code postal |
| product_category_name_translation | 71 | la traduction des catégories en anglais |

geolocation nous a surpris par sa taille : un million de lignes, alors qu'il n'y a qu'environ 19 000
codes postaux différents. On y revient plus bas.

### Comment les tables sont liées

Pour chaque table, on a d'abord cherché sa clé, c'est-à-dire la colonne qui identifie une ligne de façon
unique. Ensuite on a vérifié que les liens entre tables tiennent vraiment, en mesurant la part des
identifiants qu'on retrouve de l'autre côté.

```
                    product_category_name_translation
                                 │ product_category_name
                                 ▼
 geolocation ─zip─ customers   products ◄─product_id─┐
     │               │ customer_id                   │
     │               ▼ (1 pour 1)                    │
     │            orders ──order_id (1 pour n)──► order_items ──seller_id──► sellers ─zip─ geolocation
     │               │
     │               ├──order_id (1 pour n)──► order_payments
     │               └──order_id (1 pour n)──► order_reviews
```

Bonne nouvelle : l'intégrité est presque parfaite. Tous les articles, paiements et avis pointent vers une
commande qui existe, et tous les articles vers un produit et un vendeur connus. Les seuls trous :

- 775 commandes n'ont aucun article. En regardant leur statut, on comprend pourquoi : ce sont presque
  toutes des commandes `unavailable` (603) ou `canceled` (164).
- 1 commande n'a pas de paiement.
- 768 commandes n'ont pas d'avis, ce qui est normal puisque l'avis est facultatif.
- 2 catégories n'ont pas de traduction : `pc_gamer` et `portateis_cozinha_e_preparadores_de_alimentos`.
- Environ 0,3 % des codes postaux clients et vendeurs n'existent pas dans geolocation.

En creusant, on est tombés sur quatre pièges qui vont conditionner tout le reste.

**1. `customer_id` ne désigne pas une personne.** On pensait au départ que c'était l'identifiant du
client, mais il y en a exactement un par commande. La vraie personne, c'est `customer_unique_id` :
96 096 personnes pour 99 441 commandes. 2 997 personnes ont commandé plusieurs fois, dont une 17 fois.
Si on avait fait le RFM sur `customer_id`, tout le monde aurait eu une fréquence de 1. On travaillera
donc sur `customer_unique_id`.

**2. Un même avis peut être rattaché à plusieurs commandes.** `review_id` apparaît 814 fois en double,
toujours avec la même note. La vraie clé est donc le couple (`review_id`, `order_id`).

**3. geolocation n'a pas de clé.** Il y a en moyenne 53 points par code postal, et 261 831 lignes sont
des copies exactes d'autres lignes. Si on la joignait telle quelle aux clients, chaque client serait
démultiplié.

**4. Les jointures directes dupliquent les lignes.** 9 803 commandes ont plusieurs articles, 2 961 ont
plusieurs paiements, 547 ont plusieurs avis. Si on joint orders, items et payments d'un coup, une commande
à 3 articles et 2 paiements devient 6 lignes, et le chiffre d'affaires est faux. La solution : résumer
chaque table avant de la joindre (voir plus bas).

### Ce qui compte dans les colonnes

On a ensuite passé les colonnes une par une, en se demandant à chaque fois quelle information est
vraiment utile et ce qui cloche.

**Les commandes (orders)**

Le statut est essentiel : 97 % des commandes sont `delivered`, le reste est annulé, indisponible ou
encore en cours. Une commande annulée n'est pas une vente, donc il faudra choisir ce qu'on compte comme
vente réelle.

Les dates ne sont pas très parlantes en elles-mêmes, mais on peut en tirer ce qui intéresse vraiment la
direction : le délai de livraison (environ 10 jours en médiane) et le retard (8 % des commandes livrées
arrivent après la date promise). On s'attend à ce que le retard joue sur la satisfaction.

Les données couvrent de septembre 2016 à octobre 2018. Attention cependant : 2016 ne compte que
329 commandes, et septembre–octobre 2018 seulement 20. Le début et la fin de l'extraction sont presque
vides, ce qui faussera les courbes mensuelles si on n'y fait pas attention.

Côté incohérences, on a trouvé 166 commandes confiées au transporteur avant même d'avoir été achetées,
23 livrées avant d'être remises au transporteur, 8 marquées « livrées » sans date de livraison, et
6 non livrées qui ont pourtant une date de livraison. Les dates manquantes (environ 3 000 au total)
concernent surtout des commandes pas encore livrées, donc elles sont logiques.

**Les articles (order_items)**

C'est là que se trouve l'argent : `price` (de 0,85 à 6 735 R$, 75 R$ en médiane) et `freight_value`
(les frais de port). On a d'abord cru que `order_item_id` était une quantité, mais c'est en fait un
numéro de ligne. Acheter trois fois le même produit donne trois lignes (on a compté 10 225 répétitions).
La quantité se retrouve donc en comptant les lignes.

**Les paiements (order_payments)**

Trois paiements sur quatre se font par carte de crédit, environ un sur cinq par boleto, le reste en bon
d'achat ou carte de débit. 3 paiements ont un type `not_defined`. Les échéances vont de 1 à 24, sauf
2 paiements à 0 échéance, ce qui n'a pas de sens. 9 paiements valent 0 R$, ce sont souvent des bons
d'achat.

On a aussi vérifié que ce que le client paie correspond au prix des articles plus le port. C'est le cas
presque partout, sauf pour 249 commandes où l'écart dépasse 1 R$, probablement à cause des intérêts du
paiement en plusieurs fois ou des bons d'achat. Il faudra donc choisir une seule source pour les montants.

**Les avis (order_reviews)**

La note (moyenne 4,1 sur 5) est l'information qui nous intéresse : c'est la satisfaction. Les
commentaires sont souvent vides (88 % sans titre, 59 % sans message), ce qui est normal puisqu'ils sont
facultatifs. Pour le clustering, le texte ne nous servira pas, mais savoir si le client a pris le temps
de commenter peut être intéressant. Détail étrange : 74 avis sont datés d'avant l'achat.

**Les produits (products)**

610 produits n'ont pas de catégorie. Pour ces mêmes produits, la longueur du nom, celle de la
description et le nombre de photos manquent aussi : ce sont visiblement des fiches incomplètes.
2 produits n'ont pas de dimensions et 4 pèsent 0 g, ce qui est impossible. Les colonnes
`product_name_lenght` et `product_description_lenght` (avec la faute d'origine) décrivent la fiche
produit plutôt que le produit lui-même ; on doute qu'elles servent.

**Les clients et vendeurs**

Les clients viennent des 27 États du Brésil, les vendeurs de 23 États seulement. Les noms de ville des
vendeurs sont un vrai fouillis : `sao paulo / sao paulo`, `sao paulo - sp`, `sbc/sp`, `lages - sc`, et
même un code postal (`04482255`) saisi à la place de la ville. Il faudra les normaliser : minuscules,
sans accents, sans le « / État » à la fin.

**La géolocalisation (geolocation)**

En plus des doublons, 42 points tombent hors du Brésil (jusqu'en Europe ou en Asie d'après les
coordonnées), et une même ville peut être écrite de deux façons (`sao paulo` / `são paulo`). Comme on
dispose déjà de la table `client_geo` qui donne une position par client, geolocation ne nous servira
plus que pour placer les vendeurs.

### Garder l'essentiel : agréger avant de joindre

Pour éviter le piège des lignes dupliquées, on résume chaque table « fille » avec
`groupby(...).agg(...)` pour n'avoir plus qu'une ligne par commande (ou par code postal). Ensuite on
joint avec `validate="one_to_one"`, ce qui fait planter pandas si une jointure duplique des lignes. C'est
un bon garde-fou.

Concrètement :

- **Articles → par commande** : nombre d'articles, nombre de produits différents, nombre de vendeurs,
  total des prix, total du port.
- **Paiements → par commande** : montant payé, nombre de paiements, nombre maximal d'échéances, et le
  moyen de paiement principal (celui qui porte le plus gros montant).
- **Avis → par commande** : note moyenne, nombre d'avis, et si le client a laissé un commentaire.
- **geolocation → par code postal** : on garde la médiane des points situés au Brésil, moins sensible
  aux points aberrants qu'une moyenne. On passe d'un million de lignes à environ 19 000.
- **Commandes → par personne** (`customer_unique_id`) : récence, fréquence, montant total, panier moyen,
  note moyenne. C'est le point de départ du RFM de l'étape 6.

Après toutes ces jointures, on a bien toujours 99 441 commandes, ce qui confirme qu'aucune ligne n'a été
dupliquée.

On a fait un premier essai à l'échelle de la personne, sur les commandes livrées (93 358 personnes) :
la dernière commande remonte en médiane à 218 jours, le montant total médian est de 108 R$ et la note
moyenne de 4,2. Mais la fréquence moyenne est de 1,03 : presque tout le monde n'a commandé qu'une seule
fois. Elle ne permettra donc pas de bien séparer les clients, et il faudra ajouter d'autres indicateurs
(satisfaction, retards subis, catégories achetées, paiement en plusieurs fois…).

### Ce qu'on retient pour la suite

- On raisonne sur `customer_unique_id` dès qu'on parle de clients.
- On agrège toujours avant de joindre, et on vérifie que le chiffre d'affaires reste le même avant et
  après.
- Il faut décider ce qu'est une vente. Notre piste : les commandes `delivered`, peut-être aussi
  `shipped` et `invoiced`.
- Il faut choisir une source pour les montants. Notre piste : `payment_value` pour le RFM, parce que
  c'est ce que le client a vraiment payé, et `price + freight_value` pour le chiffre d'affaires par
  produit ou par vendeur.
- La liste des problèmes à traiter dans l'audit et le nettoyage est déjà longue : dates incohérentes,
  échéances à 0, produits à 0 g ou sans catégorie, catégories non traduites, villes mal écrites, doublons
  et points hors Brésil dans geolocation, avis en double, une commande sans paiement et des commandes sans
  article.

## Demi-journée 1 — Étape 3 : l'audit de qualité

Tout le détail est dans [notebooks/02_audit_qualite.ipynb](notebooks/02_audit_qualite.ipynb). Le principe : **on mesure,
on ne corrige rien**. Chaque contrôle passe par une petite fonction `constat(...)` qui l'enregistre dans un
registre ; à la fin, un seul tableau récapitule les 121 contrôles lancés, dont 71 trouvent au moins une ligne
concernée. On garde aussi les contrôles à 0 : savoir qu'il n'y a ni doublon de `order_id` ni article orphelin, c'est
aussi une information.

Pour les valeurs aberrantes, on utilise la règle de Tukey avec un seuil large (Q1 − 3·IQR, Q3 + 3·IQR) : on cherche
des valeurs *extrêmes*, pas simplement grandes. Mais une valeur extrême n'est pas forcément fausse (un frigo pèse
plus de 6,7 kg), d'où des contrôles « métier » à côté : poids nul, densité impossible, échéances à 0…

### Vue d'ensemble

Nombre de contrôles ayant trouvé un problème, par fichier et par famille :

| Fichier | Manquants | Doublons | Temporel | Entre fichiers | Aberrants |
|---|---:|---:|---:|---:|---:|
| orders | 6 | 0 | 6 | 4 | 2 |
| order_items | 0 | 0 | 1 | 1 | 4 |
| payments | 0 | 1 | — | 2 | 4 |
| reviews | 2 | 3 | 2 | 1 | 1 |
| products | 8 | 0 | — | 1 | 6 |
| customers | 0 | 2 | 1 | 1 | 1 |
| sellers | 0 | 0 | — | 2 | 2 |
| geolocation | 0 | 2 | — | 2 | 3 |
| translation | 0 | 0 | — | 0 | 0 |

« — » : la table n'a pas de date, la famille ne s'applique pas. La table `client_geo` (PostgreSQL) a aussi été
contrôlée : 99 441 lignes, aucune valeur manquante, aucun doublon, chaque `customer_id` retrouvé dans les deux
sens et toutes les positions au Brésil. Rien à signaler.

### orders

- **Manquants** : 160 dates de validation, 1 783 dates de remise au transporteur et 2 965 dates de livraison
  manquent. En croisant avec le statut, presque tout est logique (une commande annulée ou en cours n'a pas de date de
  livraison). Restent anormales les commandes `delivered` sans date de validation (14), sans remise transporteur (2)
  ou sans date de livraison (8).
- **Doublons** : aucun. `order_id` et `customer_id` sont uniques.
- **Temporel** : c'est la table la plus touchée.
  - 1 359 commandes sont remises au transporteur *avant* d'être validées (médiane : 17 h d'avance, jusqu'à 171 jours) ;
  - 166 sont remises au transporteur avant même l'achat ;
  - 23 sont livrées avant la remise au transporteur ;
  - 6 ont une date de livraison sans être `delivered` ;
  - 1 723 sont toujours « en cours » (`shipped`, `invoiced`, `processing`…) plus de 60 jours avant la fin de
    l'extraction : elles ne seront jamais livrées, ce sont en pratique des commandes perdues ou abandonnées ;
  - 349 commandes tombent dans des mois quasi vides (2016-09, 2016-10, 2016-12, 2018-09, 2018-10).

  Aucune commande n'est validée ou livrée avant l'achat, ni avec une date estimée antérieure à l'achat.
- **Entre fichiers** : 775 commandes sans article, 1 sans paiement, 768 sans avis. Parmi les commandes sans article,
  1 est pourtant `shipped` : c'est la seule vraiment incohérente, les autres sont annulées ou indisponibles.
- **Aberrants** : 306 livraisons prennent plus de 60 jours (jusqu'à 210) et 81 validations arrivent plus d'une semaine
  après l'achat (jusqu'à 188 jours).

### order_items

- **Manquants / doublons** : aucun, et la numérotation `order_item_id` va bien de 1 à n sans trou.
- **Temporel** : 12 articles ont une date limite d'expédition plus de 60 jours après l'achat (jusqu'à presque 3 ans).
- **Entre fichiers** : toutes les clés se retrouvent. En revanche, 549 articles appartiennent à des commandes
  `canceled` ou `unavailable` : ils ne doivent pas compter dans le chiffre d'affaires.
- **Aberrants** : aucun prix nul ou négatif. 4 074 prix dépassent 420 R$ (max 6 735 R$) et 5 538 frais de port dépassent
  45 R$ (max 410 R$) : ce sont des produits chers ou lourds, pas des erreurs. Plus parlant : 383 articles ont des
  frais de port à 0 (livraison offerte ?) et 4 124 coûtent moins cher que leur port.

### payments

- **Manquants / doublons** : aucun sur la clé (`order_id`, `payment_sequential`). Mais 80 commandes ont une
  numérotation avec des trous (souvent un seul paiement numéroté 2, sans paiement 1).
- **Entre fichiers** : 249 commandes ont un total payé qui diffère de plus de 1 R$ du prix + port (de −52 à
  +183 R$) ; 830 paiements concernent des commandes sans article.
- **Aberrants** : 3 types `not_defined`, 2 paiements à 0 échéance, 9 paiements à 0 R$. 4 046 paiements dépassent
  517 R$ (max 13 664 R$), cohérent avec les gros paniers. Aucun paiement en plusieurs fois hors carte de crédit :
  les échéances sont donc fiables.

### reviews

- **Manquants** : 88 % sans titre, 59 % sans message, ce qui est normal (champs facultatifs).
- **Doublons** : aucune ligne en double, et le couple (`review_id`, `order_id`) est bien unique. Mais 814 `review_id`
  sont partagés par plusieurs commandes, et 551 commandes ont plusieurs avis, dont **202 avec des notes
  différentes** : il faudra choisir laquelle garder (la plus récente, ou la moyenne).
- **Temporel** : 64 avis sont créés avant le jour de l'achat (l'étape 2 en comptait 74 en comparant à l'heure près,
  alors que la date de création est toujours à minuit) ; 5 127 avis sont créés avant la livraison effective, ce qui
  s'explique sans doute par l'envoi du questionnaire à la date *estimée*. Aucune réponse n'est antérieure au
  questionnaire.
- **Entre fichiers** : 1 206 avis portent sur des commandes annulées ou indisponibles (note moyenne 1,7 contre 4,1 ailleurs :
  l'avis porte alors sur l'annulation et non sur le produit).
- **Aberrants** : toutes les notes sont entre 1 et 5 ; 682 réponses arrivent plus de 30 jours après le
  questionnaire (jusqu'à 519 jours).

### products

- **Manquants** : les 610 produits sans catégorie sont aussi sans longueur de nom, de description ni nombre de
  photos : ce sont des fiches incomplètes. Les dimensions manquent pour 2 produits (dont 1 qui n'a aucune
  information du tout).
- **Doublons** : aucun.
- **Entre fichiers** : 13 produits ont une catégorie sans traduction (`pc_gamer`,
  `portateis_cozinha_e_preparadores_de_alimentos`). Tous les produits ont été vendus au moins une fois.
- **Aberrants** : 4 produits pèsent 0 g, et 18 ont une densité supérieure à 10 g/cm³ (plus lourds que du fer à volume
  égal, jusqu'à 85 g/cm³) : poids ou dimensions faux. Les 3 073 poids au-dessus de 6,7 kg et les dimensions qui
  dépassent un mètre sont plausibles (meubles, électroménager).

### customers

- **Manquants / doublons** : aucun. `customer_unique_id` se répète, c'est voulu (clients fidèles), mais 250 personnes
  ont plusieurs codes postaux et 39 apparaissent dans plusieurs États (déménagement ou livraison à une autre adresse).
- **Temporel** : 292 commandes sont passées par la même personne à la même seconde qu'une autre : vraisemblablement un
  panier découpé en plusieurs commandes. Pour la fréquence du RFM, il faudra décider si cela compte pour un ou
  plusieurs achats.
- **Entre fichiers** : chaque `customer_id` a sa commande et sa position dans `client_geo` ; 278 codes postaux sont
  absents de geolocation (sans conséquence puisqu'on utilise `client_geo`). L'État déclaré correspond toujours à
  celui du code postal.
- **Aberrants** : codes postaux tous à 5 chiffres, États tous valides, une seule ville mal saisie.

### sellers

- **Manquants / doublons** : aucun.
- **Entre fichiers** : tous les vendeurs ont vendu. 7 codes postaux sont absents de geolocation (vendeurs qu'on ne
  pourra pas placer sur une carte) et 35 vendeurs déclarent un État différent de celui de leur code postal.
- **Aberrants** : 26 villes mal saisies (`sao paulo - sp`, `sbc/sp`, `04482255`,
  `novo hamburgo, rio grande do sul, brasil`…). Une fois normalisées, 17 villes s'avèrent écrites de plusieurs façons
  (`sao paulo` sous 4 formes).

### geolocation

- **Manquants** : aucun.
- **Doublons** : 261 831 lignes entièrement dupliquées (26 %), et 94,5 % des préfixes ont plusieurs points (jusqu'à
  1 146). La table n'a pas de clé.
- **Entre fichiers** : 8 préfixes sont rattachés à plusieurs États ; 4 099 préfixes (22 %) ne servent à aucun client
  ni vendeur.
- **Aberrants** : 42 points hors du Brésil, 1 186 points à plus de 50 km de la médiane de leur préfixe (jusqu'à
  9 534 km), et 45 % des préfixes ont plusieurs orthographes de ville (`embu-guacu` / `embu-guaçu` / `embu guaçu`).

### translation

Rien à signaler : pas de manquant, pas de doublon, toutes les catégories traduites sont utilisées. Le seul problème
est du côté de products (2 catégories absentes du dictionnaire).

### Ce qu'on retient pour le nettoyage

Les problèmes ne se valent pas. On les classe en trois groupes pour l'étape 4 :

1. **Erreurs à corriger ou rejeter** : dates dans le désordre (166 + 1 359 + 23), statuts contradictoires avec les
   dates (8 + 6 + 1), échéances à 0, type `not_defined`, poids à 0 g ou densité impossible, points hors Brésil,
   villes mal saisies.
2. **Faits métier à filtrer selon l'usage** : commandes annulées / indisponibles et leurs articles, paiements et avis ;
   commandes bloquées « en cours » ; mois quasi vides au début et à la fin.
3. **Particularités normales à gérer sans rejeter** : commentaires vides, avis multiples par commande, écarts payé /
   facturé, frais de port nuls, produits lourds ou chers, doublons de geolocation (résolus par l'agrégation par
   préfixe de l'étape 2).
