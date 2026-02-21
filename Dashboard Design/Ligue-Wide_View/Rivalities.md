# Ligue-Wide View — Rivalités (head-to-head)

**Category:** Confrontations directes entre clubs (rivalités, derbys, classiques).

**Principe :** Une rivalité = une paire de clubs qui se sont affrontés. Visuels 1 et 4 : slicer « 2 clubs » sur DimClub. Visuels 2, 3 et 5 : table calculée RivalryPair (une ligne par paire Home/Away issue de DimMatch).

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
                (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                    || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1),
                CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[ClubKey] = C1) = 1
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

1. Créer un **slicer** avec `DimClub[ClubName]`, mode **liste** ou **dropdown**, avec **sélection multiple** activée. Indiquer à l’utilisateur de choisir **exactement 2 clubs** (ou ajouter une alerte si ≠ 2).
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

**Données :** Une ligne par paire (HomeClub, AwayClub) issue de DimMatch. Créer une **table calculée** des paires (sans modifier les tables existantes), par ex. :

- **Table calculée** (Modélisation → Nouvelle table) :
  ```dax
  RivalryPair =
  ADDCOLUMNS(
      SUMMARIZE(DimMatch, DimMatch[HomeClubKey], DimMatch[AwayClubKey]),
      "PairLabel",
          LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [HomeClubKey])
              & " – " & LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [AwayClubKey])
  )
  ```
- **Mesures** à créer (contexte = une ligne RivalryPair) :
  ```dax
  Total Goals in Rivalry =
  CALCULATE(SUM(FactClubMatch[GoalsFor]),
      FILTER(DimMatch,
          DimMatch[HomeClubKey] = MAX(RivalryPair[HomeClubKey])
              && DimMatch[AwayClubKey] = MAX(RivalryPair[AwayClubKey])))
  ```
  Variante sanctions : CALCULATE(COUNTROWS(FactSanction), FILTER(FactSanction, ClubKey = HomeK ou AwayK), FILTER(DimMatch, ...)) en récupérant les dates des matchs de la paire.

**Mesure (dans le contexte d’une ligne RivalryPair) :**  
Total buts dans les matchs de cette paire = CALCULATE(SUM(FactClubMatch[GoalsFor]), FILTER(DimMatch, DimMatch[HomeClubKey] = MAX(RivalryPair[HomeClubKey]) && DimMatch[AwayClubKey] = MAX(RivalryPair[AwayClubKey]))).  
Variante : même idée pour compter les sanctions (FactSanction filtrée par SanctionDateKey = dates des matchs de la paire et ClubKey ∈ {HomeClubKey, AwayClubKey}).

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
VAR HomeK = MAX(RivalryPair[HomeClubKey])
VAR AwayK = MAX(RivalryPair[AwayClubKey])
VAR MatchDates =
    CALCULATETABLE(
        VALUES(DimMatch[DateKey]),
        FILTER(DimMatch, DimMatch[HomeClubKey] = HomeK && DimMatch[AwayClubKey] = AwayK)
    )
