# ChrysaPulse : Dashboard « Prenons le pouls »

Tableau de bord interne des résultats de l'enquête **NPS Collaborateurs « Prenons le pouls »**
de [Chrysalides](https://chrysalides.me). Vue fusionnée internes + freelances, isolable d'un clic,
avec historisation des chargements dans le temps et visualisation de l'évolution (eNPS, scores,
timeline datée).

Le dashboard couvre : KPI globaux (score moyen, eNPS, répartition), classement des 10 dimensions
en format compact avec **eNPS et tendance par dimension**, verbatims mot pour mot, section
**Évolution** (comparaison vs vague précédente et timeline), et une rubrique **« Pistes d'action »**
proposant un plan à 6 mois déduit des résultats, tendances et verbatims.

Public : direction / RH de Chrysalides.

## Application

Une **page HTML unique** : [`index.html`](index.html), HTML + CSS + JS inline, données seed
embarquées. Aucun framework, aucun build, aucune dépendance npm.

Dépendances réseau :

- **Appwrite Web SDK** (CDN) : **requis pour la persistance** (lecture/écriture des chargements + connexion GitHub). Hors ligne, l'app retombe sur le cache `localStorage` puis le seed embarqué (affichage seul) ;
- Google Fonts (Comfortaa + Inter) : fallback `serif`/`sans-serif` hors ligne *(optionnel)* ;
- SheetJS via CDN : chargé **à la demande** uniquement pour l'import `.xlsx` (le `.csv` marche hors ligne) *(optionnel)*.

## Historique des chargements : Appwrite

Les vagues d'enquête sont stockées dans une base **Appwrite** (Cloud, région `fra`), **partagée entre
tous les appareils**. La **consultation est publique** ; **importer ou supprimer** une vague nécessite un **compte autorisé**
(connexion GitHub ; les contrôles d'écriture sont masqués pour les autres comptes). Le `localStorage` du navigateur sert de cache hors-ligne, et le `SEED`
embarqué de filet de secours.

- **`data/history.json`** : n'est plus la source de vérité : c'est une **sauvegarde** (bouton
  « Exporter l'historique ») et la **graine** pour initialiser une base vide (« Importer un historique »).

### Ajouter une vague de résultats

1. Se **connecter** avec un **compte autorisé** (GitHub) dans le dashboard.
2. **Importer** l'export Microsoft Forms (`.xlsx`/`.csv`), renseigner date + libellé → « Ajouter à l'historique ».
3. C'est écrit dans Appwrite ; **tous les appareils** voient la vague au rafraîchissement. Pas de commit/push.

> Détails techniques (IDs, schéma, permissions, flux OAuth) : voir [`CLAUDE.md`](CLAUDE.md) §7 et §16.

## Lancer en local

```bash
python3 -m http.server 8000
# http://localhost:8000/index.html
```

⚠️ **Ne pas ouvrir en `file://`** : Appwrite n'autorise que les hôtes déclarés (`localhost`,
`atheenais.github.io`) : en `file://` les appels sont bloqués par CORS et l'OAuth ne fonctionne pas.
Toujours passer par le serveur local ci-dessus.

## Déploiement

Hébergé sur **GitHub Pages** : tout commit sur `main` est publié tel quel, pas d'étape de build.
URL : `https://atheenais.github.io/chrysapulse/`. L'hôte `atheenais.github.io` doit rester déclaré
comme plateforme Web dans le projet Appwrite.

> ⚠️ **Données sensibles & accès.** Les verbatims sont du feedback **nominatif** de collaborateurs.
> La lecture Appwrite étant publique (et le dépôt public), ils sont lisibles par tous. L'**écriture**
> est restreinte aux **comptes autorisés** : permission Appwrite côté serveur + liste blanche `AW_EDITORS`
> qui masque les contrôles d'écriture côté interface. Voir [`CLAUDE.md`](CLAUDE.md) §16.

## Documentation technique

Voir [`CLAUDE.md`](CLAUDE.md) : architecture, modèle de données, règle eNPS, système de design,
import Forms, pièges connus, recettes de modification, tests.
