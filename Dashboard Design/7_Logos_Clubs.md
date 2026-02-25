# Guide — Logos des clubs

**Objectif :** Utiliser les logos des clubs (URLs dans `club_logos.json`) dans le rapport Power BI, notamment dans un **visuel HTML Content**.

**Référence :** Ce guide est aligné avec **`Docs/0_POWER_BI_DASHBOARD_SPECIFICATION.md`** (noms des clubs, cohérence cross-fichiers). Les libellés de clubs doivent être identiques à ceux utilisés dans les données (match_results, player_stats, transfers, etc.) — ex. « Paris SG », « Olympique Lyonnais ».

**Fichier source :** `Dashboard Design/club_logos.json` — tableau avec **`club_name`** et **`logo_url`** (une URL par club, ex. Wikimedia Commons). Les valeurs **`club_name`** correspondent à celles de **`club_colors.json`** et à la liste des clubs du jeu de données (cf. spécification).

---

## 1. Intégration des logos dans le modèle

### Option A — Colonne URL dans DimClub (recommandé)

Si tu as **ajouté une colonne d’URL des logos directement dans la table DimClub** (par ex. via Merge avec `club_logos.json` en Power Query sur `club_name` = `ClubName`, puis gardage de la colonne `logo_url` dans DimClub) :

