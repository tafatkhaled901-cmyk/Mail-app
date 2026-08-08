# Rapport d'analyse — Application « Déclarations Dépôts » (GPS Glass)

**Dépôt :** `tafatkhaled901-cmyk/Mail-app`
**Fichier analysé :** `index.html` (794 lignes, 47 569 octets — fichier unique)
**Date du rapport :** 8 août 2026
**Branche :** `claude/rapport-application-sq4k9r`

---

## 1. Résumé exécutif

L'application est un **prototype mono-fichier** (HTML + CSS + JavaScript embarqués, sans build ni
dépendances locales) destiné à consulter, filtrer et exporter les déclarations de dépôts de
GPS Glass — verre plat : **casse**, **ventes** et **transferts de stock**.

L'interface est soignée, cohérente, responsive et pensée mobile-first. Le travail de design est le
point fort du projet.

En revanche, **l'application n'est pas fonctionnelle en l'état** : un défaut de comparaison de
chaînes accentuées fait que le filtre par dépôt ne renvoie **jamais aucune ligne**, quel que soit le
dépôt choisi (§ 6.1). À cela s'ajoute l'absence totale de backend : les données sont figées en dur
dans le fichier, et l'écran de connexion ne réalise aucune authentification réelle.

| Axe | Évaluation |
|---|---|
| Design / ergonomie visuelle | Bon |
| Fonctionnement effectif | **Bloqué** (bug critique) |
| Architecture | Prototype — non extensible en l'état |
| Sécurité | **Insuffisante** — pas d'authentification réelle |
| Accessibilité | Faible |
| Outillage (tests, CI, doc) | Inexistant |

**Verdict :** excellente maquette fonctionnelle, à considérer comme un *proof of concept*.
Un passage en production exige un backend, une authentification et la correction des anomalies
listées au § 6.

---

## 2. Périmètre et objectif du projet

L'application répond à un besoin métier clair : les dépôts déclarent quotidiennement, par e-mail,
leurs opérations. Le responsable doit pouvoir les consulter de façon consolidée et les exporter.

Trois natures de déclaration sont gérées :

| Type | Libellé applicatif | Données suivies |
|---|---|---|
| `casse` | Déclarations de Casse | Réf. caisse, n° lot, feuilles jetées / restantes / total |
| `vente` | Bons de Sortie Vente | N° commande, client, réf. produit, n° lot, caisses, feuilles |
| `transfert` | Sorties de Stock | Dépôt d'origine, destination, nombre de feuilles |

Le commentaire `// DONNÉES TEST (extraites de tafatkhaled901@gmail.com)` (`index.html:416`) et le
nom du dépôt Git (`Mail-app`) indiquent que la source de données visée est **une boîte e-mail**,
dont le contenu devait être extrait puis structuré. **Cette extraction n'existe pas dans le code** :
les données ont été recopiées manuellement.

---

## 3. Architecture technique

### 3.1 Structure

Tout tient dans un seul fichier, en trois blocs :

| Bloc | Lignes | Contenu |
|---|---|---|
| CSS | 10–229 | ~220 lignes, variables CSS, 3 points de rupture responsive, feuille d'impression |
| HTML | 231–395 | 3 écrans : connexion, choix du dépôt, application |
| JavaScript | 397–792 | ~395 lignes, ~25 fonctions, état global en variables de module |

### 3.2 Modèle de navigation

Une **SPA sans routeur** : les trois écrans coexistent dans le DOM et sont commutés en
affichage/masquage par `allerVers()` (`index.html:497`).

```
Écran 1 — Connexion e-mail
        │  seConnecter()  → vérifie le suffixe @gps-glass.com
        ▼
Écran 2 — Choix du dépôt (grille générée depuis la constante DEPOTS)
        │  validerDepot() → écrit la session dans sessionStorage
        ▼
Écran 3 — Application (onglets, filtres, tableau, exports)
```

Conséquence : aucune URL ne reflète l'état. Rechargement, bouton « Précédent » du navigateur,
partage de lien et signets ne fonctionnent pas comme attendu. Une session sauvegardée dans
`sessionStorage` (`index.html:488`) atténue partiellement le problème.

### 3.3 Dépendances externes

Trois bibliothèques chargées depuis cdnjs, **sans attribut `integrity` (SRI)** et sans repli
hors-ligne (`index.html:7-9`) :

| Bibliothèque | Version figée | Usage |
|---|---|---|
| SheetJS `xlsx` | 0.18.5 | Export Excel |
| `jsPDF` | 2.5.1 | Export PDF |
| `jspdf-autotable` | 3.5.31 | Mise en tableau du PDF |

