# Player View — Fiche joueur : KPIs vs moyenne du poste

**Objectif :** Afficher une **fiche** avec les KPIs d’un **joueur sélectionné** mis en face des **mêmes KPIs** avec comme valeur la **moyenne des joueurs qui ont le même poste** (Position). Permet de comparer le joueur à la moyenne de son poste (Forward, Central midfielder, etc.).

**Tables :** `DimPlayer` (PlayerKey, FullName, **Position**), `FactPlayerSeason` (PlayerKey, ClubKey, Goals, Assists, MinutesPlayed, MatchesPlayed, Starts, YellowCards, RedCards, Shots, ShotsOnTarget, CleanSheets, Saves, etc.). Référence schéma : Ligue-Wide_View / 0_markdown.md.

---

## 1. Sélection du joueur

- **Slicer** sur `DimPlayer[FullName]` (ou `DimPlayer[PlayerKey]`) avec **sélection unique** : l’utilisateur choisit un joueur.
- Tous les visuels de la fiche sont filtrés par ce slicer (contexte = un seul joueur).

---

## 2. KPIs à afficher (exemples)

Choisir les indicateurs pertinents dans **FactPlayerSeason** (une ligne par joueur par saison), par exemple :

| KPI | Champ FactPlayerSeason | Unité |
|-----|------------------------|-------|
| Buts | Goals | entier |
| Passes décisives | Assists | entier |
| Minutes jouées | MinutesPlayed | entier |
| Matchs joués | MatchesPlayed | entier |
| Titularisations | Starts | entier |
| Cartons jaunes | YellowCards | entier |
| Cartons rouges | RedCards | entier |
| Tirs | Shots | entier |
| Tirs cadrés | ShotsOnTarget | entier |
| Clean sheets | CleanSheets | entier (gardien) |
| Arrêts | Saves | entier (gardien) |
| Dribbles réussis | SuccessfulDribbles | décimal |
| Interceptions | Interceptions | décimal |
| Tacles réussis | SuccessfulTackles | décimal |

---

## 3. Mesures DAX

Pour chaque KPI, il faut **deux mesures** : une pour la **valeur du joueur sélectionné**, une pour la **moyenne du poste**.

**Valeur du joueur (contexte = 1 joueur sélectionné)**  
En contexte d’un seul joueur, les mesures standards suffisent (le filtre du slicer restreint déjà à ce joueur) :

```dax
Player Goals = SUM(FactPlayerSeason[Goals])
Player Assists = SUM(FactPlayerSeason[Assists])
Player MinutesPlayed = SUM(FactPlayerSeason[MinutesPlayed])
// … idem pour les autres KPIs (SUM pour les totaux saison)
```

**Moyenne du poste (même Position que le joueur sélectionné)**  
On calcule la moyenne du KPI sur **tous les joueurs ayant le même poste** que le joueur sélectionné (y compris le joueur lui‑même, pour une moyenne « du poste »). On retire le filtre joueur puis on filtre par Position :

```dax
Avg Goals Same Position =
VAR CurrentPosition = SELECTEDVALUE(DimPlayer[Position])
RETURN
    IF(
        ISBLANK(CurrentPosition),
        BLANK(),
        CALCULATE(
            AVERAGE(FactPlayerSeason[Goals]),
            FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition)
        )
    )
```

*Explication :* `SELECTEDVALUE(DimPlayer[Position])` donne le poste du joueur sélectionné. `CALCULATE(AVERAGE(…), FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition))` supprime le filtre « un seul joueur » et ne garde que le filtre « même poste », donc on moyenne sur tous les joueurs de ce poste (FactPlayerSeason a une ligne par joueur par saison, donc AVERAGE par joueur puis moyenne entre joueurs selon le grain — si une ligne par joueur/saison, AVERAGE(FactPlayerSeason[Goals]) sur ce filtre donne la moyenne des Goals par joueur de ce poste).

Si **FactPlayerSeason** a une ligne par joueur par saison, la moyenne « par poste » peut être :
- **Moyenne des totaux joueur** : `AVERAGEX(VALUES(DimPlayer[PlayerKey]), CALCULATE(SUM(FactPlayerSeason[Goals])))` dans le contexte FILTER(DimPlayer, Position = CurrentPosition), ou plus simple si une ligne par joueur : `CALCULATE(AVERAGE(FactPlayerSeason[Goals]), FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition))` (chaque ligne FactPlayerSeason = un joueur, donc AVERAGE = moyenne des Goals des joueurs du poste).

Exemple pour les autres KPIs :

```dax
Avg Assists Same Position =
VAR CurrentPosition = SELECTEDVALUE(DimPlayer[Position])
RETURN
    IF(ISBLANK(CurrentPosition), BLANK(),
        CALCULATE(AVERAGE(FactPlayerSeason[Assists]),
            FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition)))

Avg MinutesPlayed Same Position =
VAR CurrentPosition = SELECTEDVALUE(DimPlayer[Position])
RETURN
    IF(ISBLANK(CurrentPosition), BLANK(),
        CALCULATE(AVERAGE(FactPlayerSeason[MinutesPlayed]),
            FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition)))
```

