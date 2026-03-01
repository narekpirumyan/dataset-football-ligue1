# Ligue-Wide View — Rivalités (head-to-head)

**Category:** Confrontations directes entre clubs (rivalités, derbys, classiques).

---

## Logique corrigée (une rivalité = tous les matchs entre 2 clubs)

**Problème :** Avec l’ancienne logique, la table **RivalryPair** avait une ligne par couple (Domicile, Extérieur) issu de DimMatch. Donc « Paris SG – Marseille » et « Marseille – Paris SG » étaient deux lignes distinctes, et les mesures ne filtraient que sur un sens (ex. seulement les matchs où le premier club était à domicile). Les totaux de buts, affluence, sanctions et points ne portaient que sur la moitié des confrontations.

**Correction :**  
- **RivalryPair** : une seule ligne par **paire non orientée** (colonnes **ClubKey1**, **ClubKey2** avec ClubKey1 &lt; ClubKey2), et **PairLabel** = "Club1 – Club2".  
- **Toutes les mesures** (Total Goals, Sanctions, Avg Attendance, Avg Fill Rate, Points Club1/Club2) filtrent les matchs avec **(Home = C1 AND Away = C2) OR (Home = C2 AND Away = C1)** pour inclure les deux sens.  

En Power BI : recréer la table calculée **RivalryPair** avec la définition ci‑dessous (ClubKey1, ClubKey2 au lieu de HomeClubKey, AwayClubKey), puis mettre à jour les mesures qui l’utilisent.

---

**Principe :** Une rivalité = une **paire non orientée** de clubs (tous les matchs entre ces deux clubs, quel que soit le sens Domicile/Extérieur). Visuels 1 et 4 : slicer « 2 clubs » sur DimClub. Visuels 2, 3 et 5 : table calculée **RivalryPair** avec **une seule ligne par paire** (ClubKey1 ≤ ClubKey2), et toutes les mesures doivent inclure **les deux sens** (Home=A Away=B **ou** Home=B Away=A).

---

## 1. Résumé tête-à-tête (W–D–L, buts)

**Type :** KPI cards + donut ou barres empilées.

**But :** Pour **deux clubs choisis** (slicer ou deux slicers), afficher le bilan des confrontations directes : victoires de chaque camp, nuls, et buts marqués par chaque camp.

**Prérequis :** Un **slicer** sur `DimClub[ClubName]` avec **sélection multiple** ; l’utilisateur choisit exactement **2 clubs**. Les mesures ci‑dessous utilisent `MIN`/`MAX` sur les ClubKey sélectionnés pour distinguer « Club 1 » et « Club 2 » (ordre arbitraire).

**Tables :** DimMatch (HomeClubKey, AwayClubKey, MatchKey), FactClubMatch (ClubKey, MatchKey, Result, Points, GoalsFor, GoalsAgainst), DimClub.

**Mesures DAX :**

```dax
// Nombre de matchs entre les 2 clubs sélectionnés (pour vérifier le contexte)
Rivalry Match Count =
VAR ClubKeys = VALUES(DimClub[ClubKey])
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(
        COUNTROWS(ClubKeys) <> 2,
        BLANK(),
        CALCULATE(
            COUNTROWS(DimMatch),
            FILTER(
                DimMatch,
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1)
            )
        )
    )

// Victoires du club 1 (ClubKey le plus petit parmi les 2 sélectionnés)
Rivalry Wins Club1 =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        CALCULATE(
            COUNTROWS(FactClubMatch),
            FactClubMatch[Result] = "W",
            FactClubMatch[ClubKey] = C1,
            FILTER(DimMatch,
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
        ))

// Victoires du club 2
Rivalry Wins Club2 =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        CALCULATE(
            COUNTROWS(FactClubMatch),
            FactClubMatch[Result] = "W",
            FactClubMatch[ClubKey] = C2,
            FILTER(DimMatch,
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
        ))

// Nuls (matchs où les deux clubs ont fait 1 point)
Rivalry Draws =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        CALCULATE(
            COUNTROWS(FactClubMatch),
            FactClubMatch[Result] = "D",
            FILTER(DimMatch,
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
        ))  // chaque match = 2 lignes FactClubMatch (D,D), donc diviser par 2 si besoin ; ou compter les matchs avec SUM(Points)=2 pour la paire

// Variante nuls : compter les matchs (une fois) où résultat = D pour l’un des deux clubs
Rivalry Draws =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
VAR RivalryMatches = FILTER(DimMatch,
    (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
        || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        COUNTROWS(FILTER(RivalryMatches,
            CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[ClubKey] = C1) = 1
                && CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[ClubKey] = C2) = 1
        )))
```