1. **Vérifier** que la colonne (ex. **LogoUrl** ou **logo_url**) contient bien une URL par ligne et qu’aucun club n’a d’URL vide si tu veux afficher son logo.
2. **Aucune relation supplémentaire** n’est nécessaire : l’URL est déjà dans DimClub.
3. **Étape suivante :** passer directement à la [section 2](#2-affichage-du-logo-dans-un-visuel-html-content) pour créer la mesure DAX et l’utiliser dans un visuel HTML Content.

### Option B — Table des logos séparée (ClubLogos)

1. **Charger le JSON dans Power BI**  
   - Power Query : **Obtenir des données → Fichier → À partir de JSON**.  
   - Choisir **`Dashboard Design/club_logos.json`**.  
   - Le tableau chargé contient **`club_name`** et **`logo_url`**.  
   - Renommer **`club_name`** en **ClubName** (ou en le nom de la colonne club utilisé dans ton modèle) pour rester cohérent avec la dimension club et avec **`club_colors.json`** (cf. Visual Charter).

2. **Liaison avec la dimension club**  
   - Créer une **relation** entre la table des logos (ex. **ClubLogos**) et **DimClub** sur le nom du club (ex. **ClubName** ↔ **club_name**).  
   - La relation permet d’obtenir l’URL du logo pour le club sélectionné (slicer, filtre, contexte du visuel). Les noms de clubs doivent être identiques à ceux des fichiers sources (cf. spécification, section « Cross-file checks »).

3. **Synchronisation avec la liste des clubs**  
   - Garder **`club_logos.json`** aligné avec **`club_colors.json`** et avec la liste des clubs du jeu de données (match_results : `home_team` / `away_team`, player_stats : `club`, etc.) : mêmes libellés **club_name** (casse et orthographe).  
   - En ajoutant ou renommant un club, mettre à jour **`club_logos.json`** (et **`club_colors.json`** si besoin), puis rafraîchir le rapport.

---

## 2. Affichage du logo dans un visuel HTML Content

Pour afficher le logo du club dans un visuel HTML, la mesure DAX dépend du **contexte de filtre** (cf. **`Docs/0_POWER_BI_DASHBOARD_SPECIFICATION.md`** : données joueur dans `player_stats_season.csv` avec `full_name`, `club` ; modèle : **DimPlayer** (FullName), **FactPlayerSeason** (PlayerKey, ClubKey), **DimClub**).

### Contexte : filtre sur le **joueur** (FullName)

C’est le cas le plus courant en **vue joueur** : le slicer ou le filtre porte sur **DimPlayer[FullName]** (nom du joueur). Le club du joueur est alors obtenu via **FactPlayerSeason** (une ligne joueur = un ClubKey pour la saison), puis l’URL du logo depuis **DimClub** (cf. **`Docs/1_POWER_BI_STAR_SCHEMA_DESIGN.md`** : FactPlayerSeason[PlayerKey] → DimPlayer, FactPlayerSeason[ClubKey] → DimClub).

**Mesure à utiliser** (URL des logos dans DimClub, colonne ex. **LogoUrl**) :

```dax
Logo Club HTML =
VAR ClubKey = SELECTEDVALUE( FactPlayerSeason[ClubKey] )
VAR UrlLogo = LOOKUPVALUE( DimClub[LogoUrl], DimClub[ClubKey], ClubKey )
RETURN
    IF(
        NOT ISBLANK( UrlLogo ),
        "<img src='" & UrlLogo & "' alt='Logo' style='max-width:80px; height:auto;' />",
        ""
    )
```

- Quand un **seul joueur** est sélectionné (slicer sur **FullName**), le filtre remonte à **FactPlayerSeason** via **PlayerKey** : `FactPlayerSeason[ClubKey]` donne le club de ce joueur pour la saison, puis **LOOKUPVALUE** récupère **DimClub[LogoUrl]** pour ce **ClubKey**.
- Remplace **LogoUrl** par le nom exact de ta colonne dans DimClub (ex. `logo_url`) si besoin.

### Contexte : filtre sur le **club** (ClubName)

Si le filtre porte directement sur le **club** (slicer sur **DimClub[ClubName]**), l’URL est dans DimClub :

```dax
Logo Club HTML =
VAR UrlLogo = SELECTEDVALUE( DimClub[LogoUrl] )
RETURN
    IF(
        NOT ISBLANK( UrlLogo ),
        "<img src='" & UrlLogo & "' alt='Logo' style='max-width:80px; height:auto;' />",
        ""
    )
```

### Si tu utilises une table des logos séparée (option B — ClubLogos)

Avec filtre sur le **joueur** (FullName), récupérer d’abord le club du joueur via FactPlayerSeason, puis l’URL depuis la table des logos (colonne ex. **club_name**, **logo_url**) :

```dax
Logo Club HTML =
VAR ClubKey = SELECTEDVALUE( FactPlayerSeason[ClubKey] )
VAR NomClub = LOOKUPVALUE( DimClub[ClubName], DimClub[ClubKey], ClubKey )
VAR UrlLogo = LOOKUPVALUE( ClubLogos[logo_url], ClubLogos[club_name], NomClub )
RETURN
    IF(
        NOT ISBLANK( UrlLogo ),
        "<img src='" & UrlLogo & "' alt='Logo' style='max-width:80px; height:auto;' />",
        ""
    )
```

Avec filtre sur le **club** : `VAR NomClub = SELECTEDVALUE( DimClub[ClubName] )` puis `LOOKUPVALUE( ClubLogos[logo_url], ClubLogos[club_name], NomClub )` comme précédemment.

2. **Utiliser la mesure dans le visuel**  
   - Insérer un visuel **HTML Content** (visuel personnalisé si nécessaire).  
   - Placer la mesure **Logo Club HTML** dans le champ du visuel.  
   - Le visuel affichera le logo du club correspondant au filtre/slicer actif.

3. **Personnalisation du rendu**  
   - Modifier le **style** dans la chaîne HTML (ex. `max-width`, `height`, `border-radius`, `object-fit`) pour adapter la taille et l’apparence du logo.  
   - Pour un bloc plus élaboré (logo + texte), construire toute la chaîne HTML dans la mesure (ex. `<div>…<img />…</div>`).

---

## 3. Remarques

- **URLs manquantes ou invalides :** Certaines URLs dans `club_logos.json` peuvent être obsolètes (404). En cas d’image non affichée, remplacer l’URL par celle de l’image sur [Wikimedia Commons](https://commons.wikimedia.org) ou [fr.wikipedia](https://fr.wikipedia.org) (clic droit sur l’image → « Copier l’adresse de l’image »), puis mettre à jour le JSON et rafraîchir.
- **Sécurité / réseau :** Les images sont chargées depuis les URLs au moment de l’affichage. Vérifier que l’environnement Power BI (Desktop / Service) peut accéder à ces domaines (ex. upload.wikimedia.org).
- **Cohérence des noms :** Les valeurs **club_name** dans `club_logos.json` doivent correspondre exactement à celles utilisées dans la dimension club, dans **`club_colors.json`** et dans les données sources (cf. **`Docs/0_POWER_BI_DASHBOARD_SPECIFICATION.md`** — champs `club`, `home_team`, `away_team`). Une seule orthographe par club dans tout le rapport (ex. « Paris SG », « Olympique Lyonnais »).

---

## 4. Résumé

La table des logos (ou la colonne URL dans DimClub) s’intègre au modèle Power BI décrit dans **`Docs/0_POWER_BI_DASHBOARD_SPECIFICATION.md`** et **`Docs/1_POWER_BI_STAR_SCHEMA_DESIGN.md`** (DimClub, ClubName). Les noms de clubs dans `club_logos.json` doivent être ceux utilisés dans les fichiers sources (match_results, player_stats, transfers, etc.).

**Filtre sur le joueur (FullName)** — URL dans DimClub (option A) :

| Étape | Action |
|-------|--------|
| 1 | Colonne URL déjà dans DimClub (ex. **LogoUrl**). |
| 2 | Créer la mesure DAX : `ClubKey = SELECTEDVALUE(FactPlayerSeason[ClubKey])`, puis `LOOKUPVALUE(DimClub[LogoUrl], DimClub[ClubKey], ClubKey)` pour l’URL. |
| 3 | Utiliser cette mesure dans un visuel **HTML Content** ; le slicer/filtre sur **DimPlayer[FullName]** affichera le logo du **club du joueur** sélectionné. |

**Filtre sur le club (ClubName)** — URL dans DimClub : mesure avec `SELECTEDVALUE(DimClub[LogoUrl])` uniquement.

**Quand tu utilises une table des logos séparée (option B) :**

| Étape | Action |
|-------|--------|
| 1 | Charger `club_logos.json` en requête Power Query. |
| 2 | Créer la relation table des logos ↔ DimClub sur le nom du club (ClubName). |
| 3 | Créer la mesure DAX qui renvoie `<img src='...' />` avec LOOKUPVALUE sur la table des logos. |
| 4 | Utiliser cette mesure dans un visuel **HTML Content**. |
