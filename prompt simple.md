# Projet 6 OpenClassrooms — Échanges avec une IA (prompt simple)

## Sommaire

1. [Échange 1 — Analyse de données : librairies et tests statistiques](#échange-1--analyse-de-données--librairies-et-tests-statistiques)
2. [Échange 2 — Pipeline (à partir de l'étape 6) : librairies et vitesse d'exécution](#échange-2--pipeline-à-partir-de-létape-6--librairies-et-vitesse-dexécution)

\---

## Échange 1 — Analyse de données : librairies et tests statistiques

### 📝 Prompt

> Joue le rôle d'un data-analyst senior et audit mon projet 6 pour OpenClassrooms.

### 🤖 Réponse

#### Retour technique — analyse de données du notebook **BottleNeck**

Je reprends chaque bloc analytique du notebook et je propose, pour chacun, au moins deux options concrètes (librairie ou test) avec leurs compromis, pour que tu puisses trancher selon ton contexte.

\---

##### 1\. Vectorisation / vitesse d'exécution (remplacer les `iterrows()`)

C'est le gain le plus immédiat vu que quasi tout le contrôle qualité tourne en boucle Python pure.

**Option A — Pandas vectorisé natif (le strict minimum, zéro dépendance)**

```python
# Au lieu de la boucle sur les doublons id\_web
doublons = liaison\['id\_web'].dropna().duplicated().sum()

# Au lieu de la boucle sur les sku manquants
web\[web\['sku'].isna()]

# Au lieu de la boucle in\_stock / stock\_quantity
incoherent = erp\_nettoye\[(erp\_nettoye\['in\_stock']) \& (erp\_nettoye\['stock\_quantity'] < 1)]
```

Gain immédiat de 10 à 100x sur un dataset de cette taille (825-1513 lignes), et surtout ça évite le formatage "une ligne imprimée par anomalie" qui a gonflé ton notebook à 218 pages.

**Option B — `polars` si tu veux industrialiser au-delà de ce projet**

Polars (API proche de pandas mais moteur Rust multi-thread) devient intéressant si tu réutilises ce genre de pipeline sur des volumes plus gros ou si tu veux des temps de calcul reproductibles pour un usage récurrent (ex. un rapport mensuel automatisé). Sur 825 lignes le gain est anecdotique, mais si BottleNeck devient un vrai produit de reporting, `polars.LazyFrame` avec exécution différée (`.collect()`) est un choix plus pérenne que pandas.

→ Pour ce projet précis : Option A suffit largement. Polars se justifie seulement si tu vois ce notebook comme le prototype d'un pipeline récurrent.

\---

##### 2\. Détection d'`outliers` (au-delà de Z-score / IQR)

Le Z-score classique, fondé sur la moyenne et l’écart-type, peut être peu adapté à une distribution de prix fortement asymétrique, car ces deux statistiques sont sensibles aux valeurs extrêmes.



En revanche, l’IQR repose sur les quartiles et ne suppose pas une distribution symétrique. Elle constitue donc déjà une méthode relativement robuste. Toutefois, ses seuils conventionnels peuvent être moins pertinents lorsque la distribution présente une forte asymétrie.



Une alternative consiste à utiliser le MAD (Median Absolute Deviation) pour calculer un score robuste, fondé sur la médiane et les écarts absolus à celle-ci. Cette méthode permet de compléter l’IQR, mais son intérêt doit être évalué en fonction de la forme de la distribution et des objectifs de détection.

**Option A — MAD (Median Absolute `Deviation`) / Z-score robuste**

```python
from scipy.stats import median\_abs\_deviation

median = df\['price'].median()
mad = median\_abs\_deviation(df\['price'])
df\['robust\_z'] = 0.6745 \* (df\['price'] - median) / mad
outliers = df\[df\['robust\_z'].abs() > 3.5]
```

Utilise la médiane et le MAD au lieu de la moyenne/écart-type — beaucoup moins sensible aux valeurs extrêmes elles-mêmes. C'est la correction naturelle et minimale à apporter à ta méthode Z-score actuelle.



**Option B — Isolation Forest (`scikit-learn`) pour une détection multivariée**

```python
from sklearn.ensemble import IsolationForest

iso = IsolationForest(contamination=0.05, random\_state=42)
df\['outlier'] = iso.fit\_predict(df\[\['price', 'stock\_quantity', 'total\_sales', 'purchase\_price']])
```

Ton audit actuel traite chaque variable séparément (prix seul, marge seule). Isolation Forest détecte les anomalies **multivariées** — un produit avec un prix normal mais une combinaison stock/ventes/marge incohérente (comme ta ligne à -665 % de marge) ressortirait naturellement ici sans qu'il faille la trouver "par hasard" en triant sur `Taux\_marge`.

→ Option A pour rester simple et interprétable (utile en soutenance) ; Option B si tu veux industrialiser la détection d'anomalies sur l'ensemble du catalogue plutôt que colonne par colonne.

\---

##### 3\. Corrélations : aller au-delà de Pearson



La heatmap actuelle utilise le coefficient de Pearson pour explorer les relations linéaires entre stock\_quantity, total\_sales et price. Ce choix est pertinent pour une première analyse exploratoire : Pearson permet de mesurer la force et le sens des associations linéaires sans exiger que les variables suivent individuellement une distribution normale.



Il convient néanmoins de garder à l'esprit que les valeurs extrêmes peuvent influencer les coefficients et que des relations non linéaires peuvent être mal représentées par cette mesure. Des nuages de points permettent de vérifier visuellement la forme des relations les plus intéressantes.



Option A — Spearman en complément de Pearson



corr\_spearman = df\[

&#x20;   \['stock\_quantity', 'total\_sales', 'price']

].corr(method='spearman')



Spearman permet d'évaluer les relations monotones, y compris lorsqu'elles ne sont pas linéaires. Comparer les deux matrices peut aider à repérer des associations dont la nature mérite d'être approfondie. Il ne s'agit pas nécessairement de remplacer Pearson, mais de compléter l'analyse lorsque cela apporte une information supplémentaire.



Option B — Corrélation partielle pour approfondir la relation prix-ventes



import pingouin as pg



result = pg.partial\_corr(

&#x20;   data=df,

&#x20;   x='price',

&#x20;   y='total\_sales',

&#x20;   covar='stock\_quantity'

)



Cette analyse permet d'étudier l'association entre le prix et les ventes après ajustement statistique sur la quantité en stock. Elle peut notamment aider à déterminer si l'association observée reste présente lorsque cette variable est prise en compte.



Les résultats doivent toutefois être interprétés avec prudence : une corrélation partielle ne démontre pas une relation causale et ne tient compte que des variables incluses dans le modèle.



En pratique : conserver Pearson pour la heatmap initiale, examiner les relations les plus marquées à l'aide de nuages de points et utiliser Spearman ou la corrélation partielle lorsque la question étudiée le justifie.
pg.partial\_corr(data=df, x='price', y='total\_sales', covar='stock\_quantity')
```

\---

##### 4\. Comparaison du taux de marge par famille de produit (au-delà de la moyenne groupée)

Tu affiches la moyenne du taux de marge par `product\_type` (Cognac 42,8 %, Huile d'olive 21,9 %, etc.) mais sans tester si ces écarts sont statistiquement significatifs ou s'ils pourraient être dus au hasard sur des petits échantillons par catégorie.

**Option A — ANOVA one-`way` (`scipy.stats.f\_oneway`) si les taux de marge sont \~normaux par groupe**

```python
from scipy.stats import f\_oneway

groups = \[pos\_marge.loc\[pos\_marge\['product\_type']==t, 'Taux\_marge'] for t in pos\_marge\['product\_type'].unique()]
f\_stat, p\_value = f\_oneway(\*groups)
```

Répond à "les écarts de marge entre familles sont-ils significatifs ou juste du bruit d'échantillonnage ?"

**Option B — `Kruskal`-Wallis (non-paramétrique), plus prudent ici**

```python
from scipy.stats import kruskal

kruskal(\*groups)
```

Vu que certaines catégories (Gin, Cognac) ont probablement très peu de produits, l'hypothèse de normalité de l'ANOVA est fragile. Kruskal-Wallis teste la même question sur les rangs, sans hypothèse de distribution — plus robuste avec des petits groupes déséquilibrés.

→ Je recommande de commencer par Kruskal-Wallis vu la taille probable de certains groupes, et de ne passer à l'ANOVA que si un test de normalité (`scipy.stats.shapiro` par groupe) le valide.

\---

##### 5\. Tests de normalité (pour justifier le choix entre les options ci-dessus)

**Option A — Shapiro-Wilk (`scipy.stats.shapiro`)**

Le plus courant, fiable jusqu'à \~5000 observations (largement ton cas avec 714 lignes).

**Option B — Test de D'Agostino-Pearson (`scipy.stats.normaltest`)**

Basé sur skewness + kurtosis combinés — plus informatif que Shapiro quand tu veux comprendre *pourquoi* une distribution n'est pas normale (asymétrie vs aplatissement), ce qui est directement utile puisque tu as déjà identifié une asymétrie sur `price`.

→ Shapiro pour trancher vite ; D'Agostino si tu veux documenter la nature de la non-normalité dans ton rapport.

\---

##### 6\. Nettoyage du texte HTML (`post\_excerpt`)

Ton usage de BeautifulSoup est correct mais génère un warning (`MarkupResemblesLocatorWarning`) et reste assez artisanal.

**Option A — `BeautifulSoup` avec `parser` explicite (fix minimal)**

```python
BeautifulSoup(x, "html.parser").get\_text(separator=" ", strip=True)
```

Régler le warning (le `html.parser` est déjà correct, le warning vient du fait que Beautiful Soup soupçonne parfois un nom de fichier) et ajouter `separator=" "` pour éviter que des balises `<div>` collées se retrouvent concaténées sans espace.

**Option B — `lxml` (via `lxml.html`) pour la vitesse**

```python
from lxml import html as lxml\_html

lxml\_html.fromstring(x).text\_content()
```

Sensiblement plus rapide que `html.parser` sur de gros volumes (utile si tu réutilises ce pipeline sur un catalogue plus large que 1513 lignes) et généralement plus tolérant aux HTML mal formés comme ceux que tu as (balises `<div>` imbriquées, `<footer>` orphelins).

→ Sur 1513 lignes la différence de vitesse est invisible ; Option A suffit. Bascule vers Option B seulement si le volume grossit.

\---

##### 7\. Export des graphiques (fixer le problème de troncature PDF)

**Option A — `kaleido` + `fig.write\_image()` (rester dans `Plotly`)**

```python
fig.write\_image("top20\_ca.png", width=1400, height=700, scale=2)
```

Reste dans ton stack actuel (Plotly), corrige directement le problème de troncature identifié dans l'audit précédent, sans changer de librairie.

**Option B — Basculer ces graphiques sur `matplotlib`/`seaborn` (déjà importés dans ton notebook)**

Pour des barres statiques destinées à un export PDF/print, `seaborn.barplot` avec `plt.savefig(dpi=200, bbox\_inches='tight')` est plus prévisible à l'impression que Plotly (conçu avant tout pour l'interactif web) et évite une dépendance supplémentaire (kaleido) puisque matplotlib/seaborn sont déjà utilisés ailleurs dans ton notebook (`histplot`).

→ Option A si tu veux garder l'interactivité en usage Jupyter et juste corriger l'export ; Option B si les graphiques n'ont vocation qu'à finir dans un PDF/PPT, auquel cas autant harmoniser tout le notebook sur une seule librairie de viz (la tienne penche déjà vers seaborn pour les distributions).

\---

Si tu veux, je peux directement réécrire une section du notebook (par exemple la détection d'outliers avec MAD + Isolation Forest, ou la heatmap de corrélation avec Spearman et p-values via pingouin) pour te donner un exemple exécutable sur tes propres données.

\---

## Échange 2 — Pipeline (à partir de l'étape 6) : librairies et vitesse d'exécution

### 📝 Prompt

> Ok maintenant j'ai besoin que tu me fasses un retour spécifique sur la partie pipeline (à partir de l'étape 6) . Que tu me proposes plus spécifiquement des librairies qui amélioreraient les analyses ou leur vitesse d'éxectution. Et surtout que tu proposes plusieurs options pertinentes à chaque fois (au moins deux).

### 🤖 Réponse

#### Retour technique — pipeline de l'Étape 6 (dates, textes, booléens, clés, `remerge`)

Je reprends chaque sous-étape avec, à chaque fois, au moins deux options de librairies ou d'approches pour gagner en vitesse et en robustesse.

\---

##### 6.1 — Dates

Ton approche actuelle fait un `str.split(" ", n=1)` pour séparer date/heure, crée 2 colonnes temporaires, supprime la colonne source, puis appelle `pd.to\_datetime()` séparément sur les 8 colonnes résultantes (`post\_date`, `post\_date\_gmt`, `post\_modified`, `post\_modified\_gmt` × date/heure). C'est redondant : tu parses en fait la même information deux fois (le split textuel, puis le parsing datetime).

**Option A — `Parser` une seule fois, puis dériver avec l'accesseur `.dt` (zéro dépendance)**

```python
web\_nettoye\['post\_date\_full'] = pd.to\_datetime(web\_nettoye\['post\_date'], format='%Y-%m-%d %H:%M:%S')
web\_nettoye\['post\_date2'] = web\_nettoye\['post\_date\_full'].dt.normalize()   # date seule, reste datetime64
web\_nettoye\['post\_date\_hour'] = web\_nettoye\['post\_date\_full'].dt.time     # heure seule
```

Un seul parsing par colonne source au lieu de deux opérations texte + deux `to\_datetime`. Ajouter `format='%Y-%m-%d %H:%M:%S'` (au lieu de laisser pandas inférer) accélère aussi le parsing de façon mesurable sur ce volume — l'inférence automatique de format est la partie la plus lente de `to\_datetime`.

**Option B — Backend `pyarrow` pour pandas (si tu montes en volume)**

```python
web\_nettoye\['post\_date\_full'] = pd.to\_datetime(web\_nettoye\['post\_date'], format='%Y-%m-%d %H:%M:%S').astype('datetime64\[ns, UTC]')
# ou dès la lecture :
web = pd.read\_excel(..., dtype\_backend='pyarrow')
```

Le moteur pyarrow (disponible depuis pandas 2.0) accélère significativement le parsing de dates et les opérations vectorisées dessus. Sur 1513 lignes le gain est imperceptible, mais si ce pipeline est amené à tourner sur des exports plus volumineux (extraction complète du catalogue, historisation), c'est le bon réflexe à prendre maintenant plutôt que de refactorer plus tard.

→ Option A suffit largement ici ; Option B seulement si tu vois ce notebook comme prototype d'un pipeline récurrent sur plus gros volume.

\---

##### 6.2 — Textes

`astype("string")` est correct pour `post\_title`, `post\_excerpt`, `post\_name`. Mais `product\_type` n'a que **7 valeurs distinctes** (Vin, Whisky, Cognac, Gin, Champagne, Huile d'olive, Autre) — le typer en string libre est sous-optimal.

**Option A — `category` `dtype` (pandas natif) pour les colonnes à faible cardinalité**

```python
web\_nettoye\['product\_type'] = web\_nettoye\['product\_type'].astype('category')
```

Pour une colonne à 7 modalités sur 1513 lignes, ça réduit l'empreinte mémoire et surtout **accélère tous tes `groupby('product\_type')`** utilisés à l'étape 5.4 (taux de marge par famille) — un groupby sur `category` est nettement plus rapide qu'un groupby sur string/object car pandas n'a plus besoin de comparer les chaînes, juste les codes entiers sous-jacents.

**Option B — `string\[pyarrow]` pour les colonnes à cardinalité élevée (`post\_title`, `post\_excerpt`)**

```python
web\_nettoye\['post\_title'] = web\_nettoye\['post\_title'].astype('string\[pyarrow]')
```

Pour les colonnes de texte libre (titres, descriptions), le backend pyarrow accélère les opérations `.str.\*` (tu en fais déjà avec `.str.split`) par rapport au `string\[python]` classique.

→ Utilise `category` pour `product\_type`/`tax\_status` (faible cardinalité, sert aux groupby) et `string\[pyarrow]` pour `post\_title`/`post\_excerpt` (cardinalité élevée, texte libre) — ce ne sont pas deux options concurrentes mais complémentaires selon la colonne.

\---

##### 6.3 — Booléens : le point le plus important de cette étape

Tu répètes **cinq fois** (à l'identique, colonne par colonne : `closed\_ping`, `tax\_status`, `posted`, `closed\_comment`, `image\_jpeg`) exactement le même pattern :

```python
for i, row in web\_nettoye.iterrows():
    if pd.isna(row\[col]):
        web\_nettoye.loc\[i, col] = False
    else:
        web\_nettoye.loc\[i, col] = True
```

C'est la même limite déjà relevée sur le reste du notebook (boucle `iterrows()` là où une opération vectorisée suffit), mais ici c'est particulièrement coûteux car **répété 5 fois** sur des colonnes différentes plutôt que factorisé.

**Option A — Vectorisation directe (`.notna()`), zéro dépendance**

```python
bool\_cols = \['closed\_ping', 'tax\_status', 'posted', 'closed\_comment', 'image\_jpeg']
for col in bool\_cols:
    web\_nettoye\[col] = web\_nettoye\[col].notna()
```

Une seule ligne remplace chacune des 5 boucles (le `for` ici boucle sur une liste de 5 noms de colonnes, pas sur 1513 lignes — c'est la distinction clé). Gain de vitesse énorme et surtout code 5x plus court et lisible.

**Option B — `numpy.where` si tu veux garder une logique conditionnelle plus riche que "NaN → False"**

```python
import numpy as np

web\_nettoye\[col] = np.where(web\_nettoye\[col].isna(), False, True)
```

Fonctionnellement identique à `.notna()` ici, mais utile si demain la règle de conversion devient plus complexe (ex. "False si NaN, True si valeur non nulle, mais garder NaN si colonne X vaut Y") — `np.where` s'étend plus facilement à des règles à 3 branches ou plus, ce que `.notna()` ne permet pas.

→ Option A pour ce cas précis (règle binaire simple), Option B seulement si la règle de conversion se complexifie.

\---

##### 6.4 — Clés

`liaison\['id\_web'] = liaison\['id\_web'].astype("string")` est fait ligne par ligne, colonne par colonne, **au moment du merge** — c'est-à-dire qu'on découvre le mismatch de type entre erp/liaison/web seulement après coup, alors que c'est exactement le genre d'erreur silencieuse qui peut casser un merge (une jointure int contre string échoue silencieusement en donnant 0 correspondance, sans erreur).

**Option A — `pandera` pour valider le schéma dès le chargement**

```python
import pandera as pa

schema = pa.DataFrameSchema({
    "product\_id": pa.Column(int, unique=True),
    "id\_web": pa.Column(str, nullable=True),
})
liaison = schema.validate(liaison)
```

Force un contrat de type explicite sur chaque fichier source **avant** toute jointure, avec des messages d'erreur clairs si un fichier change de format entre deux exécutions (ex. si le prochain export `liaison.xlsx` a `product\_id` en texte). Précieux si ce pipeline doit être relancé plusieurs fois avec des exports différents.

**Option B — Forcer les `dtype` dès `pd.read\_excel()` / `pd.read\_csv()`**

```python
liaison = pd.read\_excel("liaison.xlsx", dtype={"id\_web": "string", "product\_id": "int64"})
```

Plus léger que pandera (pas de nouvelle dépendance) : tu déclares le typage voulu dès la lecture du fichier, donc les incohérences de type entre fichiers sont visibles dès l'étape 2 (exploration) plutôt que découvertes à l'étape 6.4, après avoir déjà fait tourner toutes les analyses de l'étape 5 dessus.

→ Option B est le minimum à faire immédiatement (gratuit, pas de dépendance) ; Option A si ce notebook doit devenir un pipeline réexécuté régulièrement sur des nouveaux exports.

\---

##### 6.5 — `Remerge` : duplication de code à corriger

C'est le point structurel le plus important de cette étape. `erp\_nettoye\_liaison2` et `erp\_nettoye\_web\_nettoye\_liaison2` répètent **exactement** la même logique de merge que `erp\_nettoye\_liaison` et `erp\_nettoye\_web\_nettoye\_liaison` à l'étape 3, avec juste les données nettoyées en entrée. Si demain tu changes une règle de nettoyage, tu dois modifier le merge à deux endroits différents dans le notebook — source classique de bugs silencieux (l'un des deux se retrouve désynchronisé).

**Option A — Fonction Python simple + `.pipe()` pandas (zéro dépendance)**

```python
def construire\_table\_finale(erp, liaison, web):
    return (erp
            .merge(liaison, how='inner', on='product\_id')
            .merge(web, how='inner', on='id\_web'))

table\_finale = construire\_table\_finale(erp\_nettoye, liaison, web\_nettoye)
```

Une seule définition du merge, appelée une seule fois une fois que `erp\_nettoye`/`liaison`/`web\_nettoye` sont complètement nettoyés. Ça élimine la duplication entre étape 3 et étape 6.5, et ça répond aussi au problème méthodologique signalé précédemment (analyses lancées sur des données pas encore nettoyées) : avec cette structure, il n'existe qu'une seule table finale, obtenue après nettoyage complet.

**Option B — `sklearn.pipeline.Pipeline` / `FunctionTransformer` si tu veux formaliser un vrai pipeline réutilisable**

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import FunctionTransformer

pipeline\_nettoyage = Pipeline(\[
    ('nettoyage\_prix', FunctionTransformer(nettoyer\_prix)),
    ('nettoyage\_stock', FunctionTransformer(nettoyer\_stock)),
    ('typage\_bool', FunctionTransformer(convertir\_booleens)),
])
erp\_propre = pipeline\_nettoyage.fit\_transform(erp)
```

Plus lourd à mettre en place, mais donne une structure standard, testable étape par étape, et réutilisable si ce nettoyage doit tourner sur un nouvel export mensuel de erp/web/liaison. C'est overkill pour un notebook d'exercice ponctuel, mais c'est la bonne direction si BottleNeck te demande un jour un rapport récurrent plutôt qu'une analyse ponctuelle.

→ Option A est largement suffisante pour ce projet et résout le problème concret (duplication + ordre nettoyage/analyse) ; Option B seulement si l'objectif final est un pipeline productionnisé et réexécutable.

\---

Le fil conducteur de cette étape 6 : presque tous les problèmes viennent du fait que le nettoyage a été fait *après coup*, colonne par colonne, avec des boucles ligne par ligne redondantes, plutôt que d'être conçu comme une fonction unique appliquée une fois en amont de toute analyse. Si tu veux, je peux te réécrire l'étape 6 dans son ensemble sous cette forme fonctionnelle (Option A du point 6.5), directement exécutable sur tes fichiers.

