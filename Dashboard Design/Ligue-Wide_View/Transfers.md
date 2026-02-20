# Flux de transferts — Sankey / Alluvial

**Visual type :** Sankey ou Alluvial (flux Départ → Arrivée, épaisseur = montant)

**But :** Visualiser qui achète à qui et où circule l’argent : **DepartureClub → ArrivalClub**, épaisseur du flux = `SUM(FactTransfer[AmountME])`.

**Visuel utilisé :** **Sankey Diagram By ChartExpo** (visuel personnalisé AppSource). Le visuel fonctionne avec **Category Data** (une ou plusieurs catégories en séquence = colonnes de nœuds) et **Measure Data** (mesure = épaisseur des flux). On peut faire un flux à 2 niveaux (Départ → Arrivée) ou plus (ex. Départ → Type de transfert → Arrivée).

**Faisabilité :** Oui. La structure du modèle (FactTransfer, DimClub, DimDate, DimPlayer) permet de construire les flux et les interactions demandées.

**Référence modèle :** `0_markdown.md` — `FactTransfer` (ArrivalClubKey, DepartureClubKey, AmountME, TransferDateKey, TransferType, PlayerKey), `DimClub`, `DimDate`, `DimPlayer`.

---

## Champs utilisés

| Champ | Table | Rôle |
|-------|--------|------|
| ArrivalClubKey | FactTransfer | Club d’arrivée (cible du flux) |
| DepartureClubKey | FactTransfer | Club de départ (source du flux) |
| AmountME | FactTransfer | Montant en M€ — **épaisseur du flux** = SUM(AmountME) |
| TransferDateKey | FactTransfer | Période (filtre, tranches) |
| TransferType | FactTransfer | Filtre (Purchase, Loan, Free transfer, etc.) |
| PlayerKey | FactTransfer | Drill « top players » du flux sélectionné |
| ClubName | DimClub | Libellés des nœuds (Source = départ, Target = arrivée) |

*Note :* Dans le schéma le champ est `AmountME` ; en source il peut s’appeler `amount_ME`.

---

## 1. Données pour le Sankey

Un Sankey attend en général **Source**, **Target**, **Value**. Avec le modèle actuel, FactTransfer a deux FK vers DimClub (départ / arrivée), donc deux possibilités :

### Option A — Table calculée « flux agrégés » (recommandé)

Une table calculée DAX agrège les montants par paire (club de départ, club d’arrivée) et expose les noms pour le visuel, **sans modifier les tables existantes** (une nouvelle table uniquement) :

```dax
TransferFlow =
ADDCOLUMNS(
    SUMMARIZE(
        FactTransfer,
        FactTransfer[DepartureClubKey],
        FactTransfer[ArrivalClubKey]
    ),
    "AmountME", CALCULATE(SUM(FactTransfer[AmountME])),
    "DepartureClubName", LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [DepartureClubKey]),
    "ArrivalClubName", LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [ArrivalClubKey])
)
```

- **Source (nœud de gauche) :** `TransferFlow[DepartureClubName]`
- **Target (nœud de droite) :** `TransferFlow[ArrivalClubName]`
- **Value (épaisseur) :** `TransferFlow[AmountME]`

Pour que les filtres **TransferType** et **période (TransferDateKey)** s’appliquent, il faut que le visuel soit basé sur des données qui réagissent à ces champs. Comme `TransferFlow` est une table calculée statique, elle ne sera pas filtrée par FactTransfer. Il faut donc soit :
- utiliser **Option B** (visuel basé sur FactTransfer), soit
- ajouter dans `TransferFlow` des colonnes **TransferType** et **TransferDateKey** (et éventuellement **PlayerKey**) en les regroupant dans SUMMARIZE, ce qui change le grain (une ligne par départ/arrivée/type/date). Le visuel Sankey afficherait alors les flux agrégés ; les filtres s’appliqueraient sur cette table si tu ajoutes des segments (slicers) sur ces colonnes.

Pour rester simple **sans toucher au schéma des tables existantes** : utiliser **Option B** et un visuel Sankey qui accepte des champs de dimension + une mesure.

### Option B — Visuel basé sur FactTransfer (filtres natifs)

Si le visuel Sankey choisi accepte **deux champs de dimension** (Source / Target) et **une mesure** :

1. **Rôle « Source »** : club de départ. Comme FactTransfer a deux relations vers DimClub, il faut exposer le **nom** du club de départ. Créer une **colonne calculée** sur FactTransfer, par exemple  
   `DepartureClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], FactTransfer[DepartureClubKey])`.