Voir § 7.3 pour l'analyse de risque associée.

### 3.4 Gestion de l'état

Six variables globales (`index.html:468-474`) : `emailCo`, `depotActif`, `typeActif`, `dfAffiche`,
`sortKey`, `sortAsc`, `menuOuvert`. Approche acceptable à cette taille, mais qui devient un facteur
de régressions dès que l'application grandit.

---

## 4. Fonctionnalités implémentées

- **Connexion par e-mail** avec contrôle du domaine `@gps-glass.com` et suggestion de compte de test.
- **Sélection de dépôt** parmi 5 entrées configurables (4 génériques « Dépôt A à D » + « GPS Benicarlò »),
  avec grille responsive (2 / 3 / 4 colonnes) et changement de dépôt depuis l'en-tête.
- **Persistance de session** via `sessionStorage` (restaurée au chargement).
- **Trois onglets** colorés par type de déclaration.
- **Filtrage par période** : toutes / semaine en cours / mois en cours / intervalle de dates.
- **Tableau trié** par clic sur en-tête, avec indicateur de sens de tri.
- **Statistiques contextuelles** : par exemple, pour la casse, nombre de caisses, feuilles jetées,
  feuilles restantes et **taux de casse** ; pour les ventes, lignes, feuilles et clients distincts.
- **Mise en évidence visuelle** : ligne rouge si ≥ 10 feuilles jetées, badge tricolore sur la
  colonne « jetées » (vert < 5, ambre 5–9, rouge ≥ 10), ligne verte si transfert ≥ 50 feuilles.
- **Trois exports** : Excel `.xlsx` (avec largeurs de colonnes calculées), PDF paysage A4
  (en-tête coloré, pagination) et impression navigateur via feuille de style dédiée.
- **Retours utilisateur** : messages d'erreur en ligne, animation *shake* sur champ invalide, toasts.

---

## 5. Données embarquées

29 lignes au total, **toutes rattachées au dépôt « GPS Benicarlò »** (`index.html:417-453`) :

| Type | Lignes | Période couverte |
|---|---|---|
| Casse | 10 | 07/07/2026 → 29/07/2026 |
| Vente | 12 | 12/07/2026 → 29/07/2026 |
| Transfert | 7 | 09/07/2026 → 29/07/2026 |

**Les dépôts A, B, C et D n'ont aucune donnée** : les sélectionner affichera toujours
« Aucune déclaration trouvée » — ce qui est ici le comportement correct, mais indiscernable du bug
décrit au § 6.1.

### 5.1 Qualité des données source

Le jeu de test révèle la nature « texte libre » de la saisie amont, et donc le besoin d'une
normalisation côté ingestion :

- **Destinations mal orthographiées** : `Alicanti` vs `Alicante`, `Madride` (Madrid),
  `Valenvia` vs `Valencia`, `Ytree` (non identifiable). Sans dictionnaire de destinations,
  la statistique « Dests » comptera 7 destinations distinctes là où il n'y en a
  vraisemblablement que 3 ou 4.
- **Numéros de commande incohérents** : `098765`, `3467532`, `11111111111111`, `1234567890` —
  aucune longueur ni format stable.
- **Ligne incomplète** : vente du 12/07 (client `Gjhz`), lot `—` et 0 feuille pour 1 caisse.
- **Noms de clients douteux** : `Gjhz`, `Rearg` — probablement des saisies de test.

Les totaux de casse sont en revanche **cohérents** : dans les 10 lignes, `jetées + restantes = total`
est systématiquement vérifié. Ce contrôle mériterait d'être automatisé à l'ingestion.

---

## 6. Anomalies identifiées

### 6.1 🔴 CRITIQUE — Le filtre par dépôt ne renvoie jamais de résultat

`index.html:633-636` compare le champ `depot` de chaque ligne aux mots-clés du dépôt :

```js
let rows = DONNEES_TEST[typeActif].filter(r => {
  const depotRow = (r.depot||'').toLowerCase();
  return depotActif.motsCles.some(mc => depotRow.includes(mc));
});
```

Or les données portent `depot:"GPS Benicarlò"` (avec un **`ò` accentué**, U+00F2) tandis que les
mots-clés déclarés en `index.html:408` sont `['benicarlo','gps benicarlo','almacen gps']` —
**sans accent**. `toLowerCase()` ne supprime pas les diacritiques.

