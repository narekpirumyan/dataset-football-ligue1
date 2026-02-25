# Guide — Logos des clubs

**Objectif :** Utiliser les logos des clubs (URLs dans `club_logos.json`) dans le rapport Power BI, notamment dans un **visuel HTML Content**.

**Référence :** Ce guide est aligné avec **`Docs/0_POWER_BI_DASHBOARD_SPECIFICATION.md`** (noms des clubs, cohérence cross-fichiers). Les libellés de clubs doivent être identiques à ceux utilisés dans les données (match_results, player_stats, transfers, etc.) — ex. « Paris SG », « Olympique Lyonnais ».

**Fichier source :** `Dashboard Design/club_logos.json` — tableau avec **`club_name`** et **`logo_url`** (une URL par club, ex. Wikimedia Commons). Les valeurs **`club_name`** correspondent à celles de **`club_colors.json`** et à la liste des clubs du jeu de données (cf. spécification).

---

## 1. Intégration des logos dans le modèle

1. **Charger le JSON dans Power BI**  
   - Power Query : **Obtenir des données → Fichier → À partir de JSON**.  
   - Choisir **`Dashboard Design/club_logos.json`**.  
   - Le tableau chargé contient **`club_name`** et **`logo_url`**.  
   - Renommer **`club_name`** en **ClubName** (ou en le nom de la colonne club utilisé dans ton modèle) pour rester cohérent avec la dimension club et avec **`club_colors.json`** (cf. Visual Charter).

2. **Liaison avec la dimension club**  
   - Créer une **relation** entre la table des logos (ex. **ClubLogos**) et la **dimension club** du modèle (ex. **DimClub** ou table équivalente dérivée des données) sur le **nom du club** (ex. **ClubName**).  
   - La relation permet d’obtenir l’URL du logo pour le club sélectionné (slicer, filtre, contexte du visuel). Les noms de clubs doivent être identiques à ceux des fichiers sources (cf. spécification, section « Cross-file checks »).

3. **Synchronisation avec la liste des clubs**  
   - Garder **`club_logos.json`** aligné avec **`club_colors.json`** et avec la liste des clubs du jeu de données (match_results : `home_team` / `away_team`, player_stats : `club`, etc.) : mêmes libellés **club_name** (casse et orthographe).  
   - En ajoutant ou renommant un club, mettre à jour **`club_logos.json`** (et **`club_colors.json`** si besoin), puis rafraîchir le rapport.

---

## 2. Affichage du logo dans un visuel HTML Content

Pour afficher le logo du club **sélectionné** (slicer, filtre, etc.) dans un visuel HTML :

1. **Créer une mesure DAX** qui renvoie du HTML contenant une balise `<img>` avec l’URL du logo.  
   - Récupérer l’URL via **LOOKUPVALUE** (ou **RELATED** si le contexte est une ligne de la dimension club) à partir de la table des logos.  
   - Adapter les noms de tables/colonnes à ton modèle.

**Exemple de mesure DAX** (adapter les noms de tables/colonnes à ton modèle ; ici : dimension club **DimClub** avec colonne **ClubName**, table des logos **ClubLogos** avec **club_name** et **logo_url**) :

```dax
Logo Club HTML =
VAR NomClub = SELECTEDVALUE( DimClub[ClubName] )
VAR UrlLogo  = LOOKUPVALUE( ClubLogos[logo_url], ClubLogos[club_name], NomClub )
RETURN
    IF(
        NOT ISBLANK( UrlLogo ),
        "<img src='" & UrlLogo & "' alt='Logo' style='max-width:80px; height:auto;' />",
        ""
    )
```

*Si ta dimension club ou ta table de logos ont d’autres noms (ex. colonne `club` au lieu de `ClubName`), remplacer dans la mesure en conséquence.*

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

La table des logos s’intègre au modèle Power BI décrit dans **`Docs/0_POWER_BI_DASHBOARD_SPECIFICATION.md`** (section « Next steps » : modèle avec dimension club) et **`Docs/1_POWER_BI_STAR_SCHEMA_DESIGN.md`** (DimClub, ClubName). Les noms de clubs dans `club_logos.json` doivent être ceux utilisés dans les fichiers sources listés dans la spécification (match_results, player_stats, transfers, etc.).

| Étape | Action |
|-------|--------|
| 1 | Charger `club_logos.json` en requête Power Query. |
| 2 | Créer la relation table des logos ↔ dimension club (DimClub) sur le nom du club (ClubName). |
| 3 | Créer la mesure DAX qui renvoie `<img src='...' />` avec l’URL du club courant. |
| 4 | Utiliser cette mesure dans un visuel **HTML Content**. |
