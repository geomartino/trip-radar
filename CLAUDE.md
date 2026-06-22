# CLAUDE.md — Trip Radar

Fichier de contexte pour Claude Code / sessions futures.
Créé le 2026-06-21. Dernière mise à jour : 2026-06-22.

---

## 🗺️ Vue d'ensemble du projet

**Trip Radar** est une carte interactive mondiale des avertissements de voyage du gouvernement du Canada (Affaires mondiales Canada). Elle affiche les niveaux de risque par pays avec filtres interactifs, zoom/pan, et liens vers les pages détail de chaque destination.

**URL live :** https://geomartino.github.io/trip-radar/
**Repo :** https://github.com/geomartino/trip-radar
**Source des données :** flux JSON officiel `data.international.gc.ca/travel-voyage/index-alpha-fra.json` (liens de détail vers voyage.gc.ca)

---

## 📁 Structure du projet

```
trip-radar/
├── index.html                           # Page principale — carte interactive
├── data.json                            # Données générées automatiquement (ne pas éditer manuellement)
├── README.md                            # Documentation publique du repo
├── CLAUDE.md                            # Ce fichier — contexte pour Claude
└── .github/
    └── workflows/
        └── update-data-canada.yml       # GitHub Actions — mise à jour quotidienne (Canada)
```

---

## ⚙️ Architecture technique

### Flux de données
```
GitHub Actions (tourne 1x/jour à 4h UTC / minuit EST)
    → fetche le flux JSON officiel data.international.gc.ca/travel-voyage/index-alpha-fra.json
    → script Python (requests + json, pas de scraping HTML) parse le JSON directement
    → génère data.json avec pays + niveaux de risque + slugs URL
    → commit automatique dans le repo

Navigateur de l'utilisateur
    → charge index.html (GitHub Pages, statique)
    → fetch data.json + world-atlas TopoJSON (CDN) en parallèle
    → rendu de la carte via D3.js (projection Natural Earth)
    → colorie les pays selon les niveaux de risque
```

### Pourquoi cette architecture ?
- **100% gratuit pour toujours** — aucune API payante, aucune carte de crédit requise
- **Aucun secret à gérer** — zéro clé API, zéro surface d'attaque
- **Pas de backend** — tout est statique, hébergé gratuitement sur GitHub Pages
- **Données fraîches** — mise à jour automatique chaque nuit à minuit EST
- **Robuste** — si le workflow échoue, l'ancienne version de data.json reste en place
- **GitHub Actions gratuit** — 2000 minutes/mois pour les repos publics, largement suffisant

---

## 🔑 Secrets & config

Aucun secret requis pour le fonctionnement de base. Le workflow n'utilise aucune API externe payante.

- **`DISCORD_WEBHOOK_URL`** (optionnel) — si configuré (`gh secret set DISCORD_WEBHOOK_URL`), notifie un webhook Discord en cas d'échec du workflow de mise à jour des données. Nécessite un accès **admin** sur le repo pour être ajouté (pas seulement *write*) — voir Points d'attention.

---

## 🌍 Format de data.json

```json
{
  "updated": "YYYY-MM-DD",
  "source": "https://data.international.gc.ca/travel-voyage/index-alpha-fra.json",
  "countries": [
    {
      "name": "Nom du pays en français",
      "slug": "slug-url-voyage-gc-ca",
      "level": 1,
      "iso2": "CODE_ISO_2",
      "recentUpdate": "Texte court décrivant la dernière modification",
      "lastUpdateDate": "Date formatée en français (ex. 19 juin 2026 16:01 HAE)",
      "hasRegionalAdvisory": false,
      "hasAdvisoryWarning": false,
      "updateType": "Editorial change"
    }
  ]
}
```

`recentUpdate`, `lastUpdateDate`, `hasRegionalAdvisory`, `hasAdvisoryWarning` et `updateType` viennent directement du flux JSON officiel (`fra.recent-updates`, `fra.friendly-date`, `has-regional-advisory`, `has-advisory-warning`, `recent-updates-type`) — alimentent le panneau d'info slide-in de la carte. `hasAdvisoryWarning` est un signal distinct de `hasRegionalAdvisory` (avis spécial type élections/sécurité, pas une variation de niveau par région — seulement ~9 des 230 pays ont les deux actifs). `updateType` n'est pas traduit par le flux (toujours en anglais : `Editorial change` / `Regular text update` / `Full TAA review`) — traduit côté frontend dans `index.html` (`UPDATE_TYPE_LABELS`). Pas de texte explicatif du niveau de risque ("pourquoi") dans ce flux — uniquement disponible via les pages détail HTML, volontairement non scrapées pour l'instant (~230 requêtes supplémentaires, voir discussion dans l'historique du projet).

