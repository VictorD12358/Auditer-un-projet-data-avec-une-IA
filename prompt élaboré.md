# Audit assisté par IA — Projet 6 OpenClassrooms « BottleNeck »

> Compte rendu des trois audits successifs réalisés avec une IA sur mon notebook du \*\*Projet 6 — Data Analyst OpenClassrooms\*\* (analyse des ventes et du stock du site e-commerce de vin \*BottleNeck\*).

\---

## Contexte

Le notebook traite **3 sources de données** :

|Source|Contenu|Volume|
|-|-|-|
|`erp.csv`|Produits (prix, stock, prix d'achat, `onsale\_web`…)|825 produits|
|`web.csv`|Catalogue WordPress|1 513 lignes|
|`liaison.csv`|Correspondance `product\_id` ↔ `id\_web`|825 lignes|

Le projet est organisé en **deux parties complémentaires** :

1. **Analyse de données** : exploration, indicateurs, outliers, statistiques, visualisations, conclusions métier ;
2. **Pipeline de données** (à partir de l'étape 6) : nettoyage, typage, jointures et production d'un dataset consolidé exploitable.

## Méthode

J'ai demandé à l'IA, dans le rôle d'un **Data Analyst senior**, trois audits successifs, chacun de plus en plus ciblé. Les consignes communes :

* être critique mais factuel, sans chercher artificiellement des défauts ;
* distinguer erreur, risque, choix discutable, choix justifié et amélioration facultative ;
* rester **proportionné** à un projet de formation (\~1 500 lignes) : pas de complexification ni de technologie ajoutée « pour faire pro » ;
* proposer au moins deux options comparées lorsque plusieurs approches sont raisonnables ;
* signaler une information manquante plutôt que l'inventer.

|#|Audit|Périmètre|Lien|
|-|-|-|-|
|1|Audit général|Notebook complet (code, analyse, pipeline, visualisations, conclusions)|[Aller à la partie 1](#partie-1)|
|2|Audit approfondi — Analyse de données|Qualité statistique, outliers, corrélations, analyses manquantes, visualisations|[Aller à la partie 2](#partie-2)|
|3|Audit approfondi — Pipeline|Étape 6 et suivantes : types, jointures, robustesse, validation, performances|[Aller à la partie 3](#partie-3)|

### Classification utilisée dans l'audit général

|Symbole|Catégorie|Signification|
|-|-|-|
|🔴|Erreur|Traitement incorrect ou résultat potentiellement faux|
|🟠|Risque / point à surveiller|Dépend du contexte, nécessite vérification ou documentation|
|🟡|Choix méthodologique discutable|Plusieurs approches possibles|
|🟢|Choix justifié|Cohérent avec l'objectif du projet|
|🔵|Amélioration facultative|Le projet fonctionne, mais peut gagner en lisibilité ou en présentation|

## 📑 Sommaire

* [Partie 1 — Audit général](#partie-1)
* [Partie 2 — Audit approfondi : Analyse de données](#partie-2)
* [Partie 3 — Audit approfondi : Pipeline (à partir de l'étape 6)](#partie-3)
* [Bilan global](#bilan)

\---

<a id="partie-1"></a>

# Partie 1 — Audit général du projet

## 1\. Compréhension générale du projet

Le notebook suit le template imposé par OpenClassrooms (texte d'intro, consignes en italique, cellules à compléter) et enchaîne :

* **Étapes 1-2** : chargement, exploration et nettoyage variable par variable (prix, stock, `onsale\_web`, `purchase\_price` côté ERP ; colonnes vides, SKU manquants, doublons côté web) ;
* **Étape 3** : jointures `erp → liaison → web` ;
* **Étapes 4-5** : analyse univariée du prix (Z-score, IQR), CA, quantités, stock (rotation), marge, corrélations ;
* **Étape 6** : re-typage (dates, textes, booléens, clés) et remerge final ;
* **Étape 5.6** : tentative d'export Excel.

Cette structure correspond bien à la logique *Partie 1 (analyse) / Partie 2 (pipeline)*, sauf qu'ici les deux parties sont **imbriquées dans les mêmes étapes numérotées** plutôt que séparées en deux sections explicites.

## 2\. Audit du code

### 🟠 Risques / points à surveiller

**a) `reset\_index()` appelé sans réassignation** (2 occurrences : `In \[282]` tri CA, `In \[298]` tri quantités)

```python
erp\_nettoye\_web\_nettoye\_liaison.reset\_index()   # résultat jeté, rien ne se passe
erp\_nettoye\_web\_nettoye\_liaison.head(20)
```

`reset\_index()` retourne un nouveau DataFrame et ne modifie rien en place. L'effet est visible dans `Out\[282]` : les 20 premières lignes affichent des index d'origine (76, 199, 321, 445…) et non 0-19. Le tri reste correct (les valeurs de CA sont bien décroissantes) : ce n'est **pas une erreur de calcul**, seulement un index resté « sale ». Incohérence avec l'Étape 5.3 (tri stock) où le `reset\_index` est bien réassigné.
➡️ Correction : `df = df.reset\_index(drop=True)`.

**b) Marge fortement négative non investiguée** (`In \[324]`, `product\_id` 4355)

Un produit affiche un `Taux\_marge` de **-665,6 %** (prix de vente 12,65 €, prix d'achat 77,48 €). Il est identifié et affiché, puis silencieusement exclu du calcul de la moyenne (`pos\_marge = ... > 0`) sans investigation. Le filtre `> 0` est pragmatique et acceptable, mais une ligne à -665 % mérite une phrase explicite : erreur de saisie ? colonnes inversées ? produit réellement soldé à perte ?

**c) Portée du dataset (714/825 produits) jamais rappelée dans une conclusion**

L'`inner join` erp ↔ liaison ↔ web ne conserve que **714 des 825 produits (86,5 %)**. Le choix est cohérent (on ne peut analyser le CA web que pour les produits présents sur le site), mais aucune phrase de synthèse ne rappelle que *« CA, marge et rotation de stock portent sur 714 produits, pas sur l'intégralité du catalogue »*.

**d) Conversion booléenne fragile** (`In \[391]`-`\[413]`)

```python
for i, row in web\_nettoye.iterrows():
    if pd.isna(row\['tax\_status']): web\_nettoye.loc\[i, 'tax\_status'] = False
    else: web\_nettoye.loc\[i, 'tax\_status'] = True
```

Ça fonctionne aujourd'hui car `tax\_status` ne contient que `NaN` / `'taxable'` et `post\_mime\_type` que `NaN` / `'image/jpeg'`. Mais la logique code « rempli = True » plutôt que « égal à la valeur attendue = True » : une nouvelle modalité serait mal interprétée sans erreur. Plus robuste : `== 'taxable'` / `== 'image/jpeg'`.

### 🟡 Choix méthodologiques discutables

**e) Boucles `iterrows()` omniprésentes pour des opérations vectorisables**

Exemples : vérification d'unicité de `product\_id` (liste + `.count()`, O(n²)), détection de nulles ligne à ligne, cohérence `in\_stock` / `stock\_quantity`, et les 5 conversions booléennes de (d). Sur 825-1 513 lignes ce n'est pas un problème de performance, mais c'est un point que les évaluateurs regardent en général (maîtrise de pandas vectorisé).

```python
web\_nettoye\['tax\_status'] = web\_nettoye\['tax\_status'].eq('taxable')
```

**f) Verbosité extrême de l'audit interne des doublons** (`In \[143]` à `In \[182]`, \~800 lignes de `print`)

La démarche est bonne (comparer les colonnes différentes, comprendre que ce sont des lignes *attachment* WordPress), mais l'exécution gagnerait à être condensée : un `.head()` ou un résumé chiffré (`print(f"{len(faux\_doublons)} faux doublons, tous liés à guid/post\_type")`) plutôt que la liste exhaustive.

**g) Découpage date/heure en colonnes séparées puis reconversion en `datetime`** (Étape 6.1)

`post\_date.str.split(" ")` puis reconstruction de 8 colonnes `datetime`, alors que `pd.to\_datetime(web\_nettoye\['post\_date'])` aurait suffi et conservé date + heure dans une seule colonne. Ce n'est pas faux, mais c'est une complexification sans bénéfice net.

### 🟢 Choix justifiés

* **Diagnostic des « faux doublons » SKU** (`In \[171]`-`\[189]`) : comparaison des colonnes différentes plutôt que suppression aveugle, identification des paires produit/attachment WordPress, confirmation par un second passage, puis déduplication propre. **C'est le point le plus solide du pipeline.**
* **Analyse Pareto 80/20 du CA** (`In \[289]`-`\[295]`) : %CA → cumul trié → seuil 80 % → proportion d'articles. Résultat cohérent (\~4,6 % des produits génèrent 80 % du CA).
* **Traitement du `stock\_quantity` négatif par `.clip(lower=0)`** plutôt que suppression des lignes : préserve la cohérence avec `in\_stock` sans perdre de produits.
* **`inner join` erp ↔ liaison ↔ web** : logique, car on veut uniquement les produits réellement vendables sur le site.

### 🔵 Améliorations facultatives

* Ajouter un titre / des labels aux `px.box()` (muets, contrairement aux `px.bar()` bien légendés).
* Factoriser les 5 boucles de conversion booléenne en une petite fonction réutilisable.
* Ajouter une cellule de conclusion générale en fin de notebook (CA total, top produits, taux de marge par catégorie, rotation de stock en 5-6 lignes pour un lecteur non technique).

## 3\. Audit de la partie « Analyse de données »

Les analyses répondent globalement bien aux questions attendues (distribution des prix, palmarès CA/quantités, marge, corrélations, stock). Les interprétations restent prudentes et distinguent corrélation et causalité, sauf une nuance : l'affirmation *« le nombre de ventes est positivement corrélé aux quantités en stock, ce qui est une bonne chose »* (`In \[335]`) mélange observation et jugement. Cette corrélation peut aussi refléter que les produits populaires sont davantage réapprovisionnés (**causalité inverse**). À nuancer d'une phrase.

## 4\. Audit des valeurs aberrantes

Bon traitement global :

* prix négatifs → valeur absolue, justifiée par l'hypothèse d'une faute de frappe ;
* stock négatif → `clip` à 0 ;
* marge à -665 % identifiée.

## 5\. Audit des visualisations

Bonnes pratiques dans l'ensemble : titres clairs, labels d'axes en français, angle des ticks pour la lisibilité, demi-heatmap de présentation. Faiblesses mineures : boxplots sans titre ; verbosité des `print()` de doublons qui nuit à la lisibilité générale du document.

## 6\. Tableau de priorisation

|Priorité|Problème|Partie|Localisation|Catégorie|Impact|Action|
|-|-|-|-|-|-|-|
|**P2**|Marge négative extrême non investiguée|Analyse|`In \[324]`|🟠 Risque|Conclusion potentiellement biaisée|Ajouter une phrase d'investigation / hypothèse|
|**P2**|Portée 714/825 jamais rappelée en conclusion|Analyse|Étapes 3-5|🟠 Risque|Sur-généralisation possible|Ajouter une phrase de cadrage|
|**P3**|`reset\_index()` sans réassignation (×2)|Analyse|`In \[282]`, `\[298]`|🟠 Risque mineur|Index non réinitialisé|`df = df.reset\_index(drop=True)`|
|**P3**|Boucles `iterrows()` vectorisables|Les deux|Multiples|🟡 Discutable|Lisibilité / idiomatisme pandas|Vectoriser avec `.isna()`, `.eq()`, masques booléens|
|**P3**|Verbosité des `print()` de doublons (\~800 lignes)|Pipeline|`In \[143]`-`\[182]`|🟡 Discutable|Lisibilité du livrable|Résumer par un compte + échantillon|
|**P3**|Boxplots sans titre|Analyse|`In \[238]`-`\[264]`|🔵 Optionnel|Présentation|Ajouter titre / labels|
|**P3**|Booléens basés sur « non-NaN » plutôt que valeur exacte|Pipeline|`In \[391]`-`\[413]`|🟡 Discutable|Fragilité si les données évoluent|`== 'valeur\_attendue'` explicite|

## 7\. Synthèse

**Les 2 améliorations les plus importantes**

1. Documenter / investiguer la marge à -665 % (P2).
2. Rappeler le périmètre 714/825 dans les conclusions présentées (P2).

**Points déjà solides**

* Diagnostic fin des doublons SKU (distinction produit / attachment WordPress).
* Traitement cohérent des incohérences stock / `in\_stock` et des prix négatifs.
* Analyse Pareto 80/20 bien construite.
* Interprétations généralement prudentes (corrélation ≠ causalité, sauf une nuance mineure).
* Séparation logique nettoyage / analyse / pipeline respectée dans l'esprit, même si l'articulation finale mérite d'être resserrée.

\---

<a id="partie-2"></a>

# Partie 2 — Audit approfondi : Analyse de données

> \*\*Consigne donnée à l'IA :\*\* se concentrer \*exclusivement\* sur la partie Analyse : reconstituer la logique de chaque analyse avant toute critique, évaluer la pertinence statistique, proposer des tests plus poussés \*\*avec au moins deux options comparées\*\*, examiner l'intérêt de librairies supplémentaires et des optimisations de performance \*\*au regard d'un volume de \~1 500 lignes\*\*, et ne rien ajouter qui ne soit réellement utile.

## 1\. Reconstitution de la logique analytique

|Analyse|Question métier|Donnée utilisée|Méthode|Résultat obtenu|Conclusion tirée|
|-|-|-|-|-|-|
|Distribution du prix|Le prix est-il propre et cohérent ?|`erp.price` (825 lignes)|Histogramme + KDE|Courbe décroissante asymétrique, valeurs négatives détectées|Faute de frappe supposée → `.abs()`|
|Distribution du stock|Le stock est-il cohérent ?|`erp.stock\_quantity`|Histogramme + KDE|Valeurs < 0 détectées|`.clip(lower=0)`|
|Cohérence `onsale\_web` / stock|Le catalogue web reflète-t-il le stock réel ?|`onsale\_web`, `stock\_quantity`, `in\_stock`|Boucle de comparaison ligne à ligne|\~50 incohérences (en vente mais stock à 0)|Jugé « non corrélé », variable conservée sans retraitement|
|Prix vs prix d'achat|La marge est-elle cohérente à l'échelle du catalogue ?|`price`, `purchase\_price`|Corrélation de Pearson|r = 0,969 ; p ≈ 0,0|« Corrélation significative » (seuil de test erroné, voir plus bas)|
|Outliers prix (Z-score)|Quels produits ont un prix extrême ?|`price` (825 lignes, `erp\_nettoye`)|Z = (x−μ)/σ, seuil de 3 sur la valeur absolue|Seuil ≈ 116,4 €|Pas de suppression, identification seule|
|Outliers prix (IQR)|Idem, méthode alternative|`price` (714 lignes, dataset joint)|Lecture visuelle d'un boxplot Plotly|Seuil ≈ 83 € ; 4,48 % du catalogue outlier|« Semble cohérent », recommande une comparaison à des sources externes|
|CA par article + palmarès|Quels produits génèrent le plus de CA ?|`price × total\_sales` (714 lignes) *(formule déduite, ligne tronquée dans le PDF)*|Tri décroissant, top 20|CA total = 153 748,10 €|Bar chart top 20|
|Concentration du CA (Pareto 80/20)|Le CA est-il concentré sur peu de produits ?|%CA cumulé trié|Seuil cumulé à 80 %|4,62 % des produits font 80 % du CA|Concentration forte confirmée, mais jamais visualisée en graphique Pareto|
|Quantités vendues|Quels produits se vendent le plus en volume ?|`total\_sales`|Tri décroissant, top 20|Max 122 ventes|Bar chart top 20|
|Rotation de stock|Quels produits dorment en stock ?|`stock\_quantity`, `total\_sales` *(formule exacte tronquée dans le PDF)*|`Mois\_stock`, clip à 0|Top 20 avec le plus de mois de stock (jusqu'à 31,2 mois)|Bar chart, pas de seuil d'alerte défini|
|Taux de marge|Quelle marge pour le catalogue ? Par catégorie ?|`price`, `purchase\_price` → `Taux\_marge`|Min / max / moyenne, puis moyenne par `product\_type` (7 catégories)|Un produit à -665,6 % (isolé), marge positive moyenne 35,2 %, de 21,9 % (huile d'olive) à 42,8 % (cognac)|Bar chart des moyennes par catégorie, sans effectif ni dispersion|
|Corrélations stock / ventes / prix|Comment ces 3 variables interagissent-elles ?|`stock\_quantity`, `total\_sales`, `price`|Matrice de corrélation Pearson + heatmap|Prix ↔ ventes négatif ; stock ↔ ventes positif ; stock ↔ prix ≈ nul|Interprétée sans test de significativité ni vérification de linéarité|

## 2\. Audit de la qualité analytique, analyse par analyse

**Prix vs prix d'achat (Pearson)** — Pertinent sur le principe (relation linéaire attendue entre coût et prix de vente), mais :
(a) le seuil du test de significativité est erroné (signalé dans le tableau ci-dessus) ;
(b) aucune vérification de linéarité (un simple scatter suffirait) ni de la distribution des résidus ;
(c) le prix étant fortement asymétrique (moyenne 32,33 € > médiane 23,45 €), Pearson mesure une relation *linéaire* et peut sous-estimer une relation simplement monotone non linéaire.

**Z-score sur le prix** — Il suppose une distribution approximativement normale. Or le notebook décrit lui-même la distribution du prix comme « asymétrique dégressive » : c'est une violation d'hypothèse non discutée. Avec une telle asymétrie, un seuil de 3 est trop permissif à droite (la queue est longue et légitime : bouteilles chères) et ce compromis n'est pas mentionné.

**Marge par catégorie (`product\_type`)** — C'est la comparaison la plus fragile : 7 catégories dont on ne connaît pas les effectifs (aucun `value\_counts()` dans le notebook). Si « Huile d'olive » ou « Gin » ne comptent que 2-3 produits, une moyenne à 21,9 % ou 40,4 % n'a quasiment aucune valeur statistique. Aucune dispersion (écart-type, IC) n'est reportée. **Point le plus important à renforcer.**

**Corrélations stock / ventes / prix** — Bonne intuition d'interprétation, mais (1) c'est une lecture de corrélation, pas un test de significativité : aucune p-value n'accompagne la heatmap ; (2) l'interprétation causale implicite (« c'est une bonne chose ») mériterait la nuance de causalité inverse (le stock élevé peut être une conséquence du succès commercial).

**CA / quantités vendues / rotation de stock** — Analyses descriptives simples et bien exécutées. Seul manque réel : la Pareto (80/20), calculée numériquement, n'est jamais visualisée en vrai graphique Pareto (barres + courbe cumulée), alors que `%CA\_cumul` existe déjà.

## 3\. Renforcements statistiques proposés

### A. Comparer la marge moyenne entre catégories de produit

**Analyse concernée :** `pos\_marge.groupby('product\_type')\['Taux\_marge'].mean()` — comparaison de 7 moyennes sans test ni effectifs.
**Problème :** sans test ni effectif par groupe, on ne peut pas dire si les écarts (21,9 % à 42,8 %) sont réels ou dus au hasard / à un petit échantillon.

|Option|Méthode|Pourquoi pertinente|Avantages|Limites|Complexité|Pertinence projet|
|-|-|-|-|-|-|-|
|1|ANOVA à un facteur (ou Welch ANOVA si variances inégales)|Compare plusieurs moyennes simultanément|Standard, `scipy.stats.f\_oneway` ou `pingouin.welch\_anova`|Suppose normalité par groupe et homogénéité des variances (pour l'ANOVA classique) — à vérifier avec seulement quelques dizaines de produits par catégorie|Faible|Haute si les effectifs par catégorie sont ≥ \~10|
|2|Kruskal-Wallis + test post-hoc de Dunn|Ne suppose pas la normalité, robuste aux petits échantillons et à l'asymétrie|Plus sûr avec des groupes de taille inégale ou faible (« Huile d'olive », « Gin »)|Moins puissant si les hypothèses de l'ANOVA sont en fait respectées|Faible-modérée (`scipy.stats.kruskal`, `scikit\_posthocs.posthoc\_dunn`)|Haute — probablement l'option la plus sûre vu les effectifs inconnus|

**Intégration :** commencer par `erp\_nettoye\_web\_nettoye\_liaison\['product\_type'].value\_counts()` (gratuit, répond à la question des effectifs). Si tous les groupes ont ≥ 10-15 produits et une distribution pas trop asymétrique, l'ANOVA suffit ; sinon Kruskal-Wallis + Dunn est plus défendable. *Impact : qualité statistique + robustesse.*

### B. Relation prix ↔ prix d'achat (et corrélations en général)

|Option|Méthode|Pourquoi pertinente|Avantages|Limites|Complexité|Pertinence projet|
|-|-|-|-|-|-|-|
|1|Garder Pearson, corriger le seuil, ajouter un scatter avec droite de régression|Simple, déjà en place|Aucun changement de méthode, gain immédiat|Reste sensible à l'asymétrie du prix|Très faible|Haute (fix rapide indispensable)|
|2|Ajouter Spearman en complément (`scipy.stats.spearmanr` ou `df.corr(method='spearman')`)|Relation monotone, insensible à l'asymétrie et aux extrêmes|Robuste, facile à ajouter à côté de Pearson|Pas de pente interprétable comme une régression|Très faible|Modérée-haute : montre que la conclusion tient avec une méthode robuste|

**Recommandation :** recalculer Spearman à côté de Pearson pour toutes les paires déjà étudiées (prix/`purchase\_price`, stock/ventes, prix/ventes) et commenter en une phrase si les deux méthodes concordent. *Impact : qualité statistique.*

### C. Détection des valeurs aberrantes de prix

|Option|Méthode|Pourquoi pertinente|Avantages|Limites|Complexité|Pertinence projet|
|-|-|-|-|-|-|-|
|1 *(déjà en place)*|Z-score / IQR sur `price` seul|Simple|Facile à expliquer à un non-technicien|Univarié, Z-score sensible à l'asymétrie, seuil IQR lu à l'œil|—|—|
|2|**MAD** (Median Absolute Deviation)|Robuste à l'asymétrie et aux extrêmes eux-mêmes (le Z-score utilise moyenne et écart-type, sensibles aux outliers recherchés)|Peu de code (`1.4826 \* median(abs(x - median(x)))`), cohérent avec l'asymétrie constatée|Moins connu, à expliquer en une phrase|Faible|Modérée-haute : amélioration la plus justifiée méthodologiquement|
|3 *(optionnelle)*|Isolation Forest multivarié (`price` + `stock\_quantity` + `total\_sales`)|Détecte des anomalies combinées (ex. produit cher, jamais vendu, gros stock)|Capture des profils atypiques, pas seulement des prix extrêmes|Boîte noire relative, seuil de contamination arbitraire, overkill si l'objectif est « le prix est-il propre »|Modérée (`sklearn.ensemble.IsolationForest`)|Faible à modérée — disproportionné pour un Projet 6|

**Recommandation :** garder IQR / Z-score pour le nettoyage, ajouter MAD en complément avec une phrase du type *« MAD confirme / nuance le seuil du Z-score, ce qui est attendu vu l'asymétrie »*. *Impact : robustesse + qualité statistique.*

### D. Transformation logarithmique du prix

Le prix est asymétrique à droite (moyenne 32,33 > médiane 23,45, max 225). Une transformation `log1p(price)` avant Z-score rendrait le test plus valide et les histogrammes / boxplots plus lisibles (aujourd'hui, la poignée de bouteilles à 150-225 € écrase le reste). *Impact : qualité statistique + visualisation.*

## 4\. Librairies Python complémentaires

|Librairie|Utilisation possible|Gain attendu|Complexité|Avantage vs solution actuelle|Inconvénient|
|-|-|-|-|-|-|
|`pingouin`|Remplacer les appels `scipy.stats` séparés par des fonctions renvoyant directement r, p, IC95 %, taille d'effet (`pg.corr()`, `pg.welch\_anova()`, `pg.kruskal()`)|**Modéré** : améliore la lisibilité et la complétude du reporting|Faible|Beaucoup moins de code répétitif|Dépendance supplémentaire|
|`statsmodels`|Régression linéaire `price \~ purchase\_price` avec résumé complet (R², résidus, diagnostics)|**Modéré** si régression ajoutée|Faible-modérée|Diagnostic bien plus riche que `pearsonr`|Overkill si on ne veut qu'un coefficient|
|`scikit-learn`|Uniquement pour Isolation Forest (option C.3)|**Faible à modéré**|Modérée|Seule option pratique pour Isolation Forest|Inutile avec IQR / Z-score / MAD|
|`polars`, `pyarrow`, `duckdb`, `numba`|Remplacer pandas pour la vitesse|**Quasi nul** sur \~1 500 lignes|—|Aucun gain perceptible|Complexité ajoutée pour rien|

> \*\*En clair :\*\* `pingouin` est la seule librairie qui apporte un gain net et immédiat. Le reste est soit optionnel, soit sans intérêt à ce volume.

## 5\. Performances

Sur \~1 500 lignes, **aucune optimisation n'est perceptible** en temps d'exécution, `iterrows()` inclus. Les points suivants relèvent de la lisibilité et de la maintenabilité, pas de la vitesse :

|Traitement|Pourquoi optimisable|Solution|Gain réel à 1 500 lignes|
|-|-|-|-|
|Cohérence `onsale\_web` / `stock\_quantity` par boucle|Boucle Python ligne à ligne|Masque booléen : `mask = (df.onsale\_web) \& (df.stock\_quantity < 1)`|Nul en vitesse, gain de lisibilité|
|Conversions booléennes (5 boucles `iterrows`)|Idem|`.notna()` ou `.eq(valeur)` vectorisé|Nul en vitesse|
|Unicité de `product\_id` (liste + `.count()`)|O(n²), mais n = 825|`df\['product\_id'].duplicated().any()` ou `df\['product\_id'].is\_unique`|Nul en vitesse, mais O(n²) est un mauvais réflexe à corriger|

## 6\. Valeurs aberrantes — synthèse

Z-score et IQR, appliqués uniquement à `price`, ne « voient » pas la même chose (seuil ≈ 116 € vs ≈ 83 €) : l'IQR est robuste à l'asymétrie alors que le Z-score est tiré vers la droite par les valeurs extrêmes elles-mêmes. Cette différence n'est pas un problème, **c'est une information** : elle confirme l'asymétrie et montre que l'IQR est probablement la mesure la plus fiable ici. Une valeur aberrante en prix (bouteille chère) n'est pas nécessairement une erreur : **aucune suppression arbitraire à faire**.

**Présentation recommandée :** garder toutes les observations et ajouter une colonne booléenne `is\_outlier` plutôt que de filtrer.

## 7\. Corrélations — synthèse

Pearson est adapté pour prix ↔ `purchase\_price` (relation attendue linéaire et forte : r = 0,97). Pour stock ↔ ventes ↔ prix, vu l'asymétrie de `price` et `total\_sales`, Spearman en complément renforcerait la conclusion. Aucune p-value n'accompagne la heatmap : à ajouter au moins pour la paire la plus commentée (stock ↔ ventes) via `pearsonr()` / `spearmanr()`.

## 8\. Analyses supplémentaires à envisager

|Analyse proposée|Question métier|Variable disponible|Méthode|Résultat exploitable|
|-|-|-|-|-|
|CA et quantités par catégorie|Quelles *catégories* génèrent le CA ?|`product\_type`, `CA`, `total\_sales`|`groupby('product\_type')\[\['CA','total\_sales']].sum()` + bar chart|Aujourd'hui seule la marge est ventilée par catégorie|
|Croisement marge × rotation de stock|Quels produits sont rentables ET à réassortir en priorité, lesquels sont à liquider ?|`Taux\_marge`, `Mois\_stock`|Scatter `Mois\_stock` vs `Taux\_marge`, quadrants|Priorisation actionnable du catalogue|
|Corrélation prix ↔ ventes par catégorie|La relation globale masque-t-elle un paradoxe de Simpson ?|`product\_type`, `price`, `total\_sales`|Pearson / Spearman par groupe|Vérifie la robustesse de la conclusion globale|
|Analyse temporelle|Les produits anciens se vendent-ils différemment ?|`post\_date` (date de mise en ligne)|Corrélation ancienneté ↔ `total\_sales`|⚠️ Pas de date de transaction : une vraie tendance de ventes est **impossible** avec ces données. Proxy faible, à présenter comme une limite|

## 9\. Visualisations — audit analytique

|Graphique actuel|Répond-il à la question ?|Alternative|
|-|-|-|
|Histogramme prix + KDE|Oui, correct|KDE en échelle log-x pour mieux voir la masse des prix bas|
|Boxplot prix (Plotly)|Oui, mais montre juste les quartiles|Violin plot : forme réelle de la distribution asymétrique|
|Scatter Z-score vs prix|Correct (\~700 points, hexbin inutile)|—|
|Bar chart top 20 CA|Répond à « quels produits », pas à « quelle concentration »|**Vrai graphique Pareto** (barres CA + courbe %CA cumulé sur second axe) — les données existent déjà|
|Bar chart marge moyenne par catégorie|Cache la dispersion et les effectifs|Boxplot / strip plot de `Taux\_marge` par `product\_type`, ou bar chart + barres d'erreur (IC95 %)|
|Heatmap de corrélation|Correct|—|

> Le point le plus rentable : \*\*tracer enfin le graphique Pareto\*\*, tout le calcul est déjà fait.

## 10\. Robustesse des conclusions

|Conclusion|Solidité|Recommandation|
|-|-|-|
|« Stock ↔ ventes est une bonne chose »|Descriptive, pas causale|Reformuler en « associé positivement », sans jugement causal|
|« Marge moyenne par catégorie »|**La moins robuste du notebook**|Ne pas présenter comme un fait solide tant que `value\_counts()` et un test (ANOVA / Kruskal-Wallis) ne l'ont pas confirmé|
|« 4,62 % des produits font 80 % du CA »|Robuste, calcul simple et vérifiable|Aucune|

## 11\. Tableau de priorisation

|Priorité|Analyse concernée|Amélioration|Type|Options|Gain attendu|Complexité|
|-|-|-|-|-|-|-|
|**P1**|Marge par catégorie|Effectifs (`value\_counts`) + test (ANOVA / Kruskal-Wallis)|Statistique|ANOVA vs Kruskal-Wallis|Fort : évite une conclusion non fondée|Faible|
|**P2**|Corrélations (toutes)|Spearman en complément de Pearson|Statistique|Spearman|Modéré|Très faible|
|**P2**|Détection d'outliers prix|MAD en complément de Z-score / IQR|Statistique|MAD, +Isolation Forest|Modéré|Faible|
|**P2**|CA / concentration|Vrai graphique Pareto|Visualisation|1|Fort en présentation, données prêtes|Très faible|
|**P2**|Segmentation|CA et quantités par `product\_type`|Métier|1|Fort en valeur métier|Faible|
|**P3**|Prix (histogrammes / Z-score)|Transformation log|Statistique / visualisation|1|Modéré|Faible|
|**P3**|Reporting statistique|`pingouin`|Librairie|1|Modéré (lisibilité)|Faible|
|**P3**|Boucles `iterrows`|Vectoriser|Performance / code|1|Nul en vitesse, gain de lisibilité|Faible|
|—|polars / duckdb / numba|—|Performance|—|Quasi nul — non recommandé|—|

## 12\. Synthèse finale

**A. Analyses à conserver** — CA et palmarès top 20, quantités vendues, rotation de stock, calcul Pareto 80/20 (le calcul, pas encore sa visualisation), corrélation Pearson prix / `purchase\_price`, traitement des prix et stocks négatifs.

**B. Analyses à renforcer** — Marge par catégorie (effectifs + test manquants) ; corrélations stock / ventes / prix (Spearman + p-values) ; détection d'outliers (MAD, log).

**C. Tests statistiques possibles** — Kruskal-Wallis + Dunn (ou ANOVA / Welch ANOVA) pour la marge par catégorie ; Spearman en complément de Pearson.

**D. Librairies** — `pingouin` : utile mais facultative. `statsmodels` / `scikit-learn` : utiles seulement si régression ou détection multivariée ajoutées. `polars` / `pyarrow` / `duckdb` / `numba` : inutiles sur ce volume.

**E. Performances** — Aucune optimisation ne donne de gain mesurable à \~1 500 lignes ; les vectorisations sont des gains de lisibilité.

**F. Analyses supplémentaires** — CA et quantités par catégorie ; croisement marge × rotation de stock ; prix/ventes par catégorie (paradoxe de Simpson). L'analyse temporelle des ventes n'est pas réalisable (pas de date de transaction).

**G. Les améliorations prioritaires**

1. **Effectifs + test statistique** (Kruskal-Wallis ou ANOVA) sur la marge par catégorie : sans cela, la conclusion « le cognac a une meilleure marge que l'huile d'olive » n'est pas démontrée. *Améliore l'Étape 5.4.*
2. **Tracer le graphique Pareto** (barres + courbe cumulée) : donnée déjà calculée. *Étape 5.1.*
3. **Ajouter Spearman en complément de Pearson** sur toutes les corrélations : renforce la robustesse à peu de frais. *Étapes 2.1.1.4 et 5.5.*
4. **Ventiler CA et quantités vendues par catégorie de produit**, pas seulement la marge : lecture métier par famille de produits actuellement absente.

\---

<a id="partie-3"></a>

# Partie 3 — Audit approfondi : Pipeline (à partir de l'étape 6)

> \*\*Consigne donnée à l'IA :\*\* se concentrer \*exclusivement\* sur le pipeline à partir de l'étape 6 : reconstituer son fonctionnement avant toute proposition, auditer jointures, types, valeurs manquantes, doublons, robustesse et validation, comparer pandas aux alternatives (Polars, DuckDB, PyArrow) \*\*sans recommander de technologie plus complexe si le gain est nul à \~1 500 lignes\*\*, et distinguer améliorations utiles, optimisations inutiles et complexité inutile.

## 1\. Vue synthétique du pipeline actuel

|Étape|Entrée|Traitement|Sortie|Objectif|
|-|-|-|-|-|
|**6.1 — Dates**|`web\_nettoye` (714 lignes, texte brut pour `post\_date`, `post\_date\_gmt`, `post\_modified`, `post\_modified\_gmt`)|Split date/heure par `.str.split(" ")`, suppression des colonnes brutes, `pd.to\_datetime()` de chaque partie|8 colonnes `datetime64\[ns]`|Typer correctement les dates|
|**6.2 — Textes**|`web\_nettoye`|`post\_title`, `post\_name`, `product\_type` → `astype("string")` ; `post\_excerpt` nettoyé du HTML via BeautifulSoup puis `astype("string")`|Colonnes texte en `string\[python]`, `post\_excerpt` en texte brut|Typer et nettoyer les champs texte|
|**6.3 — Booléens**|`web\_nettoye`|5 boucles `iterrows()` : `tax\_status`, `posted`, `closed\_comment`, `closed\_ping`, `image\_jpeg` → True/False selon NaN / non-NaN|Colonnes en `bool`|Typer les indicateurs binaires|
|**6.4 — Clés**|`web\_nettoye`, `liaison`|`id\_web` → `astype("string")` dans les deux tables|Clés de jointure homogènes|Sécuriser la jointure suivante|
|**6.5 — Remerge**|`erp\_nettoye` (825), `liaison` (825), `web\_nettoye` (714)|`merge(erp\_nettoye, liaison, how='inner', on='product\_id')` → 825 lignes dont 734 avec `id\_web` ; puis merge avec `web\_nettoye` → 714 lignes|`erp\_nettoye\_web\_nettoye\_liaison2`, bien typé mais sans `CA` / `Taux\_marge` / `Mois\_stock` / `z\_price` / `%CA`|Produire un dataset final propre et typé|

## 2\. Audit d'architecture

**Opérations redondantes** — Le pipeline fait deux fusions erp ↔ liaison ↔ web : une à l'Étape 3 (pour l'analyse) et une seconde, quasi identique, à l'Étape 6.5 (pour le typage). C'est la **redondance la plus significative du notebook** : même logique de jointure écrite deux fois, sur des tables légèrement différentes, sans commentaire explicatif.

**Ordre des opérations** — Typer `web\_nettoye` avant la jointure finale est le bon ordre. Ce qui pose problème, c'est de repartir de `erp\_nettoye` / `liaison` bruts en 6.5 au lieu de la version déjà jointe et enrichie.

## 3\. Qualité des données et jointures

### Jointure 1 — `erp\_nettoye` ↔ `liaison` (Étape 6.5, `on='product\_id'`, `how='inner'`)

|Contrôle|Fait actuellement|Risque|Comment vérifier|
|-|-|-|-|
|Lignes sans correspondance|Vérifiées via `.count()` (825 → 734 sur `id\_web`)|Aucun, bon réflexe|—|
|Cause des 91 lignes sans `id\_web`|Non investiguée|Produits volontairement absents du site (B2B ?) ou problème de `liaison.csv` ?|Anti-jointure : `erp\_nettoye\[\~erp\_nettoye\['product\_id'].isin(liaison\['product\_id'])]`|

|Option|Principe|Avantages|Limites|Pertinence|
|-|-|-|-|-|
|1|`merge(..., validate="one\_to\_one")`|Plante immédiatement si l'hypothèse de cardinalité est fausse|Ne dit pas *pourquoi*|Haute : coût quasi nul, sécurité forte|
|2|Contrôle explicite avant jointure (`duplicated().sum()` sur les deux clés)|Pédagogique, montre la démarche à l'évaluateur|Un peu plus verbeux|Haute : bon complément de l'option 1|

### Jointure 2 — `erp\_nettoye\_liaison2` ↔ `web\_nettoye` (Étape 6.5, `on='id\_web'`)

Même lacune : pas de `validate=`, pas d'anti-jointure sur les **20 lignes perdues** entre 734 (`id\_web` renseigné) et 714 (jointure réussie). Ce delta n'est jamais expliqué (typo, produit supprimé du site ?).

```python
erp\_nettoye\_liaison2\[\~erp\_nettoye\_liaison2\['id\_web'].isin(web\_nettoye\['id\_web'])]
```

**Cardinalité globale à documenter en une phrase :** 825 produits ERP → 734 avec un `id\_web` → 714 retrouvés dans le catalogue web (**86,5 %**). À écrire noir sur blanc en conclusion de l'Étape 6.5.

## 4\. Transformations pandas (Étape 6)

|Traitement actuel|Pourquoi améliorable|Option 1|Option 2|Gain|Pertinence à 1 500 lignes|
|-|-|-|-|-|-|
|5 boucles `iterrows()` pour les booléens|Non vectorisé, même pattern collé 5 fois|`df\[col] = df\[col].notna()`|Petite fonction `to\_bool` appliquée à une liste de colonnes|Quasi nul en vitesse, **fort en lisibilité** (5 blocs de 6 lignes → 5 lignes)|Amélioration de code, pas de performance|
|Split date en 2 étapes répété 4 fois|`pd.to\_datetime()` parse directement `"2018-02-12 12:46:23"` en une colonne|`df\['post\_date'] = pd.to\_datetime(df\['post\_date'])`|Extraire après coup avec `.dt.date` / `.dt.time` si besoin|Quasi nul en vitesse ; schéma réduit de 8 à 4 colonnes|Pertinent pour la clarté du schéma final|
|`.apply(lambda x: BeautifulSoup(...))` sur `post\_excerpt`|—|—|—|—|✅ **Bon usage de `.apply()`** : pas d'équivalent vectorisé, rien à changer|

⚠️ **Point de qualité de données à ne pas manquer.** Les colonnes `post\_date\_hour`, `post\_date\_hour\_gmt`, `post\_modified\_hour`, `post\_modified\_hour\_gmt` sont obtenues en convertissant une heure seule (ex. `"12:46:23"`) via `pd.to\_datetime()`. Pandas lui attache alors une **date par défaut arbitraire** (la date du jour d'exécution). Ces 4 colonnes ne doivent jamais servir à un calcul de durée ou de comparaison de dates. Ce n'est pas documenté dans le notebook : à signaler en commentaire ou, mieux, à éviter avec une seule colonne `datetime` par variable de date.

## 5\. Performances — Étape 6

|Technologie|Problème résolu|Gain attendu|Complexité ajoutée|Pertinence à \~1 500 lignes|
|-|-|-|-|-|
|Vectorisation pandas|Lisibilité, pas vitesse|**Quasi nul** en temps d'exécution|Aucune (c'est même plus simple)|Pertinent pour la qualité de code|
|PyArrow|Rien de concret ici|**Quasi nul**|Modérée|Non pertinent|
|Polars|Remplacer les 2 `merge()` et les conversions|**Quasi nul**|Migration complète de l'API|Non recommandé|
|DuckDB|Jointures en SQL|**Quasi nul** (éventuel confort si on préfère le SQL)|Nouvelle dépendance et changement de paradigme|Non recommandé|

> \*\*Verdict :\*\* sur ce volume, aucune de ces technologies n'apporte de gain mesurable. Seules les vectorisations pandas simples ont du sens, motivées par la lisibilité.

## 6\. Pandas reste-t-il le bon choix ?

* **pandas (actuel)** — API maîtrisée, lisible, standard pour ce type de projet, largement documentée pour les évaluateurs. Aucune limite rencontrée dans ce pipeline. Pertinence : totale.
* **Polars** — Avantages théoriques (parallélisme, API expressive, gros volumes). Migration non triviale (syntaxe différente), aucun gain perceptible. Intérêt réel : nul.
* **DuckDB** — Pour les jointures : question de goût (SQL vs pandas), pas de performance. Aucun gain mesurable sur agrégations / CSV / Parquet à ce volume.
* **PyArrow** — `string\[pyarrow]` vs `string\[python]` : marginal. Ne deviendrait intéressant qu'avec des millions de lignes ou une interopérabilité Parquet poussée.

**Conclusion : pandas est le bon choix, sans réserve, pour ce projet.**

## 7\. Audit des types de données

|Colonne(s)|Type actuel|Optimisation envisageable|Intérêt réel à 1 500 lignes|
|-|-|-|-|
|`post\_title`, `post\_excerpt`, `post\_name`|`string\[python]`|`category` non pertinent (valeurs quasi toutes uniques)|Déjà le bon choix|
|`product\_type`|`string\[python]`|`category` plus adapté : 7 valeurs distinctes (Vin, Whisky, Huile d'olive, Champagne, Cognac, Gin, Autre) sur 714 lignes|Gain mémoire quasi nul, mais gain de **sémantique** (variable catégorielle, utile pour `groupby`). Facultatif, peu coûteux : `astype("category")`|
|Booléens (`tax\_status`, `posted`…)|`bool`|—|Déjà optimal|
|Dates|`datetime64\[ns]`|—|Déjà optimal, hors remarque sur les colonnes « heure seule »|
|`id\_web`|`string\[python]`|—|Bon choix pour une clé de jointure (évite les ambiguïtés int / float / NaN d'un `object`)|

Le passage `object` → `string\[python]` est une bonne pratique moderne. `category` ne serait pertinent que pour `product\_type`, pour la clarté sémantique et non la mémoire.

## 8\. Valeurs manquantes et doublons

**Valeurs manquantes** — Le typage booléen transforme implicitement tout `NaN` en `False` (« non rempli » = « non »). C'est défendable ici car chaque colonne n'a que 2 modalités possibles (valeur ou NaN), mais ce n'est **pas de l'imputation classique** : on redéfinit la sémantique de la colonne. À mentionner comme un choix de modélisation. Aucune imputation moyenne / médiane / mode n'est utilisée dans cette partie, ce qui est cohérent.

**Doublons** — Déduplication faite en amont (Étape 3) : rien de nouveau à signaler à l'Étape 6, il est cohérent de ne pas la refaire.

## 9\. Robustesse face à des données légèrement différentes

|Scénario|Le pipeline résiste-t-il ?|Pourquoi|
|-|-|-|
|Nouvelle modalité dans `tax\_status` ou `post\_mime\_type` (ex. `'reduced-rate'`, `'image/png'`)|❌ Non|`notna()` la traiterait comme `True` sans distinction|
|`liaison.csv` contient un `product\_id` dupliqué|❌ Non détecté|Aucun `validate=` ni contrôle d'unicité|
|Date mal formée dans `post\_date`|⚠️ Partiellement|Pas de `errors='coerce'` explicite ni de contrôle du nombre de `NaT`|
|Chemin de fichier différent sur une autre machine|❌ Non|Chemin Windows en dur (`os.chdir("C:\\\\Users\\\\...")`)|

**Recommandations proportionnées** (pas de pandera / Great Expectations ici) :

```python
# Après conversion des dates :
n\_nat = web\_nettoye\['post\_date2'].isna().sum()
assert n\_nat == 0, f"{n\_nat} dates non converties"

# Avant les merges :
assert liaison\['product\_id'].is\_unique, "product\_id dupliqué dans liaison"
assert erp\_nettoye\_web\_nettoye\_liaison2\['product\_id'].is\_unique
```

## 10\. Validation automatique

|Option|Ce qu'elle permet|Avantages|Limites|Complexité|Pertinence|
|-|-|-|-|-|-|
|`assert` pandas simples|Unicité de clé, absence de `NaT`, cardinalité|Aucune dépendance, 100 % lisible, intégrable en 10 min|Ne documente pas un schéma complet|Très faible|✅ Suffisant et recommandé|
|`merge(..., validate=...)`|Vérifie la cardinalité déclarée d'une jointure|Plante immédiatement si l'hypothèse est fausse, zéro ligne en plus|Ne remplace pas un contrôle de valeurs manquantes|Nulle|✅ Recommandé, gratuit|
|`pandera`|Schéma de validation déclaratif (types, plages, unicité)|Documentation vivante du schéma|Dépendance, courbe d'apprentissage, disproportionné|Modérée|Facultatif|
|Great Expectations|Suite complète de validation avec rapports HTML|Très complet, orienté production|Très lourd pour 1 500 lignes|Élevée|❌ Non recommandé|

## 11\. Librairies supplémentaires (pipeline)

|Librairie|Utilisation concrète|Problème résolu|Gain|Complexité|Alternative|
|-|-|-|-|-|-|
|`pathlib`|Remplacer les chemins Windows en dur (`os.chdir(...)`)|Reproductibilité inter-machines|**Modéré** : corrige un vrai problème de portabilité|Très faible|`os.path` (fonctionnel, moins idiomatique)|
|`tqdm`|Barre de progression sur les `iterrows` restants|Confort visuel|**Faible / quasi nul**|Très faible|Aucun intérêt réel|
|`pandera` / Great Expectations|Voir section 10|—|Facultatif à quasi nul|—|`assert` suffisant|

## 12\. Modularisation

**Situation actuelle :** les 5 boucles booléennes (6.3) et les 4 blocs de split de date (6.1) sont des copier-coller quasi identiques.

**Option 1 — une petite fonction :**

```python
def to\_bool(series):
    return series.notna()

for col in \['tax\_status', 'posted', 'closed\_comment', 'closed\_ping', 'image\_jpeg']:
    web\_nettoye\[col] = to\_bool(web\_nettoye\[col])
```

Réutilisable, lisible, testable en une ligne (`assert to\_bool(pd.Series(\[None, 1])).tolist() == \[False, True]`). Pertinence : haute.

**Option 2 — une petite fonction par étape** (`nettoyer\_dates(df)`, `nettoyer\_booleens(df)`…, appelées séquentiellement) : améliore la lisibilité globale mais change davantage la structure. Intéressant, facultatif pour un Projet 6. Pas de classes ni de fichier `.py` séparé : disproportionné.

## 13\. Reproductibilité, lisibilité, fichiers

* **Nommage :** globalement cohérent (`\_nettoye`, `\_liaison`, suffixe `2` pour la version retypée), mais le suffixe `2` communique mal l'intention : un nom comme `erp\_web\_liaison\_final` serait plus explicite.
* **CSV vs Parquet / PyArrow pour l'export :** à ce volume (714 lignes, quelques dizaines de Ko), Excel / CSV est largement suffisant et plus lisible par un non-technicien. Parquet n'apporterait aucun bénéfice.

## 14\. Mesurer le gain (si besoin en soutenance)

```python
%%timeit
web\_nettoye\['tax\_status'].notna()   # version vectorisée
```

à comparer à la boucle `iterrows()` équivalente. `%%timeit` est la méthode la plus simple à intégrer dans une cellule Jupyter ; `time.perf\_counter()` convient pour mesurer un bloc de plusieurs lignes. `memory\_profiler` est inutile ici (aucun enjeu mémoire).

## 15\. Tableau de priorisation

|Priorité|Étape|Amélioration|Type|Options|Gain attendu|Complexité|Pertinence|
|-|-|-|-|-|-|-|-|
|**P1**|6.5|`validate=` sur les deux `merge()` + anti-jointures pour documenter les lignes perdues|Qualité des données|`validate="one\_to\_one"`, anti-jointure|Fort|Très faible|Indispensable|
|**P2**|6.1|Une seule colonne `datetime` par variable de date|Lisibilité / code|`pd.to\_datetime()` direct|Modéré (lisibilité), quasi nul (vitesse)|Faible|Recommandé|
|**P2**|6.3|Vectoriser les 5 boucles booléennes + factoriser|Code / lisibilité|`.notna()` + fonction|Quasi nul (vitesse), fort (lisibilité)|Très faible|Recommandé|
|**P3**|6.4|`product\_type` → `category`|Type / sémantique|`astype("category")`|Quasi nul (mémoire), léger (clarté)|Très faible|Facultatif|
|**P3**|6.1|Documenter la date arbitraire des colonnes « hour »|Documentation|Commentaire ou suppression|Évite une confusion future|Nulle|Recommandé, mineur|
|—|—|Migrer vers Polars / DuckDB / PyArrow|Performance|—|Quasi nul|Élevée|❌ Non recommandé|

## 16\. Synthèse finale

**A. Architecture actuelle** — Nettoyage → jointure d'analyse enrichie (Étapes 3-5) → retypage strict (6.1-6.4) → nouvelle jointure depuis les tables sources brutes (6.5) → export jamais atteint.

**B. Points forts** — Typage moderne et cohérent (`string\[python]`, `bool`, `datetime64\[ns]`) ; bon réflexe de vérifier les comptages après jointure ; usage justifié de `.apply()` pour le nettoyage HTML ; auto-limitation assumée sur le nettoyage du texte.

**C. Robustesse** — `validate=` sur les merges, anti-jointures documentées, `assert` ciblés (unicité de clé, absence de `NaT`).

**D. Librairies** — `pathlib` : utile immédiatement. `pandera` / Great Expectations : facultatifs. Le reste : inutile ici.

**E. Performances** — Aucun gain de vitesse mesurable à \~1 500 lignes ; les vectorisations sont motivées par la lisibilité.

**F. Alternatives technologiques** — pandas reste le choix pertinent sans réserve.

**G. Les améliorations prioritaires**

1. **`validate=` et anti-jointures documentées** sur les deux `merge()` de l'Étape 6.5 : sécurise le pipeline si les données évoluent. *Étape 6.5.*
2. **Une seule colonne `datetime64` par variable** plutôt que 2 colonnes dont une à date arbitraire : réduit le risque de mésusage. *Étape 6.1.*
3. **Vectoriser et factoriser les 5 conversions booléennes** en une fonction réutilisable : gain de lisibilité, aucun changement de résultat. *Étape 6.3.*

\---

<a id="bilan"></a>

# 🎯 Bilan global

### Ce qui est solide

* Diagnostic des faux doublons SKU (produit / attachment WordPress) ;
* Analyse Pareto 80/20 et palmarès CA / quantités / rotation ;
* Traitement raisonné des valeurs aberrantes (prix négatifs, stock négatif) ;
* Typage cohérent et choix de rester sur pandas, justifié par le volume.

### Les actions à plus forte valeur

|Priorité|Action|Partie|
|-|-|-|
|**P1**|Effectifs + test statistique (ANOVA / Kruskal-Wallis) sur la marge par catégorie|Analyse|
|**P1**|`validate=` + anti-jointures sur les merges de l'Étape 6.5|Pipeline|
|**P2**|Investiguer la marge à -665 % et rappeler le périmètre 714/825|Analyse|
|**P2**|Tracer le graphique Pareto ; ajouter Spearman et MAD|Analyse|
|**P2**|Simplifier le traitement des dates et vectoriser les booléens|Pipeline|

### Ce qui a été volontairement écarté

Polars, DuckDB, PyArrow, Numba, Great Expectations, Isolation Forest : **disproportionnés** pour un projet de \~1 500 lignes, sans gain démontrable.

\---

*Ce document est le compte rendu d'échanges avec une IA utilisée comme relecteur critique. Les recommandations ont vocation à être évaluées et appliquées avec discernement.*

