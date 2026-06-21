# CLAUDE.md — Trip Radar

Fichier de contexte pour Claude Code / sessions futures.
Créé le 2026-06-21. Dernière mise à jour : 2026-06-21.

---

## 🗺️ Vue d'ensemble du projet

**Trip Radar** est une carte interactive mondiale des avertissements de voyage du gouvernement du Canada (Affaires mondiales Canada). Elle affiche les niveaux de risque par pays avec filtres interactifs, zoom/pan, et liens vers les pages détail de chaque destination.

**URL live :** https://geomartino.github.io/trip-radar/
**Repo :** https://github.com/geomartino/trip-radar
**Source des données :** https://voyage.gc.ca/voyager/avertissements

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
    → fetche voyage.gc.ca/voyager/avertissements (HTML brut)
    → script Python (BeautifulSoup) parse le HTML directement
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

Aucun secret requis. Le workflow n'utilise aucune API externe payante.

---

## 🌍 Format de data.json

```json
{
  "updated": "YYYY-MM-DD",
  "source": "https://voyage.gc.ca/voyager/avertissements",
  "countries": [
    {
      "name": "Nom du pays en français",
      "slug": "slug-url-voyage-gc-ca",
      "level": 1,
      "iso2": "CODE_ISO_2"
    }
  ]
}
```

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
- **GitHub Actions + Python/BeautifulSoup** — pipeline de mise à jour des données, zéro coût

---

## 🔧 Fonctionnalités actuelles

- [x] Carte mondiale précise (Natural Earth) colorée par niveau de risque
- [x] Zoom molette + pan clic-glisser + boutons +/−/⊙
- [x] Tooltip au survol (nom du pays + niveau)
- [x] Clic → ouvre la page détail sur voyage.gc.ca
- [x] Filtres par niveau (Tous / Niveau 1 / 2 / 3 / 4) avec compteurs
- [x] Effet "dimming" des pays non sélectionnés lors d'un filtre
- [x] Mise à jour automatique quotidienne via GitHub Actions (4h UTC)
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

1. **voyage.gc.ca peut bloquer les IPs GitHub Actions** — le workflow essaie d'abord la page FR, puis la page EN (`travel.gc.ca`) en fallback, avec 3 tentatives chacune.
2. **Le parsing HTML peut casser** si le gouvernement change la structure de leur page — surveiller les échecs du workflow et ajuster le sélecteur BeautifulSoup (images SVG avec noms `normal-precautions`, `increased-caution`, `reconsider-travel`, `do-not-travel`).
3. **Les slugs URL** sont extraits directement des liens `<a href="/destinations/...">` — fiables mais à vérifier si un pays ne redirige pas.
4. **data.json ne doit pas être édité manuellement** — il est écrasé à chaque run du workflow.
5. **Pays sans iso2** — si un nouveau pays apparaît dans les données sans match dans `ISO2_MAP` (workflow) ou `NUM_TO_ISO2` (index.html), il s'affichera gris sur la carte. Le workflow logge un avertissement `⚠️` dans ce cas.

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
Les décisions d'architecture ont été prises pour maximiser la gratuité (aucune API payante), la sécurité (aucune clé exposée), la simplicité (site 100% statique) et la maintenabilité (pipeline automatisé avec Python/BeautifulSoup).

La carte a migré de polygones SVG simplifiés faits main vers D3.js + Natural Earth TopoJSON pour une qualité géographique professionnelle.