2. **Rôle « Target »** : club d’arrivée, par ex.  
   `ArrivalClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], FactTransfer[ArrivalClubKey])`.
3. **Value** : mesure `Transfer Amount (ME) = SUM(FactTransfer[AmountME])`.

Base du visuel = **FactTransfer**. Source = `FactTransfer[DepartureClubName]`, Target = `FactTransfer[ArrivalClubName]`, Value = `[Transfer Amount (ME)]`. Les filtres **TransferType** et **TransferDateKey** (ou période via DimDate) s’appliquent directement au visuel.

*Remarque :* Cela ajoute deux colonnes calculées dans FactTransfer. Si tu ne veux aucune modification du modèle (y compris colonnes calculées), il faudra une table calculée comme en Option A et gérer les filtres par une table intermédiaire ou des segments sur cette table.

---

## 2. Configuration Power BI — Sankey Diagram By ChartExpo

ChartExpo utilise deux zones dans le panneau **Données** du visuel :
- **Category Data** : une ou plusieurs **catégories dans l’ordre**, qui définissent les **colonnes de nœuds** du Sankey (flux de la 1re vers la 2e, puis vers la 3e, etc.).
- **Measure Data** : la **mesure** qui donne l’épaisseur des flux (ex. « Sum of Count » dans l’exemple visiteurs ; pour les transferts = somme des montants).

**Configuration pour « qui achète à qui » :**

1. **Visuel :** Sankey Diagram By ChartExpo (AppSource).
2. **Source du visuel :** FactTransfer (Option B) ou TransferFlow (Option A).
3. **Category Data** (glisser dans l’ordre) :
   - **1re catégorie** : club de départ — `FactTransfer[DepartureClubName]` (Option B) ou `TransferFlow[DepartureClubName]` (Option A).
   - **2e catégorie** : club d’arrivée — `FactTransfer[ArrivalClubName]` ou `TransferFlow[ArrivalClubName]`.
   - *(Optionnel)* 3e catégorie : ex. `FactTransfer[TransferType]` pour un flux en 3 niveaux : Départ → Type (Purchase/Loan/Free) → Arrivée.
4. **Measure Data** : glisser la **mesure** `SUM(FactTransfer[AmountME])` (Option B) ou la colonne `TransferFlow[AmountME]` (Option A). C’est elle qui fixe l’épaisseur des rubans.
5. **Filtres (interactions) :**
   - **TransferType** : slicer ou filtre sur `FactTransfer[TransferType]`.
   - **Période** : slicer sur `FactTransfer[TransferDateKey]` ou sur `DimDate` (ex. année, trimestre) lié à TransferDateKey.
6. **Drill « top players » du flux sélectionné :**
   - Au clic sur un flux (paire Départ → Arrivée), afficher les joueurs les plus chers (ou les plus nombreux) sur ce flux. En Power BI : soit **drill-through** vers une page « Détail joueurs » avec filtres `DepartureClubKey` et `ArrivalClubKey` (passés par le clic), soit **visuel secondaire** (table ou graphique) sur la même page, filtré par le flux sélectionné (cross-filter depuis le Sankey si le visuel le supporte). Données : `DimPlayer[FullName]`, `FactTransfer[PlayerKey]`, `SUM(FactTransfer[AmountME])` par joueur, filtré par la paire (DepartureClubKey, ArrivalClubKey) correspondant au flux cliqué.

---

## 3. Récap faisabilité

- **Flux DepartureClub → ArrivalClub, épaisseur = SUM(AmountME) :** faisable avec **Sankey Diagram By ChartExpo** et soit deux colonnes calculées sur FactTransfer (Option B), soit une table calculée d’agrégation (Option A).
- **Filtres TransferType et période (TransferDateKey) :** naturels avec Option B ; avec Option A, à gérer via le grain de la table calculée ou des segments.
- **Drill « top players » du flux :** faisable par drill-through ou visuel secondaire filtré par la paire (DepartureClubKey, ArrivalClubKey), en utilisant FactTransfer et DimPlayer.

Si tu imposes **aucune modification du modèle** (aucune nouvelle table, aucune nouvelle colonne), le Sankey reste faisable en utilisant un visuel qui accepte deux champs de dimension issus de **tables associées** : il faudra alors que le visuel puisse utiliser, par exemple, deux champs dérivés de DimClub (un pour chaque rôle) via les relations FactTransfer → DimClub. Tous les visuels Sankey ne le permettent pas ; en pratique, Option B (deux colonnes calculées) ou Option A (une table calculée) est le plus fiable.