### Niveaux de risque

| Level | Label | Couleur CSS |
|-------|-------|-------------|
| 0 | Aucune donnée (non listé) | `--level-0: #1a2a38` |
| 1 | Prenez des mesures de sécurité normales | `--level-1: #2a6e4e` (vert) |
| 2 | Faites preuve d'une grande prudence | `--level-2: #b8860b` (jaune) |
| 3 | Évitez tout voyage non essentiel | `--level-3: #c05a1f` (orange) |
| 4 | Évitez tout voyage | `--level-4: #8b1a1a` (rouge) |

### Pattern URL des pages détail
```
https://voyage.gc.ca/destinations/{slug}
ex: https://voyage.gc.ca/destinations/colombie
```

---

## 🗺️ Carte — D3.js + TopoJSON

- Rendu via **D3.js v7** (CDN jsdelivr) + **topojson-client v3**
- Données géographiques : **world-atlas@2** (`countries-110m.json`) via unpkg CDN
- Projection : **Natural Earth** (`d3.geoNaturalEarth1`) — viewBox `0 0 2000 1001`
- Fond océan + lignes de grille (graticule) inclus
- Les pays sont des `<path>` dans un `<g>` wrapper avec les attributs :
  - `data-iso2` : code ISO 3166-1 alpha-2
  - `data-level` : niveau de risque (0–4)
  - `data-name` : nom en français
  - `data-slug` : slug URL voyage.gc.ca
- **Zoom/Pan** via `d3.zoom()` — molette, clic-glisser, boutons +/−/⊙
- Mapping codes numériques ISO → ISO2 dans `NUM_TO_ISO2` (dans `index.html`)
- **Noms de pays permanents** — bouton "Aa" dans les contrôles de la carte, bascule la classe `.labels-visible` sur `#map-svg` (labels positionnés au centroïde de chaque pays, groupe `#country-labels` rendu par-dessus tous les pays). Préférence persistée dans `localStorage` (`tripradar-labels-visible`).

---

## 🪪 Panneau d'info (slide-in)

Cliquer sur un pays avec données ouvre `#info-panel` (slide-in depuis la droite) au lieu d'ouvrir directement voyage.gc.ca :

- Nom, niveau de risque coloré, badge orange si `hasRegionalAdvisory`, badge bleu si `hasAdvisoryWarning` (signal distinct — avis spécial type élections/sécurité, pas une variation régionale)
- **Carte régionale officielle** — image PNG du gouvernement à `international.gc.ca/tama-sgcv_images/maps-cartes/{iso2}/mapfra.png` (chemin non documenté/déduit par inspection, donc masquée proprement via `onerror` si elle casse un jour). Cliquable → lightbox plein écran (`#map-lightbox`)
- Section "dernière mise à jour" (`recentUpdate` + `lastUpdateDate`) avec le type de changement traduit (`updateType` via `UPDATE_TYPE_LABELS`)
- Bouton vers voyage.gc.ca pour les détails complets + bouton désactivé "Planifier mon voyage (à venir)" en prévision de l'intégration avec trip-planner
- Fermeture par ✕, clic extérieur, ou Échap (gère aussi la fermeture du lightbox en priorité si les deux sont ouverts)

---

## 🎨 Design system

### Palette (thème sombre)
```css
--bg: #0f1923          /* fond principal */
--surface: #162030     /* header, filtres */
--border: #1e2f42      /* bordures */
--text: #e8edf2        /* texte principal */
--text-muted: #6b8299  /* texte secondaire */
--accent: #3d8ef0      /* bleu accent (logo, filtre actif) */
```

### Stack technique
- **HTML/CSS/JS vanilla** — aucun framework
- **D3.js v7** (CDN) — rendu carte et zoom/pan
- **topojson-client v3** (CDN) — parsing des données géographiques
- **world-atlas@2** (CDN unpkg) — géométries Natural Earth 110m
- **Fetch API** — pour charger data.json
- **GitHub Pages** — hébergement statique gratuit
- **GitHub Actions + Python (requests)** — pipeline de mise à jour des données, zéro coût, source officielle (data.international.gc.ca)

---

## 🔧 Fonctionnalités actuelles