Vérification exécutée :

```
depotRow = "gps benicarlò"   →   .includes("benicarlo")  =  false
```

**Impact :** l'unique dépôt disposant de données ne correspond à aucun mot-clé. Cliquer sur
« Générer » affiche systématiquement « Aucune déclaration trouvée », pour les trois onglets et
pour les cinq dépôts. **L'application ne montre jamais aucune donnée.**

**Correction recommandée** — normaliser les deux côtés de la comparaison :

```js
const norm = s => (s||'').toLowerCase()
  .normalize('NFD').replace(/[\u0300-\u036f]/g,'');
const depotRow = norm(r.depot);
return depotActif.motsCles.some(mc => depotRow.includes(norm(mc)));
```

À terme, remplacer ce rapprochement par mots-clés par un **identifiant de dépôt** (`depotId`) porté
par la donnée elle-même : la correspondance textuelle restera toujours fragile.

---

### 6.2 🟠 MAJEUR — Aucune authentification réelle

`seConnecter()` (`index.html:507-524`) se limite à vérifier que la chaîne saisie contient `@`, un `.`
et se termine par `@gps-glass.com`. **Aucun mot de passe, aucun jeton, aucune vérification serveur.**

Toute personne ouvrant la page peut saisir `nimporte.qui@gps-glass.com` et accéder à l'intégralité
des données. La mention rassurante affichée à l'écran — « Aucun mot de passe stocké par cette
application » (`index.html:261`) — est techniquement exacte mais **trompeuse** : il n'y a pas de
mot de passe parce qu'il n'y a pas d'authentification.

Ce point est sans gravité tant que les données sont fictives ; il devient bloquant dès la première
donnée réelle. Voir § 7.1.

---

### 6.3 🟠 MAJEUR — Le tri par date est chronologiquement faux

`trier()` (`index.html:724-729`) traite toute clé non numérique comme une chaîne. La colonne
« Date » contient un format `JJ/MM/AAAA`, trié alphabétiquement :

```
Tri obtenu :  03/08/2026, 11/07/2026, 12/07/2026, 29/07/2026, 30/06/2026
Tri attendu : 30/06/2026, 11/07/2026, 12/07/2026, 29/07/2026, 03/08/2026
```

L'ordre n'est correct que si toutes les lignes appartiennent au même mois et à la même année — ce
qui est le cas du jeu de test actuel, d'où l'anomalie passée inaperçue.

**Correction :** trier sur `dateISO`, déjà présent dans chaque enregistrement, lorsque la clé
demandée est `date`.

---

### 6.4 🟡 MOYEN — Les périodes « semaine » et « mois » ne renvoient rien

Les données couvrent juillet 2026 ; la date du jour est le 8 août 2026. Simulation des filtres :

| Filtre | Casse | Vente | Transfert |
|---|---|---|---|
| Toutes les déclarations | 10 | 12 | 7 |
| Semaine en cours | **0** | **0** | **0** |
| Mois en cours | **0** | **0** | **0** |
| Intervalle (30 j par défaut) | 9 | 12 | 6 |

Ce n'est pas un bug de code — la logique de `filtrerPeriode()` est correcte — mais le jeu de
données de démonstration rend deux des quatre filtres muets, ce qui donne l'impression d'une
application cassée lors d'une présentation. **Il faut rafraîchir les dates du jeu de test**, ou
mieux, les générer relativement à la date courante.

---

### 6.5 🟡 MOYEN — Décalage de fuseau horaire sur les dates

Deux endroits mélangent temps UTC et temps local :

- `index.html:480-483` : `d.toISOString().slice(0,10)` produit une date **UTC**, utilisée pour
  pré-remplir des champs interprétés en heure **locale**. À l'est de Greenwich en soirée, la date
  proposée par défaut peut être décalée d'un jour.
- `index.html:666` : `new Date(r.dateISO)` interprète `"2026-07-29"` comme minuit **UTC**, alors que
  les bornes `debut` / `fin` sont construites en heure **locale**. Pour un utilisateur en UTC−n, une
  ligne située en bordure d'intervalle sera incluse ou exclue à tort.

**Correction :** construire les dates locales explicitement
(`new Date(y, m-1, d)`) plutôt que de passer par le parsing ISO.

---

### 6.6 🟡 MOYEN — Injection HTML possible dès l'arrivée de données réelles

`afficherTableau()` (`index.html:688-698`) insère les valeurs directement via `innerHTML` :

```js
return `<td${...}>${aff}</td>`;
```

