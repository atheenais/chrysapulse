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
| `index.html` | **SOURCE DE VÉRITÉ unique.** Application complète et autonome (HTML + CSS + JS inline + données seed embarquées). Sert la racine sur GitHub Pages. |
| `data/history.json` | **Archive committée de l'historique des chargements** (= tableau brut de snapshots, exactement la sortie du bouton « Exporter l'historique »). Chargée au boot quand le dashboard est servi en http(s). Voir §7. |
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

Aucune dépendance npm. Dépendances réseau **optionnelles** :
- Google Fonts (Fraunces + Inter) — fallback `serif`/`sans-serif` si hors ligne.
- SheetJS `https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js` — chargé **à la demande** uniquement pour l'import `.xlsx`. Le `.csv` fonctionne hors ligne.

---

## 4. Lancer / prévisualiser

Ouvrir le fichier dans un navigateur suffit (`file://`).

```bash
# Aperçu local recommandé (évite certaines restrictions file://) :
python3 -m http.server 8000
# puis http://localhost:8000/index.html
```

> Le `localStorage` fonctionne aussi en `file://` sur Chrome/Firefox, mais un petit serveur local est plus fiable
> et représentatif du comportement réel.
>
> ⚠️ Le **chargement de `data/history.json` au boot ne se déclenche QU'EN http(s)** (`fetch` est bloqué en
> `file://`). Pour tester ce comportement, passe par le serveur local ci-dessus, pas par un double-clic.

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
Rendu          : render, renderDims, barRow, renderEnps, renderVerbatims, renderEvolution,
                 buildTimelineSelect, buildTimeline, buildEvoTable, buildSnapSelect, renderSnapList, showEmpty
Deltas/affich. : deltaHtml, dcell, roleFR, fr1, escapeHtml, cap, norm, uid
Import         : handleFile, parseCSV, rowsToRespondents, aoaToRespondents, loadSheetJS, showImportForm
Câblage UI     : wire
Boot           : initHistory(); buildSnapSelect(); wire(); render();
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

## 7. Persistance & historique

- Clé localStorage : **`chrysalides_pulse_history_v1`**.
- `initHistory()` : charge l'historique depuis localStorage ; si vide/absent → seed `Juin 2026` (`seedSnap()`, **id stable `snap_seed_juin_2026`**, date `2026-06-22`) à partir de `SEED`.
- Sauvegarde via `saveHistory()` après **toute** mutation.
- Le `localStorage` est **propre au navigateur** → d'où l'archive committée + l'**Export/Import `.json`** pour la portabilité.

### Archive committée `data/history.json` ↔ boot-sync (depuis v repo GitHub)

- `data/history.json` = **tableau brut de snapshots**, exactement le format produit par le bouton « Exporter l'historique » (qui télécharge désormais un fichier nommé `history.json`).
- `syncFromFile()` s'exécute **après** `initHistory()`, **uniquement si la page est servie en `http(s)`** (`fetch` est bloqué en `file://`). Elle :
  1. `fetch('data/history.json')` ;
  2. **fusionne par `id`** : le fichier fait foi, les snapshots **purement locaux** (présents en localStorage mais absents du fichier — ex. un import non encore committé) sont **conservés et ajoutés** ;
  3. `saveHistory()` + re-render.
- **Conséquence voulue** : committer un nouveau `history.json` propage le chargement à **tous les appareils** au prochain rafraîchissement. Supprimer localement un snapshot présent dans le fichier le **réaffiche** au boot suivant (le fichier est la source de vérité). Pour retirer définitivement un chargement, l'enlever **du fichier** et re-committer.
- **Id stable du seed** = indispensable : sans lui, chaque navigateur générerait un id aléatoire pour « Juin 2026 » et la fusion créerait des doublons.
- ⚠️ **Migration ponctuelle** : un navigateur qui a déjà un seed « Juin 2026 » avec un **ancien id aléatoire** (avant cette version) verra transitoirement un doublon de Juin après déploiement → cliquer une fois sur **« Réinitialiser »**, ou supprimer le doublon, suffit à réaligner.