- [x] Carte mondiale précise (Natural Earth) colorée par niveau de risque
- [x] Zoom molette + pan clic-glisser + boutons +/−/⊙
- [x] Tooltip au survol (nom du pays + niveau)
- [x] Noms de pays affichables en permanence (bouton "Aa", persisté en localStorage)
- [x] Clic → panneau d'info slide-in (niveau, badges régional/avis important, carte régionale officielle + lightbox, dernière mise à jour) avec lien vers la page détail complète sur voyage.gc.ca
- [x] Filtres par niveau (Tous / Niveau 1 / 2 / 3 / 4) avec compteurs
- [x] Effet "dimming" des pays non sélectionnés lors d'un filtre
- [x] Mise à jour automatique quotidienne via GitHub Actions (4h UTC), source JSON officielle
- [x] Détection de casse du pipeline (chute suspecte du nombre de pays) → issue GitHub auto-créée/fermée, notification Discord optionnelle
- [x] Date de dernière mise à jour affichée dans le header

---

## 🚀 Milestones & issues GitHub

### Milestone 1 — Sources de données internationales
https://github.com/geomartino/trip-radar/milestone/1

- **Issue #1** — Source USA (US State Department / travel.state.gov)
- **Issue #2** — Source UK (FCDO / gov.uk/foreign-travel-advice)

Architecture prévue : un workflow `update-data-{pays}.yml` par source, un `data-{pays}.json` par source, sélecteur de source dans le front-end. Les niveaux seront normalisés sur l'échelle 1-4 commune.

### Milestone 2 — Intégration prix de billets d'avion
https://github.com/geomartino/trip-radar/milestone/2

Trois options documentées, par ordre de complexité :

- **Issue #3 — Option A** : lien Google Flights dans le tooltip (sans API, ~15 min)
  - URL : `https://www.google.com/travel/flights?q=vols+vers+{nom_pays}`
  - Zéro backend, zéro coût, implémentation triviale

- **Issue #4 — Option B** : prix réels via API Amadeus côté client
  - API gratuite (2000 req/mois), clé visible dans le code
  - Nécessite un mapping `iso2 → code IATA aéroport` à construire
  - Auth OAuth2 Amadeus (token Bearer, expire 30 min)

- **Issue #5 — Option C** : Amadeus + fonction serverless (Cloudflare Workers / Netlify)
  - Clé API sécurisée côté serveur, cache centralisé
  - Plus complexe mais plus robuste pour usage public

**Recommandation** : commencer par l'Option A, puis évaluer si les vrais prix justifient l'Option B ou C.

---

## 🐛 Points d'attention

1. **Source officielle, donc plus stable** — depuis le remplacement du scraping HTML par le flux JSON `data.international.gc.ca/travel-voyage/index-alpha-fra.json` (jeu de données *open data*, destiné à la consommation automatisée), le risque de blocage d'IP GitHub Actions est beaucoup plus faible qu'avec le scraping de la page rendue.
2. **Le format du flux JSON peut quand même changer** — le workflow détecte une casse totale (0 pays) ou partielle (chute de plus de 30% du nombre de pays vs la veille) et échoue plutôt que de committer des données corrompues. En cas d'échec, une issue GitHub est créée/mise à jour automatiquement (et un webhook Discord notifié si `DISCORD_WEBHOOK_URL` est configuré).
3. **Les slugs URL** viennent directement du champ `fra.url-slug` du flux JSON — fiables, plus de parsing de lien HTML.
4. **data.json ne doit pas être édité manuellement** — il est écrasé à chaque run du workflow.
5. **Territoires sans polygone propre** — le flux JSON inclut quelques entrées avec un code non standard (ex. Açores = `PT-20`) qui n'ont pas de polygone distinct dans world-atlas ; elles sont filtrées (`len(iso2) != 2`) et n'apparaissent pas dans `data.json`. Le `country-iso` du flux officiel élimine le besoin d'un mapping nom→ISO2 codé en dur (et corrige au passage deux pays mal résolus par l'ancien `ISO2_MAP` : Saint-Vincent-et-Grenadines, et la collision Espagne/Îles Canaries qui partageaient `ES`).

---

## 📦 Famille de projets Trip*

| Repo | Description |
|------|-------------|
| trip-planner | Planificateur de voyage |
| trip-countdown | Compte à rebours avant le départ |
| **trip-radar** | Carte des avertissements de voyage ← ce projet |

**Propriétaire GitHub :** geomartino

---

## 💬 Contexte de développement

Ce projet a été initié dans une session Claude.ai (claude-sonnet-4-6) le 2026-06-21.
Les décisions d'architecture ont été prises pour maximiser la gratuité (aucune API payante), la sécurité (aucune clé exposée), la simplicité (site 100% statique) et la maintenabilité (pipeline automatisé en Python, basé sur le flux JSON officiel du gouvernement depuis le 2026-06-22, auparavant du scraping HTML avec BeautifulSoup).

La carte a migré de polygones SVG simplifiés faits main vers D3.js + Natural Earth TopoJSON pour une qualité géographique professionnelle.