RETURN
    CALCULATE(
        COUNTROWS(FactSanction),
        FILTER(FactSanction,
            (FactSanction[ClubKey] = HomeK || FactSanction[ClubKey] = AwayK)
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

**Mesure DAX (contexte = une ligne RivalryPair) :**  
Si l’affluence est dans **DimMatch[Attendance]** :  
`Avg Attendance in Rivalry = CALCULATE(AVERAGE(DimMatch[Attendance]), FILTER(DimMatch, DimMatch[HomeClubKey] = MAX(RivalryPair[HomeClubKey]) && DimMatch[AwayClubKey] = MAX(RivalryPair[AwayClubKey])))`.  
Si elle est dans **FactAttendance** (lié à DimMatch par MatchKey) : même FILTER sur DimMatch, puis CALCULATE(AVERAGE(FactAttendance[Attendance])).

**Power BI — étapes :** Barres horizontales. Axe Y = `RivalryPair[PairLabel]`, Valeur = `[Avg Attendance in Rivalry]`. Trier décroissant, Top N (ex. 15). Tooltips : PairLabel, [Avg Attendance in Rivalry].

---

## 3 BIS. Taux de remplissage (FillRate) dans les confrontations directes

**Type :** Barres horizontales ou tableau.

**But :** Comme le visuel 3, mais avec le **taux de remplissage moyen** (FillRatePct) au lieu de l’affluence — identifier les paires dont les matchs remplissent le plus le stade (en %).

**Schéma :** RivalryPair n’a **aucune relation directe** vers DimMatch ni FactAttendance (uniquement vers DimClub via HomeClubKey et AwayClubKey). FillRatePct est dans **FactAttendance** ; FactAttendance est liée à **DimMatch** par MatchKey. Pour obtenir le bon contexte « matchs de cette paire », il faut filtrer explicitement DimMatch puis laisser ce filtre se propager à FactAttendance. En outre, le filtre RivalryPair → DimClub peut sur-filtrer FactAttendance (toutes les lignes où le club est l’un des deux), au lieu de ne garder que les matchs **entre** ces deux clubs ; il faut donc neutraliser le filtre sur DimClub dans la mesure et ne s’appuyer que sur le filtre DimMatch.

**Mesure DAX (contexte = une ligne RivalryPair) :**

```dax
Avg Fill Rate in Rivalry =
VAR HomeK = MAX(RivalryPair[HomeClubKey])
VAR AwayK = MAX(RivalryPair[AwayClubKey])
RETURN
    CALCULATE(
        AVERAGE(FactAttendance[FillRatePct]),
        FILTER(DimMatch,
            DimMatch[HomeClubKey] = HomeK && DimMatch[AwayClubKey] = AwayK),
        REMOVEFILTERS(DimClub)
    )
```

*Pourquoi REMOVEFILTERS(DimClub) :* en contexte d’une ligne RivalryPair, les relations RivalryPair → DimClub filtrent DimClub sur les deux clubs de la paire ; ce filtre se répercute sur FactAttendance (liée à DimClub par ClubKey) et peut donner un résultat incorrect (tous les matchs à domicile de l’un ou l’autre club au lieu des matchs **entre** les deux). En retirant le filtre sur DimClub, on ne garde que le filtre explicite sur DimMatch (matchs de la paire), qui se propage à FactAttendance via MatchKey et restreint bien aux lignes d’affluence de ces matchs.

**Power BI — étapes :** Barres horizontales. Axe Y = `RivalryPair[PairLabel]`, Valeur = `[Avg Fill Rate in Rivalry]`. Trier décroissant, Top N (ex. 15). **Format :** afficher la valeur en **%** (Format du champ → Pourcentage). Tooltips : PairLabel, [Avg Fill Rate in Rivalry] ; optionnel : [Avg Attendance in Rivalry] pour comparer.

---

## 4. Historique des matchs d’une rivalité (chronologie)

**Type :** Table ou timeline.

**But :** Pour **deux clubs sélectionnés**, lister tous les matchs les opposant : date, journée, score (ou résultat W/D/L), éventuellement stade / affluence.

**Prérequis :** Même slicer que pour le visuel 1 : **2 clubs** sélectionnés dans `DimClub[ClubName]` (sélection multiple). Le visuel Table est filtré par ce slicer ; il faut que les lignes affichées soient les **matchs** opposant ces deux clubs, donc la source du visuel doit être **DimMatch** (ou une table qui a une ligne par match avec Date, Matchday, domicile, extérieur, score).

**Champs à afficher :** Pour chaque match entre les 2 clubs : Date, Matchday, équipe à domicile, équipe à l’extérieur, buts domicile, buts extérieur, résultat.  
- **DimMatch** fournit MatchKey, DateKey, Matchday, HomeClubKey, AwayClubKey, Stadium, Attendance.  
- **DimDate** fournit la date affichable (via DateKey).  
- Les noms des clubs : via **DimClub** (relation HomeClubKey / AwayClubKey). Pour avoir « Domicile » et « Extérieur » dans la même table visuelle, il faut soit deux colonnes calculées sur DimMatch (HomeClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], DimMatch[HomeClubKey]), AwayClubName = idem), soit utiliser DimClub deux fois (rôle « Domicile » / « Extérieur ») si le visuel Table le permet.

**Mesures pour le score (optionnel si déjà dans FactClubMatch) :** En contexte d’une ligne DimMatch, buts domicile = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[ClubKey] = DimMatch[HomeClubKey]). Buts extérieur = idem avec AwayClubKey.

**Power BI — étapes :**

1. Créer deux **colonnes calculées** sur **DimMatch** (si pas déjà fait) :  
   `HomeClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], DimMatch[HomeClubKey])`  
   `AwayClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], DimMatch[AwayClubKey])`
2. Mesures pour les buts dans le contexte d’une ligne DimMatch :  
   `Goals Home = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[ClubKey] = MAX(DimMatch[HomeClubKey]))`  
   `Goals Away = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[ClubKey] = MAX(DimMatch[AwayClubKey]))`
