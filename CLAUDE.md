# CLAUDE.md — Trip Radar

Fichier de contexte pour Claude Code / sessions futures.
Créé le 2026-06-21.

---

## 🗺️ Vue d'ensemble du projet

**Trip Radar** est une carte interactive mondiale des avertissements de voyage du gouvernement du Canada (Affaires mondiales Canada). Elle affiche les niveaux de risque par pays avec filtres interactifs et liens vers les pages détail de chaque destination.

**URL live :** https://geomartino.github.io/trip-radar/
**Repo :** https://github.com/geomartino/trip-radar
**Source des données :** https://voyage.gc.ca/voyager/avertissements

---

## 📁 Structure du projet

```
trip-radar/
├── index.html                        # Page principale — carte interactive
├── data.json                         # Données générées automatiquement (ne pas éditer manuellement)
├── README.md                         # Documentation publique du repo
├── CLAUDE.md                         # Ce fichier — contexte pour Claude
└── .github/
    └── workflows/
        └── update-data.yml           # GitHub Actions — mise à jour quotidienne des données
```

---

## ⚙️ Architecture technique

### Flux de données
```
GitHub Actions (tourne 1x/jour à 6h UTC)
    → fetche voyage.gc.ca/voyager/avertissements (HTML brut)
    → script Python (BeautifulSoup) parse le HTML directement
    → génère data.json avec pays + niveaux de risque + slugs URL
    → commit automatique dans le repo

Navigateur de l'utilisateur
    → charge index.html (GitHub Pages, statique)
    → fetch data.json (fichier statique, ultra rapide)
    → colorie la carte SVG selon les niveaux
```

### Pourquoi cette architecture ?
- **100% gratuit pour toujours** — aucune API payante, aucune carte de crédit requise
- **Aucun secret à gérer** — plus de clé Anthropic, zéro surface d'attaque
- **Pas de backend** — tout est statique, hébergé gratuitement sur GitHub Pages
- **Données fraîches** — mise à jour automatique chaque matin
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

## 🗺️ Carte SVG

- La carte est un SVG inline dans `index.html`
- Les pays sont représentés par des `<path>` avec les attributs :
  - `data-iso2` : code ISO 3166-1 alpha-2
  - `data-level` : niveau de risque (0–4)
  - `data-name` : nom en français
  - `data-slug` : slug URL voyage.gc.ca
- Les chemins SVG (`MAP_PATHS`) sont des polygones simplifiés (Natural Earth)
- Le viewBox est `0 0 2000 1001`

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
- **HTML/CSS/JS vanilla** — aucun framework, aucune dépendance
- **SVG inline** — pas de librairie de carte externe
- **Fetch API** — pour charger data.json
- **GitHub Pages** — hébergement statique gratuit
- **GitHub Actions + Python/BeautifulSoup** — pipeline de mise à jour des données, zéro coût

---

## 🔧 Fonctionnalités actuelles

- [x] Carte mondiale colorée par niveau de risque
- [x] Tooltip au survol (nom du pays + niveau)
- [x] Clic → ouvre la page détail sur voyage.gc.ca
- [x] Filtres par niveau (Tous / Niveau 1 / 2 / 3 / 4) avec compteurs
- [x] Effet "dimming" des pays non sélectionnés lors d'un filtre
- [x] Mise à jour automatique quotidienne via GitHub Actions
- [x] Date de dernière mise à jour affichée dans le header

---

## 🚀 Évolutions prévues / idées futures

- [ ] Ajouter d'autres sources (US State Department, UK FCDO, etc.)
- [ ] Barre de recherche par pays
- [ ] Panneau latéral avec liste des pays filtrés
- [ ] Mode bilingue FR/EN
- [ ] Historique des changements de niveau (suivi dans le temps)
- [ ] Notifications si un pays change de niveau
- [ ] Améliorer la précision des chemins SVG (actuellement très simplifiés)

---

## 📦 Famille de projets Trip*

| Repo | Description |
|------|-------------|
| trip-planner | Planificateur de voyage |
| trip-countdown | Compte à rebours avant le départ |
| **trip-radar** | Carte des avertissements de voyage ← ce projet |

**Propriétaire GitHub :** geomartino

---

## 🐛 Points d'attention

1. **Les chemins SVG sont très simplifiés** — certains petits pays peuvent être imprécis ou manquants. Amélioration possible avec TopoJSON/D3.
2. **Le parsing HTML de voyage.gc.ca peut casser** si le gouvernement change la structure de leur page — surveiller les échecs du workflow GitHub Actions et ajuster le sélecteur BeautifulSoup en conséquence.
3. **Les slugs URL** sont extraits directement des liens `<a href>` sur la page — ils devraient être fiables, mais vérifier manuellement si un pays ne redirige pas correctement.
4. **data.json ne doit pas être édité manuellement** — il est écrasé à chaque run du workflow.

---

## 💬 Contexte de développement

Ce projet a été initié dans une session Claude.ai (claude-sonnet-4-6) le 2026-06-21.
Les décisions d'architecture ont été prises pour maximiser la gratuité (aucune API payante), la sécurité (aucune clé exposée), la simplicité (site 100% statique) et la maintenabilité (pipeline automatisé avec Python/BeautifulSoup).
