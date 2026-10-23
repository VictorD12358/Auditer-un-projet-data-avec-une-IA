# Mise en concurrence de deux stratégies de prompt IA — Audit du Projet 6 BottleNeck

## 1\. Objectif de la démarche

Pour améliorer mon Projet 6 (analyse BottleNeck), j'ai testé deux façons de solliciter la même IA (Claude) sur exactement la même tâche — auditer le notebook et proposer des pistes d'amélioration — pour comparer la qualité, la rigueur et l'exploitabilité des résultats selon la manière de formuler la demande. Cette page documente :

* les deux systèmes de prompt utilisés (texte exact) ;
* ce que chacun a produit ;
* les points où les deux réponses convergent (redondance) ;
* les points où elles divergent ou se complètent ;
* les décisions que j'ai retenues, celles que j'ai écartées, et pourquoi (avec sources externes à l'appui quand c'est pertinent).

## 2\. Les deux systèmes de prompt

### 2.1 — « élaboré »

Une seule instruction très détaillée, écrite en une fois, qui définit d'emblée le rôle, le périmètre et la structure attendue de la réponse, puis un second prompt du même type pour la partie pipeline :

> \*« Tu es un Data Analyst senior, spécialisé en Python, pandas, statistiques, analyse exploratoire, visualisation et optimisation des traitements de données. Je vais te fournir en pièce jointe le PDF de mon Projet 6 Data Analyst OpenClassrooms (...) »\*

> \*« Tu es un Data Analyst senior spécialisé en Python, pandas, data engineering léger, optimisation des pipelines de données et bonnes pratiques de traitement de données. (...) »\*

Caractéristiques : peu d'échanges (1 à 2 messages), mais chaque message porte lui-même toute la spécification du livrable attendu (rôle, angle d'analyse, format de sortie).

### 2.2 — « Prompt simple »

Une conversation en plusieurs tours, où chaque relance ajoute une contrainte précise plutôt que de tout spécifier dès le départ :