Aucun échappement. Avec les données actuelles, figées dans le code, le risque est nul. Mais **dès
que les lignes proviendront d'e-mails** — c'est l'objectif annoncé du projet — un nom de client ou
une référence contenant `<img src=x onerror=...>` s'exécutera dans le navigateur du responsable.
C'est une faille XSS stockée classique, à traiter **avant** le branchement de la source réelle
(utiliser `textContent`, ou échapper systématiquement).

---

### 6.7 🟡 MOYEN — Les emojis ne s'affichent pas dans le PDF téléchargé

`expPDF()` (`index.html:748-766`) écrit `${lbl.emoji} ${lbl.nom}` avec la police standard
`helvetica` de jsPDF, qui utilise un encodage mono-octet. Les emojis (⚠️ 🛒 🚚) ne font pas partie
de ce jeu de caractères et seront rendus de façon incorrecte ou absents. Les caractères accentués
courants passent, mais le `ò` de « Benicarlò » est à vérifier.

**Correction :** retirer les emojis du PDF, ou embarquer une police Unicode (`addFont`).

---

### 6.8 🔵 MINEUR

| # | Constat | Emplacement |
|---|---|---|
| a | `maximum-scale=1.0` bloque le zoom sur mobile — non conforme WCAG 1.4.4 | `index.html:5` |
| b | La feuille d'impression force `#app{display:block!important}` : imprimer depuis l'écran de connexion produit une page vide | `index.html:216-225` |
| c | Attributs de style dupliqués en ligne alors que la classe `.av` les porte déjà | `index.html:274` |
| d | Les 4 dépôts génériques (« Dépôt A », « Ville A »…) sont des libellés de gabarit non renseignés | `index.html:403-406` |
| e | L'`id` du dépôt est interpolé dans un `onclick` sans échappement — sans risque avec la config actuelle, fragile par principe | `index.html:534` |
| f | Le sens de tri n'est pas réinitialisé au changement d'onglet côté affichage | `index.html:605-612` |

---

## 7. Sécurité

### 7.1 Contrôle d'accès

Il n'existe **aucun contrôle d'accès réel**. Le filtre par domaine e-mail est un contrôle
d'interface, contournable en modifiant une variable depuis la console du navigateur, ou tout
simplement en saisissant une adresse plausible. Comme les données sont livrées avec la page, elles
sont de toute façon lisibles dans le code source sans même passer par l'écran de connexion.

Tant que le contenu reste fictif, l'exposition est nulle. **Le jour où des données commerciales
réelles (clients, volumes, commandes) seront intégrées, une authentification serveur devient
obligatoire.**

### 7.2 Stockage local

`sessionStorage` conserve `{email, depot}` en clair (`index.html:558`). Peu sensible en soi, mais à
revoir en même temps que l'authentification (jeton à durée de vie limitée plutôt qu'objet libre).

### 7.3 Chaîne d'approvisionnement

Les trois bibliothèques sont chargées depuis un CDN tiers **sans contrôle d'intégrité (SRI)** :
si cdnjs était compromis ou usurpé, du code arbitraire s'exécuterait dans la page.

