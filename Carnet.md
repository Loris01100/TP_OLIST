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
