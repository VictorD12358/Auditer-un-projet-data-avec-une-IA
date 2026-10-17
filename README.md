# Auditer-un-projet-data-avec-une-IA

IA générative · Prompt engineering · Python · Statistiques · Formation

Comparaison de deux façons de solliciter la même IA pour auditer et améliorer mon projet d'analyse de ventes e-commerce, avec tri des recommandations, décisions argumentées et application dans le notebook.

## Contexte

Ce projet a été réalisé dans le cadre de ma formation de Data Analyst chez OpenClassrooms. Il porte sur l'usage de l'IA dans mon travail : plutôt que d'accepter les suggestions d'une IA, j'ai voulu les mettre à l'épreuve sur un cas réel. J'ai donc demandé à Claude d'auditer mon projet 6 (analyse du stock et des ventes d'un site de vins et spiritueux) avec **deux stratégies de prompt**, puis j'ai comparé les résultats avant de décider quoi appliquer.

Attention : il s'agit bien d'une comparaison de **deux méthodes de prompting sur le même modèle**, et non de deux IA différentes.

## Démarche

1. **Deux stratégies de prompt**
   - **Prompt simple (itératif)** : une conversation en plusieurs tours. Un premier prompt court (« joue le rôle d'un data analyst senior »), puis des relances qui ajoutent chaque fois une contrainte (analyse de données, puis pipeline, avec au moins deux options pour chaque point).
   - **Prompt élaboré (structuré)** : une seule instruction détaillée qui fixe d'emblée le rôle, le périmètre et le format de sortie, puis un second prompt du même type pour la partie pipeline

2. **Grille de comparaison des deux sorties** : chaque point relevé est classé comme convergent (les deux IA l'identifient), complémentaire (une seule IA le trouve) ou divergent (solutions différentes pour le même problème)

3. **Décision argumentée pour chaque recommandation** : retenue ou écartée, avec une justification externe (références statistiques et techniques) et une évaluation du rapport effort / gain

4. **Application dans le notebook** des recommandations retenues

## Résultats

- **9 points convergents** entre les deux approches, dont la duplication de la logique de jointure (jugée la redondance la plus significative du notebook), la marge à −665,6 % non investiguée, l'intérêt de Spearman, du MAD et d'un test de comparaison des marges par catégorie
- Le **prompt élaboré** a relevé 6 problèmes factuels supplémentaires, comme un `reset_index()` sans réassignation, une conversion booléenne fragile ou l'absence d'effectifs par catégorie avant de comparer les marges
- Le **prompt simple** a surtout apporté des variantes de solutions sur des problèmes déjà identifiés (corrélation partielle, `lxml`, `sklearn.Pipeline`), et un constat sur la troncature des graphiques à l'export
- Constat : dans mon cas, la qualité de l'audit dépend davantage de la **précision de la structure demandée dans le prompt initial** (rôle, périmètre, format de sortie) que du nombre d'échanges
- **Recommandations appliquées** : Spearman en complément de Pearson (prix / prix d'achat : 0,99 ; prix / stock : −0,57 ; prix / ventes : −0,69), score MAD en complément du Z-score, test de Kruskal-Wallis sur la marge par type de produit (H = 152,8 ; p < 10⁻²⁹), diagramme de Pareto, documentation de l'anomalie de marge
- **Recommandations écartées** (avec justification) : migration vers Polars/DuckDB (gain nul à ce volume), validation de schéma avec pandera ou Great Expectations (disproportionné), barres de progression `tqdm`

## Aller plus loin : analyses complémentaires

- **Prise de recul méthodologique** : j'ai documenté les limites de ma propre comparaison (même modèle, un seul projet) plutôt que de conclure trop vite
- **Contre-vérification des recommandations** : chaque conseil retenu est relié à une source externe, et non adopté parce qu'une IA le propose
- **Décisions « écartées » documentées**, pas seulement les décisions retenues : savoir refuser une optimisation disproportionnée fait partie du travail d'analyste
- **Nouvelle lecture des données grâce à Spearman** : la corrélation de Spearman révèle une relation négative marquée entre le prix et le stock que Pearson ne mettait pas en évidence

## Réalisations

**Score MAD pour détecter les valeurs atypiques (Python)**

```python
mediane_price = df['price'].median()
MAD_price = (df['price'] - mediane_price).abs().median()

df['score_MAD_price'] = (df['price'] - mediane_price).abs() / MAD_price
```

**Comparaison de la marge par type de produit (Python)**

```python
from scipy.stats import kruskal

groupes_marge = [g['Taux_marge'].values
                 for _, g in df_marge.groupby('product_type')]
statistique, p_value = kruskal(*groupes_marge)
```

![Comparaison des deux prompts](images/comparatif_prompts.png)
*Points convergents et complémentaires entre les deux stratégies*

![Matrice de Spearman](images/spearman.png)
*Matrice de corrélation de Spearman : stock, ventes, prix*

![Diagramme de Pareto](images/pareto.png)
*Diagramme de Pareto du chiffre d'affaires*

## Limites

- La comparaison repose sur une seule expérience, sur un seul projet : les conclusions ne se généralisent pas telles quelles
- Le prompt simple dépend du jugement de la personne qui relance. Le résultat reflète aussi la qualité de mes propres questions.
- Le test de Kruskal-Wallis indique qu'il existe des différences de marge, mais pas entre quelles catégories (un test post-hoc serait la suite logique)

## Stack

`Claude` · `Prompt engineering` · `Python` · `pandas` · `scipy` · `plotly` · `Jupyter` · `Notion`

[Voir le projet complet sur GitHub →]()
[Retour à la liste de projets →]()
