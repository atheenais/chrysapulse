# ChrysaPulse — Dashboard « Prenons le pouls »

Tableau de bord interne des résultats de l'enquête **NPS Collaborateurs « Prenons le pouls »**
de [Chrysalides](https://chrysalides.me). Vue fusionnée internes + freelances, isolable d'un clic,
avec historisation des chargements dans le temps et visualisation de l'évolution (eNPS, scores,
timeline datée).

Public : direction / RH de Chrysalides.

## Application

Une **page HTML autonome** : [`index.html`](index.html) — HTML + CSS + JS inline, données seed
embarquées. Aucun framework, aucun build, aucune dépendance npm.

Dépendances réseau **optionnelles** (l'app fonctionne sans) :

- Google Fonts (Fraunces + Inter) — fallback `serif`/`sans-serif` hors ligne ;
- SheetJS via CDN — chargé **à la demande** uniquement pour l'import `.xlsx` (le `.csv` marche hors ligne).

## Historique des chargements

L'historique des vagues d'enquête vit à deux endroits :

- **`data/history.json`** — archive committée dans le repo, source de vérité partagée entre
  appareils. C'est exactement la sortie du bouton « Exporter l'historique » du dashboard.
- **`localStorage`** du navigateur — état de travail local.

Quand le dashboard est **servi en http(s)** (GitHub Pages, serveur local), il charge
`data/history.json` au démarrage et le fusionne avec le local. En `file://`, il retombe sur le
seed embarqué + localStorage (le `fetch` étant bloqué par le navigateur).

### Ajouter une vague de résultats

1. Dans le dashboard, **importer** l'export Microsoft Forms (`.xlsx`/`.csv`), renseigner date + libellé.
2. Cliquer **« Exporter l'historique (.json) »** → télécharge `history.json`.
3. Remplacer `data/history.json` par ce fichier, **commit + push**.
4. Au prochain rafraîchissement, tous les appareils voient la nouvelle vague.

## Lancer en local

```bash
python3 -m http.server 8000
# http://localhost:8000/index.html
```

Ouvrir `index.html` en double-clic (`file://`) fonctionne aussi, mais **sans** le chargement
auto de `data/history.json` (passer par le serveur local pour le tester).

## Déploiement

Hébergé sur **GitHub Pages** : tout commit sur `main` est publié tel quel, pas d'étape de build.
URL : `https://atheenais.github.io/chrysapulse/`.

> ⚠️ **Données sensibles.** Les verbatims contiennent du feedback nominatif de collaborateurs.
> Ce dépôt étant public, ces verbatims sont publiquement lisibles. À garder en tête avant de
> committer de nouvelles vagues.

## Documentation technique

Voir [`CLAUDE.md`](CLAUDE.md) : architecture, modèle de données, règle eNPS, système de design,
import Forms, pièges connus, recettes de modification, tests.