1. *« Joue le rôle d'un data-analyst senior et audit mon projet 6 pour OpenClassrooms »*
2. *« Ok maintenant j'ai besoin que tu me fasses un retour spécifique sur la partie analyse de données. Que tu me proposes plus spécifiquement des librairies (...) au moins deux \[options] »*
3. *« Ok maintenant (...) retour spécifique sur la partie pipeline (à partir de l'étape 6) (...) au moins deux \[options] »*

Caractéristiques : prompt initial très court, mais la spécification se construit progressivement au fil des tours (d'où « élaboré » = construit par itérations, et non « détaillé dès le départ »).

**Point méthodologique important à documenter** : les deux prompts ont été envoyés à Claude, donc la comparaison isole surtout l'effet du *style de prompting* (un seul prompt exhaustif vs plusieurs prompts itératifs), pas une comparaison entre deux IA différentes au sens strict. Si l'intitulé de la page est « mise en concurrence de deux IA », il faut le préciser pour être rigoureux : c'est une mise en concurrence de deux *méthodes de prompting sur le même modèle*.

## 3\. Grille de lecture utilisée pour comparer les deux sorties

Pour chaque point relevé par l'une ou l'autre IA, j'ai classé selon :

* **Convergent** : les deux IA relèvent le même point (redondance — gain marginal à documenter les deux fois).
* **Complémentaire** : un point relevé par une seule IA, qui apporte une information réellement nouvelle.
* **Divergent** : les deux IA proposent des solutions différentes pour le même problème.

## 4\. Résultats convergents (les deux IA sont d'accord)

|Sujet|Prompt simple|Prompt élaboré|Décision retenue|
|-|-|-|-|
|Marge à -665,6 % non investiguée (`In \[324]`, produit 4355)|Signalé comme point à commenter avant d'écarter la ligne|Signalé identiquement|**Corrigé** — ajout d'une phrase d'investigation|
|Boucles `iterrows()` au lieu de pandas vectorisé|Signalé, avec verdict clair : gain de lisibilité, pas de vitesse à ce volume|Signalé, avec la même conclusion sur le volume|**Corrigé partiellement** — vectorisation appliquée, mais présent comme un gain de qualité de code, pas de performance (voir §6)|
|Ajouter Spearman en complément de Pearson|Recommandé (priorité P2)|Recommandé (Option A de la section corrélations)|**Retenu** — voir justification §7|
|Ajouter MAD comme alternative robuste au Z-score|Recommandé (priorité P2)|Recommandé (Option A de la section outliers)|**Retenu** — voir justification §7|
|Test statistique pour comparer la marge par catégorie (Kruskal-Wallis / ANOVA)|Recommandé en priorité P1, avec argumentation détaillée sur les effectifs par groupe|Recommandé, avec le même arbitrage (Kruskal-Wallis si petits groupes)|**Retenu** — voir justification §7|
|`pandera` pour la validation de schéma|Évoqué comme option facultative, jugée disproportionnée à ce stade|Proposé comme Option A pour sécuriser les clés de jointure|**Écarté** — les deux IA convergent en fait sur le même niveau de priorité une fois qu'on lit le détail : "prompt simple" est plus explicite en disqualifiant clairement l'outil pour ce volume ("disproportionné pour un notebook de formation à ce stade")|
|Polars / DuckDB / PyArrow pour la performance|Verdict explicite : "quasiment nul à ce volume — non recommandé"|Présenté comme Option B "si le pipeline devient récurrent"|**Écarté** — gain marginal, voir §8|

## 5\. Résultats complémentaires — trouvés uniquement par le « prompt élaboré »

Ce sont des points que le prompt élaboré (mes propres échanges) **n'a pas soulevés**, et qui sont pourtant concrets et vérifiables dans le PDF :

|Point|Détail|Gravité|
|-|-|-|
|`reset\_index()` appelé sans réassignation (`In \[282]`, `In \[298]`)|Le tri est correct mais l'index n'est jamais remis à plat (`df = df.reset\_index(drop=True)` manquant) — incohérent avec l'Étape 5.3 où c'est fait correctement|🟠 Mineur mais à corriger pour homogénéité|
|Conversion booléenne basée sur "non-NaN" plutôt que sur la valeur attendue|`tax\_status`/`post\_mime\_type` etc. deviennent `True` dès qu'ils sont renseignés, sans vérifier que la valeur est bien celle attendue (`'taxable'`, `'image/jpeg'`) — fonctionne aujourd'hui mais fragile si une nouvelle modalité apparaît|🟠 Risque de régression silencieuse|
|Effectifs par catégorie absents avant de comparer les taux de marge|La conclusion "le Cognac a une meilleure marge que l'huile d'olive" n'est jamais appuyée par un `value\_counts()` — certaines catégories peuvent reposer sur très peu de produits|🟠 Conclusion potentiellement non fondée|
|Nuance de causalité sur la corrélation stock/ventes|Le notebook interprète "stock corrélé positivement aux ventes" comme "une bonne chose", sans envisager la causalité inverse (les produits populaires sont réapprovisionnés davantage)|🟡 Nuance d'interprétation|
|Analyse Pareto jamais visualisée|Le calcul du 80/20 est fait mais jamais représenté par un vrai graphique Pareto (barres + courbe cumulée)|🟡 Amélioration de présentation|
|Colonnes "heure seule" avec date arbitraire|`post\_date\_hour` etc. reçoivent une date par défaut (le jour d'exécution du notebook) lors du parsing — dangereux si quelqu'un les réutilise pour un calcul de durée|🟠 Risque de mésusage futur|
|Vérification factuelle de mes propres affirmations|J'avais indiqué dans mon prompt avoir utilisé MAD et Isolation Forest — l'IA a vérifié le PDF et **m'a corrigé** : seules Z-score et IQR sont réellement présentes dans le notebook|🟢 Point de méthode important, voir §9|

## 6\. Résultats complémentaires — trouvés uniquement par le « Prompt simple »

|Point|Détail|
|-|-|
|Troncature **visuelle** des graphiques Plotly à l'export PDF (pas seulement le code)|Vérifié en rastérisant la page contenant le graphique "Top 20 des articles par CA" : seuls 8 des 20 produits restent visibles, les 12 autres sont coupés net par la marge de page. Le "Prompt simple" avait bien identifié la troncature du **code** par impression navigateur, mais pas celle des graphiques eux-mêmes.|
|Corrélation partielle (`pingouin.partial\_corr`) pour isoler l'effet du stock sur la relation prix/ventes|Piste statistique plus poussée, non mentionnée par le "Prompt simple"|
|`lxml` comme alternative plus rapide à `BeautifulSoup` pour le nettoyage HTML|Piste de librairie non mentionnée par le "Prompt simple" (qui juge au contraire que `.apply(BeautifulSoup)` est déjà la bonne solution, sans proposer d'alternative)|
|`sklearn.Pipeline` pour formaliser le pipeline de nettoyage|Piste d'industrialisation non mentionnée par le "Prompt simple"|

**Constat**  : le volume d'informations réellement complémentaires apporté par le "Prompt simple" est plus faible que celui apporté par le "Prompt simple". La plupart de mes apports sont des **variantes de librairies** sur des points déjà identifiés par l'autre prompt (donc redondants sur le *problème*, complémentaires seulement sur la *solution proposée*), alors que le "Prompt simple" a trouvé des **problèmes factuels supplémentaires** (`reset\_index` oublié, fragilité du typage booléen) qui n'avaient tout simplement pas été vus ailleurs.

## 7\. Décisions retenues — justification externe

### 7.1 Spearman en complément de Pearson

Les deux IA le recommandent pour la même raison : la distribution du prix est asymétrique (moyenne 32,33 € > médiane 23,45 €), et Pearson ne mesure qu'une relation linéaire. Une synthèse méthodologique le confirme : Spearman est recommandé notamment quand *"one or both variables are heavily skewed, or the data contain outliers you can't justify removing"*, car il travaille sur les rangs plutôt que sur les valeurs brutes ([CASRAI — Spearman rank correlation](https://casrai.org/guides/spearman-rank-correlation-in-spss)). Une autre source résume la même règle de décision pour une relation monotone mais non linéaire ([socscistatistics.com](https://www.socscistatistics.com/tutorials/spearman-correlation/)).
→ **Décision : ajouté**, coût quasi nul (une ligne de code), gain d'interprétation réel.

### 7.2 MAD (Z-score modifié) en complément du Z-score classique

Justifié par le fait que le Z-score classique est calculé avec la moyenne et l'écart-type, deux statistiques elles-mêmes tirées par les valeurs extrêmes qu'on cherche à détecter — un phénomène documenté comme le "masking effect" ([MetricGate — Z-Score vs Modified Z-Score](https://metricgate.com/blogs/zscore-vs-modified-zscore/)). Le MAD, basé sur la médiane, reste stable même en présence de valeurs extrêmes.
→ **Décision : ajouté** en complément (pas en remplacement) du Z-score existant et de l'IQR, pour croiser les lectures.

### 7.3 Kruskal-Wallis plutôt qu'ANOVA pour comparer la marge par catégorie de produit

Les deux IA convergent sur l'idée qu'un test est nécessaire avant d'affirmer que "le Cognac a une meilleure marge que l'huile d'olive". Le choix entre ANOVA et Kruskal-Wallis dépend directement des effectifs par catégorie (actuellement inconnus dans le notebook) et de la normalité des résidus. Une synthèse de référence est claire sur ce point : Kruskal-Wallis est préférable quand l'échantillon est petit avec des valeurs extrêmes, car il est basé sur les rangs et donc peu sensible aux valeurs aberrantes ([CASRAI — Kruskal-Wallis](https://www.casrai.org/guides/kruskal-wallis-test-when-to-use-and-how-to-report)) ; à l'inverse, si les effectifs sont suffisants et la distribution proche de la normale, l'ANOVA reste plus puissante ([socscistatistics.com — Kruskal-Wallis](https://www.socscistatistics.com/tutorials/kruskal-wallis/)).
→ **Décision : d'abord vérifier les effectifs par `product\_type` (`value\_counts()`) puis choisir. Etant donné le faible nombre de catégories (7) et le risque de petits groupes (Gin, Cognac), Kruskal-Wallis est le choix par défaut retenu**, avec un test de normalité (`shapiro`) pour confirmer si besoin de basculer vers l'ANOVA.

## 8\. Décisions écartées — gain marginal (avec justification)

### 8.1 Polars / DuckDB / PyArrow

Les deux IA arrivent à la même conclusion mais avec des arguments légèrement différents : le "Prompt simple" chiffre explicitement "quasiment nul à ce volume", le "Prompt élaboré" nuance en évoquant un intérêt uniquement si le pipeline devient récurrent. Une source technique de référence confirme que pandas reste l'outil pertinent pour les datasets petits à moyens et l'analyse exploratoire, Polars étant pensé pour la performance sur de gros volumes ([Databricks — Polars vs Pandas](https://www.databricks.com/blog/polars-vs-pandas)).
→ **Décision : aucun changement.** Sur 714-1513 lignes, migrer vers Polars n'apporterait aucun gain mesurable et représenterait un coût de réécriture significatif pour zéro bénéfice démontrable.

### 8.2 `pandera` / Great Expectations pour la validation de schéma

Les deux IA le mentionnent, mais le "Prompt simple" est plus tranché : disproportionné pour un notebook de formation à ce stade, alors que quelques `assert` ciblés (unicité de clé, absence de `NaT` après conversion de date) suffisent.
→ **Décision : aucun changement — remplacé par des `assert` simples** sur les points de risque réels identifiés (unicité de `product\_id`, cardinalité des jointures).

### 8.3 `tqdm` sur les boucles restantes

Proposé puis explicitement écarté par le "Prompt simple" lui-même : les boucles s'exécutent en quelques millisecondes sur ce volume, une barre de progression n'a aucune utilité.
→ **Décision : aucun changement.**

## 9\. Point de méthode à mettre en avant dans la page Notion

Un élément à souligner tel quel dans la page finale : dans l'échange "Prompt simple", j'avais affirmé avoir utilisé MAD et Isolation Forest dans mon notebook. L'IA n'a pas pris cette affirmation pour argent comptant — elle a rouvert le PDF, cherché ces termes, et m'a explicitement corrigé en constatant que seules les méthodes Z-score et IQR étaient réellement présentes. C'est un exemple concret et vérifiable de vérification factuelle plutôt que de validation complaisante d'une affirmation utilisateur, à citer comme preuve de la fiabilité de la démarche d'audit.

## 10\. Bilan — quelle stratégie de prompt a le mieux servi le projet ?

*(section à compléter — pistes de rédaction ci-dessous, à valider par toi avant publication)*

* Le **"Prompt simple"** (une instruction unique très structurée, précisant le rôle et le livrable attendu dès le départ) a produit une couverture plus large et plus systématique : il a trouvé des problèmes factuels que l'approche itérative n'a pas soulevés (export cassé, `reset\_index` oublié, fragilité du typage booléen), et il a intégré de lui-même des tableaux de priorisation, des verdicts effort/gain et une vérification factuelle de mes propres affirmations.
* Le **"Prompt élaboré"** (conversation construite par relances successives) a produit des réponses plus ciblées sur ce que je demandais explicitement à chaque tour (systématiquement deux options par point, avec code), mais a de fait couvert un périmètre plus étroit — parce que je devais moi-même penser à poser chaque question, alors que le "Prompt simple" avait anticipé une grille de lecture complète dès le premier message.
* Conclusion à nuancer : la qualité du résultat semble ici plus corrélée à la **précision de la structure demandée dans le prompt initial** (rôle + périmètre + format de sortie attendu) qu'au nombre d'échanges. Un prompt long mais qui spécifie clairement le livrable attendu a mieux "cadré" l'exhaustivité de l'audit qu'une conversation où je pilotais moi-même, tour par tour, ce qui devait être creusé.

