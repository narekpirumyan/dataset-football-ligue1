# Guide — Logos des clubs

**Objectif :** Utiliser les logos des clubs (URLs dans `club_logos.json`) dans le rapport Power BI, notamment dans un **visuel HTML Content**.

**Fichier source :** `Dashboard Design/club_logos.json` — tableau avec `club_name` et `logo_url` (une URL par club, ex. Wikimedia Commons).

---

## 1. Intégration des logos dans le modèle

1. **Charger le JSON dans Power BI**  
   - Power Query : **Obtenir des données → Fichier → À partir de JSON**.  
   - Choisir **`club_logos.json`**.  
   - Le tableau chargé doit contenir au moins : **`club_name`**, **`logo_url`**.  
   - Renommer les colonnes si besoin pour correspondre à ton modèle (ex. `club_name` → **ClubName** si tu utilises ce libellé partout).

2. **Liaison avec la dimension club**  
   - Créer une **relation** entre la table des logos et ta dimension club (ex. **DimClub**) sur le nom du club (ex. **ClubName** / **club_name**).  
   - La relation permet d’obtenir l’URL du logo pour le club sélectionné (slicer, filtre, contexte du visuel).

3. **Synchronisation avec la liste des clubs**  
   - Garder **`club_logos.json`** aligné avec la liste des clubs (ex. `club_colors.json` ou `Dataset/clubs.xlsx`) : mêmes libellés **club_name** (casse et orthographe).  
   - En ajoutant ou renommant un club, mettre à jour le JSON et rafraîchir le rapport.

---

## 2. Affichage du logo dans un visuel HTML Content

Pour afficher le logo du club **sélectionné** (slicer, filtre, etc.) dans un visuel HTML :

1. **Créer une mesure DAX** qui renvoie du HTML contenant une balise `<img>` avec l’URL du logo.  
   - Récupérer l’URL via **LOOKUPVALUE** (ou **RELATED** si le contexte est une ligne de la dimension club) à partir de la table des logos.  
   - Adapter les noms de tables/colonnes à ton modèle.

**Exemple de mesure (à adapter aux noms de tes tables/colonnes) :**

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
- **Cohérence des noms :** Les valeurs **club_name** dans `club_logos.json` doivent correspondre exactement à celles utilisées dans la dimension club (et dans `club_colors.json`) pour que la relation et LOOKUPVALUE fonctionnent.

---

## 4. Résumé

| Étape | Action |
|-------|--------|
| 1 | Charger `club_logos.json` en requête Power Query. |
| 2 | Créer la relation table des logos ↔ dimension club (sur le nom du club). |
| 3 | Créer la mesure DAX qui renvoie `<img src='...' />` avec l’URL du club courant. |
| 4 | Utiliser cette mesure dans un visuel **HTML Content**. |
