# CLAUDE.md — Dashboard « Prenons le pouls » (NPS Collaborateurs Chrysalides)

Fichier d'instructions pour poursuivre le développement de l'outil avec Claude Code.
Lis ce document **en entier** avant toute modification.

---

## 1. Objectif de l'outil

Tableau de bord interne, **très lisible**, des résultats de l'enquête NPS Collaborateurs
« Prenons le pouls » de Chrysalides. Il doit :

- afficher les résultats d'un sondage Microsoft Forms (10 dimensions notées /10 + commentaires facultatifs) ;
- offrir une **vue fusionnée internes + freelances**, isolable d'un clic ;
- respecter **les verbatims mot pour mot** ;
- respecter les **codes graphiques de Chrysalides** (https://chrysalides.me) ;
- conserver un **historique de chaque chargement dans le temps** et en visualiser l'**évolution** (eNPS global et par question, timeline datée, comparaison vs chargement précédent).

Public : direction / RH de Chrysalides. Langue de l'interface et du code : **français**.

---

## 2. État actuel (déjà livré)

- KPI globaux : score moyen /10, eNPS global, répondants, répartition Promoteurs/Passifs/Détracteurs.
- Classement des 10 dimensions avec barres internes vs freelances (toujours visibles, segment actif mis en avant, tri dynamique).
- eNPS par dimension.
- Verbatims regroupés par question, étiquetés Interne/Freelance, retranscrits exactement (`white-space:pre-line`). **Un verbatim a été retiré sur demande** (Interne #15, Q5).
- Sélecteur de **population** (Tous / Internes / Freelances) : recalcule tout en direct.
- Sélecteur de **chargement** : affiche n'importe quel snapshot historisé.
- Section **Évolution** : comparaison actuel vs précédent, timeline SVG (métrique eNPS / score moyen × dimension Global / Q1..Q10), tableau « scores détaillés par chargement ».
- Section **Gérer les chargements** : import `.xlsx`/`.csv`, liste des snapshots avec **Afficher** + **Supprimer** (par chargement), **Exporter/Importer l'historique (.json)**, **Réinitialiser**, **état vide** + restauration du chargement initial.

---

## 3. Fichiers & source de vérité

| Fichier | Rôle |
|---|---|
| `index.html` | **SOURCE DE VÉRITÉ unique.** Application complète (HTML + CSS + JS inline + `SEED` embarqué). Inclut la config et la couche **Appwrite** (constantes `AW_*`). Sert la racine sur GitHub Pages. |
| `data/history.json` | **Backup / graine** de l'historique (tableau brut de snapshots = sortie du bouton « Exporter l'historique »). **N'est plus chargé au boot** (remplacé par Appwrite, voir §7) ; sert à seeder la base via « Importer un historique (.json) » et de sauvegarde. |
| `CLAUDE.md` | Ce document. |
| `README.md` | Présentation produit + déploiement. |

> **Important.** L'outil a été bootstrappé par un générateur Python (`build.py` + `seed.json`), mais
> **ils ne sont plus nécessaires**. À partir de maintenant, **on édite directement le fichier HTML**.
> Ne réintroduis pas d'étape de build : l'autonomie du fichier (un seul `.html` à ouvrir/partager) est une exigence produit.
> Si tu as besoin du générateur d'origine pour repartir des données brutes, demande-le.

Le fichier est organisé en 3 blocs dans cet ordre :
1. `<style>` … `</style>` — tout le CSS (tokens dans `:root`).
2. `<body>` … markup des sections.
3. `<script>` … `</script>` — toute la logique. La donnée de référence est la constante **`SEED`** en haut du script.

Aucune dépendance npm. Dépendances réseau :
- **Appwrite Web SDK** `https://cdn.jsdelivr.net/npm/appwrite@17.0.0` — **requis pour la persistance** (lecture/écriture des chargements). Hors ligne, le dashboard retombe sur le cache `localStorage` puis sur `SEED` (affichage seulement, pas d'écriture). Voir §7.
- Google Fonts (Fraunces + Inter) — fallback `serif`/`sans-serif` si hors ligne. *(optionnel)*
- SheetJS `https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js` — chargé **à la demande** uniquement pour l'import `.xlsx`. Le `.csv` fonctionne hors ligne. *(optionnel)*

---

## 4. Lancer / prévisualiser

Ouvrir le fichier dans un navigateur suffit (`file://`).

```bash
# Aperçu local recommandé (évite certaines restrictions file://) :
python3 -m http.server 8000
# puis http://localhost:8000/index.html
```

> ⚠️ **Teste toujours via le serveur local (http), pas en `file://`.** Appwrite n'autorise que les hôtes
> déclarés comme **plateformes Web** dans le projet (`localhost` et `atheenais.github.io`) — en `file://`
> l'origine est `null` et les appels Appwrite sont bloqués par CORS. La connexion GitHub (OAuth) exige aussi http(s).

---

## 5. Architecture technique

### 5.1 Modèle de données

```js
SEED = {
  questions: [ { id:1, label:"…(libellé complet)…", short:"…(libellé court)…" }, … ], // 10 questions, id 1..10
  respondents: [
    {
      id: 1,
      role: "freelance" | "interne",
      scores:   { "1": 10, "2": 9, … "10": 9 },   // clé = id question (string), valeur = note 0..10
      comments: { "1": "verbatim exact…" }          // optionnel, clés partielles
    }, …
  ]
}
```

### 5.2 Snapshot (chargement historisé)

```js
snapshot = {
  id: "snap_xxxxx",          // uid()
  date: "2026-06-22",        // YYYY-MM-DD, sert au tri et à l'axe temps
  label: "Juin 2026",        // libellé affiché
  questions: [...],          // copie de SEED.questions (même sondage)
  respondents: [...]         // données brutes du chargement
}
```

Chaque snapshot embarque **ses propres données brutes** → tous les écrans peuvent se recalculer pour n'importe quel chargement.

### 5.3 État global (variables en tête de `<script>`)

| Variable | Sens |
|---|---|
| `HISTORY` | tableau de snapshots, **trié par date croissante** (`sortHistory()`) |
| `CURID` | id du snapshot affiché (`null` si historique vide) |
| `SEG` | population active : `'all' | 'interne' | 'freelance'` |
| `TLMETRIC` | métrique de la timeline : `'enps' | 'mean'` |
| `Q`, `R` | raccourcis vers `questions` / `respondents` du snapshot courant, posés par `bind()` |
| `aw`, `awAccount`, `awDB` | client Appwrite + services (initialisés paresseusement par `awReady()`) |
| `USER` | utilisateur connecté (`account.get()`) ou `null` ; pilote la barre d'auth et `requireLogin()` |

### 5.4 Flux de rendu

`render()` est le point d'entrée unique :

1. Si `HISTORY` vide → `showEmpty(true)` + `renderSnapList()` + return (état vide).
2. Sinon `bind()` (pose `Q`, `R`) puis met à jour : KPI, `renderDims`, `renderEnps`, `renderVerbatims`, `renderEvolution`, `renderSnapList`.
3. Les sélecteurs réémettent `render()` ou un sous-rendu via `wire()`.

Toujours **passer par `render()`** après une mutation de `HISTORY`/`CURID`/`SEG`.

### 5.5 Carte des fonctions

```
Données/calcul : enpsOf, meanOf, qScores, qAvg, qEnps, allScores, allScoresSeg, statsFor, pop
Historique     : loadHistory, saveHistory, seedSnap, sortHistory, initHistory, curSnap, prevOf, bind
Appwrite/data  : awReady, snapToRow, rowToSnap, loadFromAppwrite, awCreateSnap, awDeleteSnap, awDeleteAll, awSeed
Auth GitHub    : isLogged, refreshUser, handleOAuthReturn, loginGitHub, logout, requireLogin, renderAuth
Rendu          : render, renderDims, barRow, renderEnps, renderVerbatims, renderEvolution,
                 buildTimelineSelect, buildTimeline, buildEvoTable, buildSnapSelect, renderSnapList, showEmpty
Deltas/affich. : deltaHtml, dcell, roleFR, fr1, escapeHtml, cap, norm, uid
Import         : handleFile, parseCSV, rowsToRespondents, aoaToRespondents, loadSheetJS, showImportForm
Câblage UI     : wire
Boot           : initHistory(); buildSnapSelect(); wire(); render(); renderAuth();
                 puis (async) handleOAuthReturn → refreshUser → renderAuth → loadFromAppwrite → render
```

---

## 6. Règle eNPS (à ne pas casser)

Échelle 0–10. Classification :
- **Promoteurs** : note **≥ 9**
- **Passifs** : note **7–8**
- **Détracteurs** : note **≤ 6**

`eNPS = round(%promoteurs − %détracteurs)` → plage théorique −100 à +100.
Implémenté dans `enpsOf(array)`. Toute modification des seuils doit l'être **uniquement ici** (et la légende des KPI mise à jour en conséquence).

> Note méthodo : il n'existe pas de question « recommanderiez-vous… » unique dans ce sondage. L'eNPS est donc
> calculé sur l'ensemble des 10 dimensions notées → c'est un **indice d'engagement façon eNPS**, pas un eNPS canonique.
> Si une vraie question de recommandation est ajoutée au Forms, prévoir un eNPS dédié basé sur elle.

---

## 7. Persistance & historique — Appwrite

**Source de vérité = base Appwrite** (partagée entre tous les appareils). `localStorage` n'est plus qu'un **cache hors-ligne**, `SEED` un filet de dernier recours.

### Backend Appwrite (Cloud, région `fra`)
Constantes en tête du `<script>` (`index.html`) :
```
AW_ENDPOINT = 'https://fra.cloud.appwrite.io/v1'
AW_PROJECT  = '6a3d2314000e362641e2'
AW_DB       = '6a3d3aed003676b53dfb'   // Database ID (auto-généré ; le nom est "chrysapulse")
AW_TABLE    = 'pulse_snapshots'        // Table/Collection ID
```
- **SDK 17** : le global CDN expose **`Appwrite.Databases`** (API classique, **pas** `TablesDB`), méthodes **positionnelles** : `listDocuments(db, col, queries)`, `createDocument(db, col, id, data)`, `deleteDocument(db, col, id)`. Auth : `account.get()`, `createOAuth2Token(provider, success, failure)` (qui **redirige** le navigateur en SDK web), `createSession(userId, secret)`, `deleteSession('current')`.
- **Modèle ligne** : 1 row = 1 snapshot. Colonnes `date` (string), `label` (string), `payload` (**longtext** = JSON `{questions, respondents}`). Le **`$id` de la row = l'id du snapshot** (`snap_seed_juin_2026`, ou `uid()`). `snapToRow()` / `rowToSnap()` font la conversion.
- **Permissions** (niveau table, Row Security OFF) : Read = `Any` (lecture publique) ; Create/Update/Delete = utilisateurs autorisés. ⚠️ **Restreindre l'écriture à une Team** (voir §sécurité) — `Role.users()` laisserait écrire n'importe quel compte GitHub.

### Auth — GitHub OAuth (flux token, cross-domaine)
1. `loginGitHub()` → `createOAuth2Token(Github, here, here)` redirige vers GitHub.
2. Retour sur la page avec `?userId&secret` → `handleOAuthReturn()` appelle `createSession(userId, secret)` puis nettoie l'URL.
3. Le provider GitHub doit être **activé** dans Auth → Providers (App ID + secret d'une *GitHub OAuth App*, callback = celui affiché par Appwrite). Les hôtes `localhost` + `atheenais.github.io` doivent être déclarés comme **plateformes Web**.

### Flux de chargement / écriture
- **Boot** : `initHistory()` peint le cache local instantanément, puis `loadFromAppwrite()` (lecture anon) **écrase** `HISTORY` avec la base (autoritaire) et met à jour le cache. Échec réseau → on garde le cache.
- **Écritures** (toutes derrière `requireLogin()`) : import d'une vague → `createDocument` ; suppression → `deleteDocument` ; réinitialiser → `awDeleteAll()` + `awSeed()`. Après chaque écriture → `loadFromAppwrite()` + `render()`.

### Workflow « ajouter un chargement de résultats »
1. Se **connecter** (GitHub) dans le dashboard.
2. **Importer** l'export Forms (`.xlsx`/`.csv`) → renseigner date + libellé → « Ajouter à l'historique » (insert Appwrite).
3. Tous les appareils voient la vague au prochain rafraîchissement (lecture publique). Pas de commit/push nécessaire — la donnée vit dans Appwrite, plus dans le repo.
4. `data/history.json` reste une **sauvegarde** (bouton « Exporter l'historique ») et la **graine** initiale (« Importer un historique (.json) » quand la base est vide).

> Clé localStorage (cache) : **`chrysalides_pulse_history_v1`**. En cas de changement de schéma snapshot incompatible, incrémenter la version de la clé. Le seed garde l'**id stable `snap_seed_juin_2026`** (évite les doublons à l'import).

---

## 8. Import — contrat de données (export Forms)

`handleFile()` route selon l'extension :
- `.csv` → `parseCSV()` (parseur RFC4180 minimal, hors ligne).
- `.xlsx`/`.xls` → `loadSheetJS()` puis `XLSX.utils.sheet_to_json(..., {header:1})` → `aoaToRespondents()`.

Les deux convergent vers **`rowsToRespondents(rows)`** qui :
- normalise les en-têtes (`norm()` : minuscule, sans accent, espaces compactés) ;
- repère la **colonne rôle** par le mot-clé `papillon` ; rôle = `freelance` si la valeur contient « freelance », sinon `interne` ;
- repère chaque **colonne question** via `QKEYS` (mot-clé unique par question) ;
- prend la **colonne commentaire** = colonne juste après si son en-tête commence par `commentaire`.

```js
QKEYS = {1:'reconnaissance',2:'equilibre',3:'liberte',4:'contribution',5:'accompagnement',
         6:'dynamique collective',7:'pertinence',8:'sens et valeur',9:'ambiance',10:'appartenance'}
```

Structure Forms attendue (ordre indicatif, le mapping est tolérant à l'ordre) :
`Id | Heure de début | Heure de fin | Adresse de messagerie | Nom | "…papillon… ?" |
[Question 1] | Commentaire | [Question 2] | Commentaire1 | … | [Question 10] | Commentaire9`

> Si Microsoft change les libellés de colonnes, ajuster `QKEYS` (et au besoin le repérage du rôle). Les libellés
> de questions affichés viennent de `SEED.questions`, pas du fichier importé (sondage identique d'une vague à l'autre).

---

## 9. Système de design (codes graphiques)

Tous les tokens sont dans `:root` (haut du `<style>`). **Ne jamais coder une couleur en dur** : utiliser les variables.

```css
--corail:#D11D65;  --corail-soft:#F7D9E4;   /* aile chaude / freelances (magenta de marque) */
--teal:#13AB91;    --teal-soft:#D6F0EA;     /* aile froide / internes */
--gold:#F5C842;                              /* accent / valeurs moyennes */
--encre:#091628;   --ardoise:#6E7E9E;        /* textes (bleu nuit / périer) */
--ligne:#E7E1D8;   --fond:#FBF8F4;  --carte:#FFFFFF;
--vert:#23A455; --jaune:#F5C842; --rouge:#E0533B;  /* promoteurs / passifs / détracteurs */
--r:18px;  --shadow:…                         /* rayon, ombre */
```

- Typo : **Comfortaa** (display — titres + chiffres clés, arrondie, police officielle de marque) + **Inter** (corps/données, lisibilité). Métaphore visuelle : « papillon » (teal/magenta = deux ailes).
- Sémantique couleur eNPS : `≥30` teal, `0..29` gold, `<0` rouge (cf. `renderEnps`, cartes, table).
- ⚠️ Le **rouge détracteur `#E0533B`** est volontairement **distinct du magenta `#D11D65`** (aile chaude) pour ne pas confondre « freelance » et « détracteur ». Quelques couleurs dérivées sont codées en dur : texte de puce freelance / delta baissier = `#A01457` (magenta foncé), texte interne / delta haussier = `#0c6e67` (teal foncé).

> ✅ **Palette = charte officielle Chrysalides**, extraite du kit global Elementor de https://chrysalides.me
> (couleurs `--e-global-color-*` + typo `Comfortaa`). Pour ajuster : **modifier uniquement les valeurs `:root`**
> (+ les 2 hex dérivés ci-dessus si besoin) — tout le reste suit. Autres couleurs de marque dispo si besoin :
> turquoise vif `#3CE8D0`, indigo `#4054B2`, pêche `#FFBC7D`, navy alt `#0F2044`. Valeurs A.O.A.H
> (Accomplissement, Originalité, Authenticité, Humanité), siège Roubaix.

---

## 10. Timeline SVG — repères

Générée par `buildTimeline()` (SVG inline, responsive via `viewBox="0 0 860 300"`).

- `data` = une valeur par snapshot selon `TLMETRIC` (`enps`/`mean`) et la dimension choisie (`#tlDim` : `global` ou un id de question), pour la population `SEG`.
- Échelle Y auto : eNPS arrondi au pas de 5 et borné [−100,100] (avec ligne du zéro pointillée si la plage la traverse) ; score moyen borné [0,10] au pas 0,5.
- `X(i)` répartit les points ; point unique → centré.
- Couleur de courbe selon `SEG` (teal/corail/encre).
- Valeurs nulles → coupure de la ligne (gestion des trous).

Pour ajouter une métrique : étendre le `miniseg#tlMetric`, gérer la valeur dans `buildTimeline()` et l'échelle Y.

---

## 11. Conventions & pièges connus (GOTCHAS)

1. **Barres de progression** : `.fill` **doit** être `display:block`. Un `<span>` `inline` ignore `width` (bug historique : barres invisibles). La largeur est posée en ligne (`width: val*10 %`), l'animation passe par `transform:scaleX` (indépendante de la largeur), donc la barre est correcte même sans JS.
2. **Échelle des barres** : vraie échelle **0–10** (honnêteté des écarts). Ne pas « zoomer » sans le dire.
3. **Verbatims** : exactitude obligatoire. `escapeHtml()` + `white-space:pre-line` (sauts de ligne préservés). Ne jamais reformuler/corriger un verbatim.
4. **Pas de `<form>` ni de stockage navigateur côté artifact** : ici on est en fichier téléchargé, `localStorage` OK. Ne pas dépendre de localStorage pour le rendu de base (le seed garantit un affichage même sans stockage).
5. **Accessibilité / mouvement** : `prefers-reduced-motion` coupe transitions **et** animations. Conserver le focus clavier visible et le responsive (breakpoints 820px / 520px).
6. **`render()` après chaque mutation** d'état. Après modif de `HISTORY`, appeler aussi `buildSnapSelect()`.
7. **Suppression du dernier chargement** → état vide géré (`showEmpty`) + bouton de restauration. Ne pas réintroduire de plantage si `HISTORY` est vide (toujours garder le garde-fou en tête de `render()`).
8. **`norm()`** retire les accents mais **pas les apostrophes** : en tenir compte pour les mots-clés de mapping.

---

## 12. Recettes de modification courantes

**Changer la palette vers la charte exacte** → éditer les hex dans `:root` uniquement.

**Modifier un libellé de question** → `SEED.questions[].label` / `.short` (le `short` sert aux barres et cartes, le `label` complet aux verbatims).

**Changer les seuils eNPS** → `enpsOf()` + légende KPI + bornes couleur (`renderEnps`, cartes compare, `buildEvoTable`).

**Ajouter une métrique timeline** → `#tlMetric` (markup) + branche dans `buildTimeline()` + échelle Y.

**Comparer deux chargements au choix (au lieu de « précédent »)** → ajouter deux `select` dans la section Évolution, et une variante de `renderEvolution()` paramétrée par deux ids (réutiliser `statsFor` + `deltaHtml`/`dcell`).

**Exporter le dashboard en PDF/PNG** → impression navigateur ou ajout d'un bouton `window.print()` + `@media print`.

**Nouveau type de répondant (au-delà interne/freelance)** → généraliser `role`, le filtre `SEG`, les couleurs et `pop()`. Impacte beaucoup d'endroits : faire une passe complète.

---

## 13. Tests (recommandé avant chaque livraison)

Vérification headless rapide avec Playwright :

```python
# pip install playwright --break-system-packages && python3 -m playwright install chromium
import asyncio
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        b = await p.chromium.launch()
        pg = await (await b.new_context()).new_page()
        errs = []; pg.on("pageerror", lambda e: errs.append(str(e)))
        await pg.goto("http://localhost:8000/index.html")  # http obligatoire (CORS Appwrite + OAuth)
        await pg.wait_for_timeout(800)
        rows = await pg.eval_on_selector_all(".dims .row", "e=>e.length")
        fills = await pg.eval_on_selector_all(".dims .fill",
                "els=>els.filter(e=>e.getBoundingClientRect().width>2).length")  # barres non nulles
        print("rows", rows, "fills>2px", fills, "errs", errs or "aucune")
        await b.close()
asyncio.run(main())
```

Checklist manuelle : toggle population (dont un segment vide → doit afficher `—`/`0%`, pas `NaN`), changement de chargement, **connexion GitHub** (barre 🟢), import `.csv` + ajout (insert Appwrite), comparaison + timeline (eNPS et score, global + une question), suppression d'un chargement (connecté), état vide + restauration, export/import `.json`, **rechargement déconnecté = données toujours là** (lecture publique), responsive mobile.

---

## 14. Pistes / backlog

- Brancher la **charte graphique exacte** (hex officiels) quand disponible.
- **Comparaison libre** entre deux chargements (cf. §12).
- **Export PDF/PNG** d'une vue.
- Vraie question de **recommandation** dans le Forms → eNPS canonique dédié.
- Filtres additionnels (ancienneté, mission, site) si le Forms s'enrichit.
- ✅ **Multi-utilisateurs : fait** — persistance migrée vers **Appwrite** (cf. §7). Reste à faire côté sécurité (cf. §16).

---

## 15. Règles d'or

1. **Un seul fichier HTML** : pas de build, pas de npm. (L'autonomie « 100 % hors-ligne » a été assouplie : Appwrite est désormais **requis pour la persistance** ; le `SEED` + cache `localStorage` assurent un affichage de repli hors-ligne.)
2. **Lisibilité d'abord** (public direction/RH).
3. **Verbatims intouchables**, **données honnêtes** (échelles, eNPS).
4. **Tokens `:root`** pour tout style ; rien en dur.
5. **`render()`** orchestre tout ; `loadFromAppwrite()` après chaque écriture.
6. Tester en headless + checklist avant de livrer.
7. **Écriture = connecté + autorisé** (cf. §16). Ne jamais élargir les permissions de la table au-delà de la Team d'éditeurs.

---

## 16. Sécurité (à ne pas relâcher)

- **Écriture restreinte à une Team.** Le provider GitHub OAuth laisse *n'importe quel* compte GitHub se connecter → si les permissions d'écriture de `pulse_snapshots` sont `Role.users()`, n'importe qui peut insérer/supprimer. **Restreindre Create/Update/Delete à une Team d'éditeurs** (`Role.team(<id>)`), dont seul Fab (et les personnes autorisées) sont membres. La lecture reste publique (`Any`).
- **Lecture publique = données publiques.** Avec endpoint + Project ID (présents dans le source), tout le monde peut lire les verbatims via l'API. C'est assumé tant que le dashboard est public. Pour fermer : passer Read sur la Team + retirer `data/history.json` du repo public.
- **Secrets.** Seul le **Project ID** (public) vit dans `index.html`. Le **client secret GitHub** ne vit que dans la console Appwrite — jamais dans le code ni partagé. Ne **jamais** committer de clé API serveur Appwrite.
- **Données sensibles** : CV/feedback nominatifs. Pas de logs de données perso.