3. Visuel **Table**. Base = **DimMatch**. Colonnes : DimDate[Date] (ou DimMatch[Matchday]), DimMatch[HomeClubName], DimMatch[AwayClubName], [Goals Home], [Goals Away], et optionnel DimMatch[Stadium], DimMatch[Attendance].
4. **Filtre :** Le slicer « 2 clubs » filtre DimClub ; pour que la table n’affiche que les matchs entre ces 2 clubs, DimMatch doit être filtré par HomeClubKey et AwayClubKey. Comme DimMatch est lié à DimClub via HomeClubKey et AwayClubKey, la sélection de 2 clubs dans DimClub ne filtre pas automatiquement « les matchs où Home et Away sont exactement ces 2 clubs ». Il faut un **filtre visuel** ou une **mesure** qui renvoie BLANK() pour les lignes qui ne sont pas des matchs entre les 2 clubs sélectionnés. Solution simple : **filtrer DimMatch** avec un filtre tel que « HomeClubKey est dans la liste des clubs sélectionnés ET AwayClubKey est dans la liste des clubs sélectionnés ET HomeClubKey <> AwayClubKey ». En Power BI, si le slicer est sur DimClub[ClubKey] (ou ClubName), la relation DimClub → DimMatch existe via HomeClubKey ; une seule relation est active, donc le filtre ne s’applique qu’à un côté. Pour filtrer les matchs « où les deux clubs sont ceux sélectionnés », utiliser une **mesure de filtre** : ajouter une colonne calculée sur DimMatch, ex. `IsRivalryMatch = (COUNTROWS(VALUES(DimClub[ClubKey])) = 2) && (DimMatch[HomeClubKey] IN VALUES(DimClub[ClubKey])) && (DimMatch[AwayClubKey] IN VALUES(DimClub[ClubKey])) && (DimMatch[HomeClubKey] <> DimMatch[AwayClubKey])`, puis Filtre du visuel Table : IsRivalryMatch = TRUE. (Ou créer une table dérivée / vue qui ne contient que ces matchs.)
5. Trier par Date ou Matchday (croissant).

---

## 5. Matrice ou heatmap « qui bat qui »

**Type :** Matrice (rows = club, columns = club) ou heatmap.

**But :** Vue ligue : pour chaque paire (Club i vs Club j), afficher un indicateur (points remportés par le club en ligne à domicile ou sur l’ensemble des confrontations, ou différence de buts). Permet de repérer d’un coup d’œil les rapports de force.

**Option A — Tableau simple (recommandé)**  
Source = **RivalryPair**. Colonnes : `RivalryPair[PairLabel]`, puis une mesure **Points club 1 (Home) dans la paire** = en contexte de la ligne RivalryPair, somme des points du club Home dans les matchs de cette paire (CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[ClubKey] = MAX(RivalryPair[HomeClubKey]), FILTER(DimMatch, ...))). Idem pour le club Away. Colonne optionnelle : différence de buts ou « vainqueur » (texte). Trier par points ou par libellé.

**Option B — Matrice « qui bat qui »**  
Il faut **deux dimensions** (club en ligne, club en colonne). Power BI ne permet pas facilement d’utiliser deux fois la même table avec des rôles différents pour Lignes et Colonnes avec une mesure qui dépend des deux. **Contournement :** créer une table **DimClubOpponent** = copie de DimClub (ou même table avec un rôle « Adversaire »). Lignes = DimClub[ClubName], Colonnes = DimClubOpponent[ClubName], Valeur = mesure qui, pour le club ligne et le club colonne, renvoie les points du club ligne dans les matchs les opposant. Cette mesure nécessite de connaître « club ligne » et « club colonne » : utiliser MAX(DimClub[ClubKey]) et MAX(DimClubOpponent[ClubKey]) si les deux tables sont dans le contexte (relations vers FactClubMatch / DimMatch à gérer).

**Mesure pour Option A (points du club domicile dans la paire) :**

```dax
Points Home in Rivalry =
CALCULATE(
    SUM(FactClubMatch[Points]),
    FactClubMatch[ClubKey] = MAX(RivalryPair[HomeClubKey]),
    FILTER(DimMatch,
        DimMatch[HomeClubKey] = MAX(RivalryPair[HomeClubKey])
            && DimMatch[AwayClubKey] = MAX(RivalryPair[AwayClubKey]))
)
Points Away in Rivalry =
CALCULATE(
    SUM(FactClubMatch[Points]),
    FactClubMatch[ClubKey] = MAX(RivalryPair[AwayClubKey]),
    FILTER(DimMatch,
        DimMatch[HomeClubKey] = MAX(RivalryPair[HomeClubKey])
            && DimMatch[AwayClubKey] = MAX(RivalryPair[AwayClubKey]))
)
```

**Power BI — étapes (Option A) :** Visuel **Table**. Champs : `RivalryPair[PairLabel]`, `[Points Home in Rivalry]`, `[Points Away in Rivalry]`. Optionnel : colonne calculée ou mesure « Vainqueur » (si Points Home > Points Away alors "Domicile", sinon si Points Away > alors "Extérieur", sinon "Nul"). Trier par une des colonnes de points (décroissant).

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