**Visuel HTML Content (cartes W–D–L + total buts en une seule visualisation)**  
Une seule mesure renvoie le HTML affichant **5 cartes** : Matchs, Victoires Club 1, Victoires Club 2, Nuls, **Total buts**. À utiliser dans un visuel **HTML Content**. Les cartes utilisent une **grille CSS** (`grid-template-columns: repeat(5, 1fr)`) et `width: 100%` pour **s’adapter à la taille du visuel**. Trois couleurs distinctes : **Matchs** = gris (#E8E8E8 / #9e9e9e), **Victoires Club 1, Victoires Club 2, Nuls** = bleu (#E3F2FD / #0066CC), **Total buts** = vert (#E8F5E9 / #1AAB40). Pour éviter l’affichage « - » ou vide lorsqu’une valeur est 0, chaque valeur est gérée avec `COALESCE(..., 0)` et un affichage explicite de `"0"`. Si moins de 2 ou plus de 2 clubs sont sélectionnés, un message invite à sélectionner exactement 2 clubs.

```dax
Rivalry Head-to-Head Cards HTML =
VAR vMatchCount = COALESCE([Rivalry Match Count], 0)
VAR vWins1 = COALESCE([Rivalry Wins Club1], 0)
VAR vWins2 = COALESCE([Rivalry Wins Club2], 0)
VAR vDraws = COALESCE([Rivalry Draws], 0)
VAR vGoals1 = COALESCE([Rivalry Goals Club1], 0)
VAR vGoals2 = COALESCE([Rivalry Goals Club2], 0)
VAR vTotalGoals = vGoals1 + vGoals2
VAR vMc = IF(vMatchCount = 0, "0", FORMAT(vMatchCount, "0"))
VAR vW1 = IF(vWins1 = 0, "0", FORMAT(vWins1, "0"))
VAR vW2 = IF(vWins2 = 0, "0", FORMAT(vWins2, "0"))
VAR vDr = IF(vDraws = 0, "0", FORMAT(vDraws, "0"))
VAR vTg = IF(vTotalGoals = 0, "0", FORMAT(vTotalGoals, "0"))
RETURN
    IF(
        COUNTROWS(VALUES(DimClub[ClubKey])) <> 2,
        "<div style='font-family:Segoe UI;padding:12px;color:#808080;'>Sélectionnez exactement 2 clubs</div>",
        "<style>.rival-cards{display:grid;grid-template-columns:repeat(5,1fr);...}.rival-card-matchs{...}.rival-card-wdl{...}.rival-card-goals{...}</style>"
        & "<div class='rival-cards'>"
        & "<div class='rival-card rival-card-matchs'>...Matchs...</div>"
        & "<div class='rival-card rival-card-wdl'>...Victoires Club 1...</div>"
        & "<div class='rival-card rival-card-wdl'>...Victoires Club 2...</div>"
        & "<div class='rival-card rival-card-wdl'>...Nuls...</div>"
        & "<div class='rival-card rival-card-goals'>...Total buts...</div>"
        & "</div>"
    )
```

**Power BI — visuel HTML Content :** Ajouter un visuel **HTML Content**, placer la mesure **Rivalry Head-to-Head Cards HTML** (table **RivalryPair**) dans **Valeurs**. Lier le slicer « 2 clubs » au visuel. Les 5 cartes s’affichent côte à côte. Toute valeur nulle (0 match, 0 victoire, 0 nul, 0 but) s’affiche explicitement comme **0**, et non comme « - » ou vide.

---

Pour éviter la complexité des nuls, on peut compter simplement les matchs où **points du club 1 = 1** dans le contexte du match (une ligne FactClubMatch par club par match) :

```dax
Rivalry Draws =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        CALCULATE(
            COUNTROWS(DimMatch),
            FILTER(DimMatch,
                ((DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
                && CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[ClubKey] = C1) = 1
            )
        ))
```

**Buts marqués par le club 1 (dans les matchs de la rivalité) :**

```dax
Rivalry Goals Club1 =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        CALCULATE(
            SUM(FactClubMatch[GoalsFor]),
            FactClubMatch[ClubKey] = C1,
            FILTER(DimMatch,
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
        ))

Rivalry Goals Club2 =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        CALCULATE(
            SUM(FactClubMatch[GoalsFor]),
            FactClubMatch[ClubKey] = C2,
            FILTER(DimMatch,
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
        ))
```

**Points totaux (pour barres côte à côte) :**

```dax
Rivalry Points Club1 =
VAR C1 = MIN(DimClub[ClubKey])
VAR C2 = MAX(DimClub[ClubKey])
RETURN
    IF(COUNTROWS(VALUES(DimClub[ClubKey])) <> 2, BLANK(),
        CALCULATE(
            SUM(FactClubMatch[Points]),
            FactClubMatch[ClubKey] = C1,
            FILTER(DimMatch,
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
        ))
// Rivalry Points Club2 : idem en C2
```

**Power BI — étapes :**

**1. Mise en place du slicer (sélection multiple)**

- **Ajouter le slicer :** Dans le ruban **Visualisations**, cliquer sur l’icône **Slicer**. Une zone vide apparaît sur la page.
- **Champ du slicer :** Dans le panneau **Données** (à droite), dans **Champs du visuel**, glisser **`DimClub[ClubName]`** (ou `DimClub[ClubKey]` si tu préfères filtrer par clé ; les mesures utilisent ClubKey, le nom est plus lisible pour l’utilisateur).
- **Style d’affichage :** Avec le slicer sélectionné, ouvrir le panneau **Format du visuel** (icône rouleau de peinture). Sous **Sélection** (ou **Options** selon la version) :
  - **Style visuel** : choisir **Liste** (cases à cocher) ou **Liste déroulante**.
  - Activer **Sélection multiple** (option « Plusieurs sélections » ou « Multi-select ») pour permettre de cocher plusieurs clubs.
- **Comportement :** L’utilisateur peut cocher **exactement 2 clubs** pour voir le tête-à-tête. Si 0, 1 ou plus de 2 clubs sont sélectionnés, les mesures renvoient BLANK() (grâce au test `COUNTROWS(VALUES(DimClub[ClubKey])) <> 2`).
- **Indication à l’utilisateur :** Optionnel : ajouter une **zone de texte** au-dessus ou à côté du slicer avec le libellé « Choisir 2 clubs pour le bilan tête-à-tête » pour rappeler la règle.
- **Liaison aux visuels :** Par défaut, le slicer filtre tous les visuels de la page qui utilisent le modèle (cartes, graphiques). Si une page a plusieurs slicers, vérifier dans **Modélisation** ou dans **Modifier les interactions** (Format → Modifier les interactions) que ce slicer filtre bien les visuels 1 et 4.

2. **KPI / cartes :** ajouter 4–6 visuels « Carte » : titre + valeur.  
   - Carte 1 : titre « Victoires [Club 1] », valeur `[Rivalry Wins Club1]`.  
   - Carte 2 : « Nuls », valeur `[Rivalry Draws]`.  
   - Carte 3 : « Victoires [Club 2] », valeur `[Rivalry Wins Club2]`.  
   - Cartes 4–5 : « Buts Club 1 » / « Buts Club 2 » avec `[Rivalry Goals Club1]` / `[Rivalry Goals Club2]`.  
   (Pour afficher le nom du club 1/2 dans le titre, on peut utiliser une mesure texte ou laisser « Club 1 » / « Club 2 ».)
3. **Donut W–D–L :** visuel **Donut**. Légende = une table à 3 lignes (W, D, L) ou colonne calculée ; valeur = mesure qui renvoie [Rivalry Wins Club1], [Rivalry Draws], [Rivalry Wins Club2] selon la catégorie. En pratique : créer une table **RivalryResult** (ResultLabel, Ordre) avec lignes ("Victoires club 1", 1), ("Nuls", 2), ("Victoires club 2", 3), et une mesure **Rivalry Result Value** qui selon MAX(RivalryResult[ResultLabel]) renvoie la bonne mesure. Ou plus simple : **graphique en barres** à 2 barres (Club 1, Club 2), valeur = Points ; ou 3 barres (Victoires 1, Nuls, Victoires 2).
4. **Format :** Lier le slicer à tous les visuels de la page. Optionnel : désactiver « Sélection multiple » et utiliser **deux slicers** (Club A, Club B) avec chacun une sélection unique — alors il faut deux tables de clubs ou un paramètre pour désigner « premier » / « second » club.

---

## 2. Classement des « rivalités » (paires) par intensité ou buts

**Type :** Graphique en barres horizontales ou tableau.

**But :** Montrer les paires de clubs qui ont produit le plus de buts (ou le plus de sanctions) dans leurs confrontations directes — repérer les chocs les plus animés ou les plus tendus.

**Données :** **Une ligne par paire non orientée** (chaque duel A–B n’apparaît qu’une fois). La table doit utiliser ClubKey1 = MIN(Home, Away) et ClubKey2 = MAX(Home, Away) pour dédupliquer, sinon on ne compte que les matchs dans un seul sens (ex. seulement PSG domicile vs OM extérieur).

- **Table calculée** (Modélisation → Nouvelle table) :
  ```dax
  RivalryPair =
  VAR PairsWithOrder =
      ADDCOLUMNS(
          SUMMARIZE(DimMatch, DimMatch[HomeClubKey], DimMatch[AwayClubKey]),
          "ClubKey1", IF([HomeClubKey] < [AwayClubKey], [HomeClubKey], [AwayClubKey]),
          "ClubKey2", IF([HomeClubKey] < [AwayClubKey], [AwayClubKey], [HomeClubKey])
      )
  VAR UnorderedPairs = SUMMARIZE(PairsWithOrder, [ClubKey1], [ClubKey2])
  RETURN
      ADDCOLUMNS(
          UnorderedPairs,
          "PairLabel",
              LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [ClubKey1])
                  & " – " & LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [ClubKey2])
      )
  ```
- **Mesure Total Goals** (contexte = une ligne RivalryPair) — **les deux sens** (Home/Away ou Away/Home) :
  ```dax
  Total Goals in Rivalry =
  VAR C1 = MAX(RivalryPair[ClubKey1])
  VAR C2 = MAX(RivalryPair[ClubKey2])
  RETURN
      CALCULATE(
          SUM(FactClubMatch[GoalsFor]),
          FILTER(DimMatch,
              (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                  || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1)
          )
      )
  ```
  Variante sanctions : idem en filtrant les matchs par (Home=C1 AND Away=C2) OR (Home=C2 AND Away=C1), puis FactSanction par dates de ces matchs et ClubKey ∈ {C1, C2}.

**Power BI — étapes :** Barres horizontales → Axe Y = `RivalryPair[PairLabel]`, Valeur = `[Total Goals in Rivalry]` (ou `[Sanctions in Rivalry]`). Trier décroissant ; Filtre visuel Top N (ex. 15). Tooltips : PairLabel, les deux mesures.

---

## 2 BIS. Sanctions par rivalité (raisons sélectionnées, exclusions)

**Type :** Graphique en barres horizontales ou tableau.

**But :** Pour chaque paire de clubs, afficher le **nombre de sanctions** survenues lors de leurs confrontations directes, en ne comptant que certaines raisons et en **excluant** : « Accumulation of yellow cards », « Controversial social media post », « Late for anti-doping control ». Permet de comparer l’intensité disciplinaire (hors cartons accumulés et hors faits hors-terrain) entre les rivalités.

**Raisons retenues (à inclure dans la mesure) :**  
Toutes celles utilisées dans le document Sanctions (Ligue-Wide) **sauf** les trois ci‑dessus. Liste à utiliser dans le FILTER sur `FactSanction[Reason]` :
- Abusive language in mixed zone  
- Deliberate elbow strike  
- Inappropriate gesture towards an opponent  
- Insulting the referee  
- On-field brawl  
- Red card - last man foul  
- Repeated protesting  
- Simulation  
- Spitting  
- Straight red card - dangerous tackle  
- Unsportsmanlike conduct  

**Données :** Table **RivalryPair**, **FactSanction** (ClubKey, SanctionDateKey, Reason), **DimMatch**. Les sanctions sont rattachées aux matchs de la paire par date (SanctionDateKey = date du match) et par club (ClubKey = HomeClubKey ou AwayClubKey de la paire).

**Mesure DAX (contexte = une ligne RivalryPair) :**

On restreint les sanctions aux **dates des matchs** de la paire (FactSanction n’étant pas liée à DimMatch, on utilise les DateKey de ces matchs) et aux **clubs** de la paire, puis on filtre sur les raisons (exclusion des 3 indiquées, inclusion des 11 autres).

```dax
Sanctions in Rivalry (selected reasons) =
VAR C1 = MAX(RivalryPair[ClubKey1])
VAR C2 = MAX(RivalryPair[ClubKey2])
VAR MatchDates =
    CALCULATETABLE(
        VALUES(DimMatch[DateKey]),
        FILTER(DimMatch,
            (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1)
        )
    )
RETURN
    CALCULATE(
        COUNTROWS(FactSanction),
        FILTER(FactSanction,
            (FactSanction[ClubKey] = C1 || FactSanction[ClubKey] = C2)
            && CONTAINS(MatchDates, [DateKey], FactSanction[SanctionDateKey])
            && FactSanction[Reason] <> "Accumulation of yellow cards"
            && FactSanction[Reason] <> "Controversial social media post"
            && FactSanction[Reason] <> "Late for anti-doping control"
            && (
                FactSanction[Reason] = "Abusive language in mixed zone"
                || FactSanction[Reason] = "Deliberate elbow strike"
                || FactSanction[Reason] = "Inappropriate gesture towards an opponent"
                || FactSanction[Reason] = "Insulting the referee"
                || FactSanction[Reason] = "On-field brawl"
                || FactSanction[Reason] = "Red card - last man foul"
                || FactSanction[Reason] = "Repeated protesting"
                || FactSanction[Reason] = "Simulation"
                || FactSanction[Reason] = "Spitting"
                || FactSanction[Reason] = "Straight red card - dangerous tackle"
                || FactSanction[Reason] = "Unsportsmanlike conduct"
            )
        )
    )
```

*Note :* CONTAINS(MatchDates, DimMatch[DateKey], FactSanction[SanctionDateKey]) exige que la table `MatchDates` ait une colonne avec la même référence que DimMatch[DateKey]. Si la table calculée a une colonne nommée différemment (ex. `[DateKey]` sans préfixe), utiliser cette colonne dans CONTAINS. Adapter la casse des libellés **Reason** si le modèle diffère.

**Power BI — étapes :** Barres horizontales. Axe Y = `RivalryPair[PairLabel]`, Valeur = `[Sanctions in Rivalry (selected reasons)]`. Trier décroissant, Top N (ex. 15). Tooltips : PairLabel, [Sanctions in Rivalry (selected reasons)]. Optionnel : ajouter [Total Goals in Rivalry] en tooltip pour comparer buts et sanctions.

---

## 3. Affluence dans les confrontations directes

**Type :** Barres horizontales ou tableau.

**But :** Pour chaque paire de clubs (ou top N), afficher l’affluence moyenne (ou totale) lors de leurs matchs — mettre en avant les « classiques » qui remplissent les stades.

**Données :** Même table de paires RivalryPair (ou liste de matchs). Mesure d’affluence dans le contexte « matchs de cette paire » : utiliser FactAttendance (Attendance, FillRatePct) ou DimMatch[Attendance] selon le modèle, en filtrant les matchs par (HomeClubKey, AwayClubKey).

**Mesure DAX (contexte = une ligne RivalryPair)** — inclure **les deux sens** (matchs A–B et B–A) :  
Si l’affluence est dans **DimMatch[Attendance]** :  
`Avg Attendance in Rivalry = VAR C1 = MAX(RivalryPair[ClubKey1]) VAR C2 = MAX(RivalryPair[ClubKey2]) RETURN CALCULATE(AVERAGE(DimMatch[Attendance]), FILTER(DimMatch, (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2) || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1)))`.  
Si elle est dans **FactAttendance** (lié à DimMatch par MatchKey) : même FILTER sur DimMatch (avec les deux sens), puis CALCULATE(AVERAGE(FactAttendance[Attendance])).

**Power BI — étapes :** Barres horizontales. Axe Y = `RivalryPair[PairLabel]`, Valeur = `[Avg Attendance in Rivalry]`. Trier décroissant, Top N (ex. 15). Tooltips : PairLabel, [Avg Attendance in Rivalry].

---

## 3 BIS. Taux de remplissage (FillRate) dans les confrontations directes

**Type :** Barres horizontales ou tableau.

**But :** Comme le visuel 3, mais avec le **taux de remplissage moyen** (FillRatePct) au lieu de l’affluence — identifier les paires dont les matchs remplissent le plus le stade (en %).

**Schéma :** RivalryPair n’a **aucune relation directe** vers DimMatch ni FactAttendance (uniquement vers DimClub via HomeClubKey et AwayClubKey). FillRatePct est dans **FactAttendance** ; FactAttendance est liée à **DimMatch** par MatchKey. Pour obtenir le bon contexte « matchs de cette paire », il faut filtrer explicitement DimMatch puis laisser ce filtre se propager à FactAttendance. En outre, le filtre RivalryPair → DimClub peut sur-filtrer FactAttendance (toutes les lignes où le club est l’un des deux), au lieu de ne garder que les matchs **entre** ces deux clubs ; il faut donc neutraliser le filtre sur DimClub dans la mesure et ne s’appuyer que sur le filtre DimMatch.

**Mesure DAX (contexte = une ligne RivalryPair) :**

```dax
Avg Fill Rate in Rivalry =
VAR C1 = MAX(RivalryPair[ClubKey1])
VAR C2 = MAX(RivalryPair[ClubKey2])
RETURN
    CALCULATE(
        AVERAGE(FactAttendance[FillRatePct]),
        FILTER(DimMatch,
            (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1)),
        REMOVEFILTERS(DimClub)
    )
```

*Pourquoi les deux sens :* une rivalité = tous les matchs entre les deux clubs ; il faut inclure (Home=C1, Away=C2) **et** (Home=C2, Away=C1).  
*Pourquoi REMOVEFILTERS(DimClub) :* en contexte d’une ligne RivalryPair, les relations RivalryPair → DimClub filtrent DimClub sur les deux clubs ; ce filtre peut sur-filtrer FactAttendance. En le retirant, on ne garde que le filtre explicite sur DimMatch (matchs de la paire), qui se propage à FactAttendance via MatchKey.

**Power BI — étapes :** Barres horizontales. Axe Y = `RivalryPair[PairLabel]`, Valeur = `[Avg Fill Rate in Rivalry]`. Trier décroissant, Top N (ex. 15). **Format :** afficher la valeur en **%** (Format du champ → Pourcentage). Tooltips : PairLabel, [Avg Fill Rate in Rivalry] ; optionnel : [Avg Attendance in Rivalry] pour comparer.

---

## 4. Historique des matchs d’une rivalité (chronologie)

**Type :** Table ou timeline.

**But :** Pour **deux clubs sélectionnés**, lister tous les matchs les opposant : date, journée, score (ou résultat W/D/L), éventuellement stade / affluence.

**Prérequis :** Même slicer que pour le visuel 1 : **2 clubs** sélectionnés dans `DimClub[ClubName]` (sélection multiple). Le visuel Table a pour source **DimMatch** ; pour n’afficher que les matchs entre ces deux clubs, utiliser la mesure **Is Rivalry Match** en filtre du visuel.

---

### Colonnes calculées (DimMatch)

Créées dans le modèle pour afficher les noms des clubs dans la table :

```dax
HomeClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], DimMatch[HomeClubKey])
AwayClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], DimMatch[AwayClubKey])
```

---

### Mesures (DimMatch)

**Buts à domicile / à l’extérieur** (contexte = une ligne DimMatch) :

```dax
Goals Home = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[ClubKey] = MAX(DimMatch[HomeClubKey]))
Goals Away = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[ClubKey] = MAX(DimMatch[AwayClubKey]))
```

**Filtre « match de la rivalité »** : renvoie TRUE lorsque exactement 2 clubs sont sélectionnés et que la ligne courante est un match entre ces deux clubs. À utiliser dans le **filtre du visuel Table** (Filtre sur ce visuel → **Is Rivalry Match** = TRUE).

```dax
Is Rivalry Match =
VAR SelectedKeys = VALUES(DimClub[ClubKey])
VAR HomeKey = MAX(DimMatch[HomeClubKey])
VAR AwayKey = MAX(DimMatch[AwayClubKey])
RETURN
    COUNTROWS(SelectedKeys) = 2
    && CONTAINS(SelectedKeys, DimClub[ClubKey], HomeKey)
    && CONTAINS(SelectedKeys, DimClub[ClubKey], AwayKey)
    && HomeKey <> AwayKey
```

---

### Power BI — étapes

1. **Colonnes et mesures** : Les colonnes calculées **HomeClubName**, **AwayClubName** et les mesures **Goals Home**, **Goals Away**, **Is Rivalry Match** sont définies sur la table **DimMatch** (voir ci‑dessus ou modèle .bim).
2. **Visuel Table** : Source = **DimMatch**. Colonnes à ajouter : **DimMatch[Matchday]** (ou **DimDate[Date]** via la relation DateKey), **DimMatch[HomeClubName]**, **DimMatch[AwayClubName]**, **[Goals Home]**, **[Goals Away]**. Optionnel : **DimMatch[Stadium]**, **DimMatch[Attendance]**.
3. **Filtre du visuel** : Filtre sur ce visuel → **Is Rivalry Match** = TRUE. Ainsi, lorsque l’utilisateur sélectionne exactement 2 clubs dans le slicer, la table n’affiche que les matchs entre ces deux clubs.
4. **Tri** : Trier par **Matchday** ou **Date** (croissant).

---

## 5. Matrice ou heatmap « qui bat qui »

**Type :** Matrice (rows = club, columns = club) ou heatmap.

**But :** Vue ligue : pour chaque paire (Club i vs Club j), afficher un indicateur (points remportés par le club en ligne à domicile ou sur l’ensemble des confrontations, ou différence de buts). Permet de repérer d’un coup d’œil les rapports de force.

**Option A — Tableau simple (recommandé)**  
Source = **RivalryPair** (une ligne par paire non orientée, ClubKey1 / ClubKey2). Colonnes : `RivalryPair[PairLabel]`, puis une mesure **Points club 1 dans la paire** = somme des points du club ClubKey1 dans **tous** les matchs de la paire (sens A→B et B→A). Idem **Points club 2 dans la paire** pour ClubKey2. Colonne optionnelle : différence de buts ou « vainqueur » (texte). Trier par points ou par libellé.

**Option B — Matrice « qui bat qui »**  
Il faut **deux dimensions** (club en ligne, club en colonne). Power BI ne permet pas facilement d’utiliser deux fois la même table avec des rôles différents pour Lignes et Colonnes avec une mesure qui dépend des deux. **Contournement :** créer une table **DimClubOpponent** = copie de DimClub (ou même table avec un rôle « Adversaire »). Lignes = DimClub[ClubName], Colonnes = DimClubOpponent[ClubName], Valeur = mesure qui, pour le club ligne et le club colonne, renvoie les points du club ligne dans les matchs les opposant. Cette mesure nécessite de connaître « club ligne » et « club colonne » : utiliser MAX(DimClub[ClubKey]) et MAX(DimClubOpponent[ClubKey]) si les deux tables sont dans le contexte (relations vers FactClubMatch / DimMatch à gérer).

**Mesures pour Option A (points de chaque club dans la paire, tous matchs confondus) :**

```dax
Points Club1 in Rivalry =
VAR C1 = MAX(RivalryPair[ClubKey1])
VAR C2 = MAX(RivalryPair[ClubKey2])
RETURN
    CALCULATE(
        SUM(FactClubMatch[Points]),
        FactClubMatch[ClubKey] = C1,
        FILTER(DimMatch,
            (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
    )
Points Club2 in Rivalry =
VAR C1 = MAX(RivalryPair[ClubKey1])
VAR C2 = MAX(RivalryPair[ClubKey2])
RETURN
    CALCULATE(
        SUM(FactClubMatch[Points]),
        FactClubMatch[ClubKey] = C2,
        FILTER(DimMatch,
            (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
    )
```

**Power BI — étapes (Option A) :** Visuel **Table**. Champs : `RivalryPair[PairLabel]`, `[Points Club1 in Rivalry]`, `[Points Club2 in Rivalry]`. Optionnel : mesure « Vainqueur » (si Points Club1 > Points Club2 alors nom Club 1, sinon si Points Club2 > alors nom Club 2, sinon "Nul"). Trier par une des colonnes de points (décroissant).

---

## Récap des visuels

| # | Visuel | Objectif principal |
|---|--------|-------------------|
| 1 | KPI + donut/barres W–D–L, buts | Bilan tête-à-tête pour 2 clubs choisis |
| 2 | Barres / tableau | Top paires par buts ou par sanctions (rivalités « chaudes ») |
| 2 BIS | Barres / tableau | Sanctions par paire (raisons sélectionnées ; exclusions : accumulation yellow cards, social media, anti-doping) |
| 3 | Barres / tableau | Top paires par affluence (classiques à fort enjeu) |
| 3 BIS | Barres / tableau | Top paires par taux de remplissage (FillRate %) |
| 4 | Table / timeline | Liste chronologique des matchs pour 2 clubs sélectionnés |
| 5 | Matrice ou tableau | Qui bat qui (points ou écart de buts par paire) |

**Interactions :** Lier le slicer « 2 clubs » aux visuels 1 et 4. Optionnel : drill-through ou filtre croisé depuis le visuel 2, 2 BIS, 3 ou 3 BIS (clic sur une paire) vers une page détail ou le visuel 4.