Répéter le même pattern pour chaque KPI (Starts, YellowCards, Shots, etc.).

---

## 4. Jauges par KPI (min, max, moyenne en repère)

Pour chaque KPI, une **jauge** affiche la valeur du joueur sélectionné avec un repère visuel par rapport à l’échelle et à la moyenne.

**Comportement des jauges :**

- **Valeur affichée :** celle du joueur (mesures « Player … », ex. `[Player Goals]`).
- **Min et Max de l’échelle :** définissent les bornes de la jauge. Deux modes possibles (paramétrable, ex. slicer ou paramètre) :
  - **Global :** min et max calculés sur **tous les joueurs** (ex. `MIN(FactPlayerSeason[Goals])` et `MAX(FactPlayerSeason[Goals])` en `ALL(DimPlayer)` ou équivalent).
  - **Même poste :** min et max calculés uniquement parmi les joueurs ayant le **même poste** que le joueur sélectionné (même logique que les mesures « Avg … Same Position », avec `MIN` / `MAX` au lieu de `AVERAGE`).
- **Ligne « objectif » (repère) :** la **moyenne** (globale ou du poste, selon le même choix que min/max) est affichée comme **ligne de référence** (constant line / objectif) sur la jauge. Ce n’est **pas un objectif à atteindre** : le but est d’avoir un **trait comme repère** pour comparer visuellement la valeur du joueur à cette moyenne.

**Résumé :**

| Élément jauge | Option « Global » | Option « Même poste » |
|---------------|-------------------|------------------------|
| Min | Min du KPI sur tous les joueurs | Min du KPI sur les joueurs du même poste |
| Max | Max du KPI sur tous les joueurs | Max du KPI sur les joueurs du même poste |
| Ligne repère | Moyenne du KPI (tous les joueurs) | Moyenne du KPI (même poste) — ex. `[Avg Goals Same Position]` |

**Mesures DAX suggérées pour min/max :** même pattern que les moyennes, en remplaçant `AVERAGE` par `MIN` et `MAX`, avec `ALL(DimPlayer)` pour le global et `FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition)` pour le poste.

---

## 5. Mise en page Power BI (fiche côte à côte)

- **Une jauge par KPI** (cf. section 4) : valeur = joueur ; min/max = global ou même poste ; ligne repère = moyenne (affichée comme « objectif » sans être un objectif).
- **Deux colonnes** (ou deux blocs) sur la même page :
  - **Colonne 1 — Joueur sélectionné :** titre « [Nom du joueur] » ou « Joueur ». Cartes KPI ou jauges : `[Player Goals]`, `[Player Assists]`, `[Player MinutesPlayed]`, etc.
  - **Colonne 2 — Moyenne du poste :** titre « Moyenne [Position] » (ex. « Moyenne Forward »). Cartes KPI : `[Avg Goals Same Position]`, etc., ou repère intégré dans les jauges.
- **Choix min/max :** slicer ou paramètre pour basculer entre **min/max global** (tous les joueurs) et **min/max même poste**.
- **Slicer** : `DimPlayer[FullName]` (sélection unique), placé en haut ou à gauche ; il filtre toute la page.

**Résumé visuel :**

| KPI      | Joueur (sélectionné) | Moyenne poste |
|----------|----------------------|---------------|
| Buts     | [Player Goals]       | [Avg Goals Same Position] |
| Passes   | [Player Assists]     | [Avg Assists Same Position] |
| Minutes  | [Player MinutesPlayed] | [Avg MinutesPlayed Same Position] |
| …        | …                    | … |

---

## 6. Option : titre dynamique « Moyenne [Position] »

Pour afficher le libellé du poste dans le titre du bloc « Moyenne poste » :

- Zone de texte avec **valeur dynamique** (si Power BI le permet) ou mesure texte :
- `Position Label = SELECTEDVALUE(DimPlayer[Position]) & " (moyenne)"`
- Utiliser cette mesure dans un visuel titre ou une carte pour afficher « Forward (moyenne) », etc.

---

## 7. Précisions

- **Un seul joueur sélectionné :** les mesures « Player … » donnent les totaux de ce joueur ; les mesures « Avg … Same Position » donnent la moyenne des joueurs du même poste (tous, sans exclure le joueur sélectionné). Pour une moyenne **hors ce joueur**, utiliser `FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition && DimPlayer[PlayerKey] <> SELECTEDVALUE(DimPlayer[PlayerKey]))` dans le CALCULATE.
- **Grain FactPlayerSeason :** une ligne par joueur par saison ; si plusieurs saisons, SUM sur la page donne le total multi-saisons du joueur. Pour la **moyenne du poste** :
  - **Une saison filtrée** : `CALCULATE(AVERAGE(FactPlayerSeason[Goals]), FILTER(DimPlayer, …))` = moyenne des buts des joueurs du poste sur la saison.
  - **Plusieurs saisons (totaux joueur)** : pour comparer au total du joueur (SUM sur toutes ses saisons), préférer la moyenne des **totaux par joueur** :  
    `AVERAGEX(VALUES(DimPlayer[PlayerKey]), CALCULATE(SUM(FactPlayerSeason[Goals]), FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition)))` (en gardant la variable `CurrentPosition`).