Par ailleurs, **`xlsx` 0.18.5 est une version ancienne visée par un avis de sécurité connu**
(pollution de prototype, corrigée dans les versions ultérieures de SheetJS ; la bibliothèque n'est
d'ailleurs plus distribuée par ses auteurs via npm). `jsPDF` 2.5.1 date également de plusieurs
versions.

**Recommandations :** ajouter les attributs `integrity` et `crossorigin`, mettre à jour les
versions, et — pour une application interne — héberger les bibliothèques localement plutôt que de
dépendre d'un réseau externe.

---

## 8. Accessibilité et ergonomie

**Points positifs :** contrastes de texte élevés sur fond sombre, cibles tactiles généreuses,
libellés explicites, états vides informatifs, retours d'action systématiques.

**Points à corriger :**

- Les cartes de dépôt, le badge de dépôt et les en-têtes de colonne triables sont des `div` / `th`
  porteurs d'un `onclick`, **sans `role`, sans `tabindex`, sans gestion clavier** : ces
  interactions sont inaccessibles au clavier et aux lecteurs d'écran.
- Les `<label class="field-label">` ne sont **pas associés** aux champs (`for` / `id` absents).
- L'information est parfois portée **par l'emoji seul** (icônes de dépôt, indicateurs), sans
  équivalent textuel.
- **Le zoom est désactivé** (§ 6.8-a).
- Le code couleur des badges de casse (vert / ambre / rouge) n'est **pas doublé** d'un indicateur
  non chromatique.
- Aucune région `aria-live` : les toasts et les changements de tableau ne sont pas annoncés.

---

## 9. Qualité et maintenabilité

| Élément | État |
|---|---|
| Tests automatisés | ❌ Aucun |
| Intégration continue | ❌ Aucune |
| README / documentation | ❌ Absent |
| Licence | ❌ Absente |
| `.gitignore` | ❌ Absent |
| Linter / formateur | ❌ Aucun |
| Découpage en modules | ❌ Fichier unique de 794 lignes |
| Gestion de version des données | ❌ Données en dur dans le code |

**Historique Git :** 8 commits, tous du 4 août 2026, dont 6 intitulés « Add files via upload »
(dépôts via l'interface web GitHub). Les diffs sont massifs — jusqu'à 909 lignes ajoutées en une
fois — ce qui rend l'historique inexploitable pour comprendre l'évolution du projet ou revenir en
arrière de façon ciblée.

**Points positifs à souligner :** nommage cohérent et en français, commentaires de section clairs,
variables CSS bien organisées, structure de configuration (`DEPOTS`, `CFG`, `LABELS`) qui isole
correctement le paramétrable du code — un vrai bon réflexe.

---

## 10. Ce qui manque pour une mise en production

1. **Une source de données réelle.** C'est le cœur manquant du projet. Aujourd'hui, alimenter
   l'application signifie éditer du JavaScript à la main. Il faut une ingestion automatique depuis
   la boîte e-mail (API Gmail / IMAP), un analyseur des messages de déclaration, et un stockage.
2. **Une authentification serveur** (OAuth avec le compte professionnel serait le plus naturel,
   puisque les utilisateurs disposent déjà d'une adresse `@gps-glass.com`).
3. **Une base de données** avec un identifiant de dépôt normalisé, remplaçant le rapprochement par
   mots-clés.
4. **La normalisation des données à l'ingestion** : destinations, clients et références issus de
   texte libre (§ 5.1).
5. **Une gestion des droits** : aujourd'hui tout utilisateur voit tous les dépôts.
6. **La correction des anomalies** du § 6, en commençant par la 6.1.

---

## 11. Feuille de route recommandée

### Immédiat — rendre le prototype démontrable (~1 h)

1. Corriger le filtre accentué (§ 6.1) — **sans quoi rien ne s'affiche**.
2. Corriger le tri par date (§ 6.3).
3. Rafraîchir les dates du jeu de test pour que « semaine » et « mois » renvoient des lignes (§ 6.4).
4. Retirer les emojis du PDF (§ 6.7).
5. Rétablir le zoom mobile (§ 6.8-a).

### Court terme — fiabiliser (~1 jour)

6. Échapper les données injectées dans le DOM (§ 6.6).
7. Ajouter les attributs SRI et mettre à jour les bibliothèques (§ 7.3).
8. Rendre les interactions accessibles au clavier (§ 8).
9. Corriger le traitement des fuseaux horaires (§ 6.5).
10. Ajouter un `README.md` décrivant l'installation, la configuration des dépôts et le mode d'emploi.

### Moyen terme — industrialiser

11. Extraire le CSS et le JS dans des fichiers séparés, découper le JS en modules.
12. Externaliser les données dans un `data.json`, préalable naturel au branchement d'une API.
13. Mettre en place un backend : ingestion e-mail, base de données, API, authentification.
14. Ajouter des tests sur la logique métier (filtrage, tri, statistiques) et une CI GitHub Actions.

---

## 12. Conclusion

Le projet démontre une **vraie compréhension du besoin métier** et un **sens du design remarquable
pour un fichier unique écrit sans framework** : trois écrans cohérents, un thème sombre maîtrisé,
un responsive à trois points de rupture, trois formats d'export et des statistiques pertinentes,
le tout en 794 lignes.

Sa limite est celle de sa catégorie : c'est une **maquette fonctionnelle**, pas une application.
Les données sont recopiées à la main, la connexion est décorative, et un défaut d'accentuation d'un
seul caractère suffit aujourd'hui à empêcher tout affichage.

La correction du § 6.1 est **l'action la plus rentable du projet** : quelques lignes de code qui
transforment une page qui n'affiche rien en une démonstration convaincante.

---

*Rapport généré par analyse statique du code source. Les comportements décrits aux § 6.1, 6.3 et
6.4 ont été vérifiés par exécution.*