### Workflow « ajouter un chargement de résultats »

1. Dans le dashboard hébergé (ou en local), **importer** le nouvel export Forms (`.xlsx`/`.csv`) → renseigner date + libellé.
2. Cliquer **« Exporter l'historique (.json) »** → récupère `history.json`.
3. Remplacer `data/history.json` du repo par ce fichier, **committer & pousser**.
4. Au prochain rafraîchissement, tous les appareils voient le nouveau chargement.

- En cas de changement de schéma de snapshot incompatible, **incrémenter la version de la clé** (`…_v2`) et prévoir une migration.

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
--corail:#F25C3B;  --corail-soft:#FBE3DB;   /* aile chaude / freelances */
--teal:#129B92;    --teal-soft:#D6EFEC;     /* aile froide / internes */
--gold:#F4A93B;                              /* accent / valeurs moyennes */
--encre:#1E2A32;   --ardoise:#5A6B75;        /* textes */
--ligne:#E7E1D8;   --fond:#FBF8F4;  --carte:#FFFFFF;
--vert/--jaune/--rouge                       /* promoteurs / passifs / détracteurs */
--r:18px;  --shadow:…                         /* rayon, ombre */
```

- Typo : **Fraunces** (display, titres/chiffres) + **Inter** (corps/données). Métaphore visuelle : « papillon » (teal/corail = deux ailes).
- Sémantique couleur eNPS : `≥30` teal, `0..29` gold, `<0` rouge (cf. `renderEnps`, cartes, table).

> ⚠️ **La palette est une interprétation fidèle de l'identité Chrysalides, pas les hex officiels exacts**
> (le logo n'a pas pu être lu automatiquement). Si Fab fournit la charte exacte, **remplacer uniquement les
> valeurs `:root`** : tout le reste suit. Référence marque : https://chrysalides.me — valeurs A.O.A.H
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
        await pg.goto("http://localhost:8000/index.html")  # http requis pour tester syncFromFile
        await pg.wait_for_timeout(800)
        rows = await pg.eval_on_selector_all(".dims .row", "e=>e.length")
        fills = await pg.eval_on_selector_all(".dims .fill",
                "els=>els.filter(e=>e.getBoundingClientRect().width>2).length")  # barres non nulles
        print("rows", rows, "fills>2px", fills, "errs", errs or "aucune")
        await b.close()
asyncio.run(main())
```

Checklist manuelle : toggle population, changement de chargement, import `.csv`, comparaison + timeline (eNPS et score, global + une question), suppression d'un chargement, état vide + restauration, export/import `.json`, rechargement de page (persistance), responsive mobile.

---

## 14. Pistes / backlog

- Brancher la **charte graphique exacte** (hex officiels) quand disponible.
- **Comparaison libre** entre deux chargements (cf. §12).
- **Export PDF/PNG** d'une vue.
- Vraie question de **recommandation** dans le Forms → eNPS canonique dédié.
- Filtres additionnels (ancienneté, mission, site) si le Forms s'enrichit.
- **Multi-utilisateurs** (évolution probable) : si l'historique doit être partagé entre plusieurs personnes/machines, migrer la persistance de `localStorage` vers **Supabase** (table `snapshots` en JSONB), en gardant GitHub Pages pour l'hébergement statique. Conserver l'export/import `.json` comme passerelle de migration.

---

## 15. Règles d'or

1. **Un seul fichier HTML autonome** : pas de build, pas de dépendance obligatoire en ligne.
2. **Lisibilité d'abord** (public direction/RH).
3. **Verbatims intouchables**, **données honnêtes** (échelles, eNPS).
4. **Tokens `:root`** pour tout style ; rien en dur.
5. **`render()`** orchestre tout ; sauvegarde après chaque mutation.
6. Tester en headless + checklist avant de livrer.
