---
name: new-rule
description: Ajoute une nouvelle règle de lint à claudoscope (fichier de règle, test, enregistrement, fixtures, snapshots, E2E, dogfooding, vérification). À utiliser dès qu'on demande de créer, ajouter ou implémenter une règle claudoscope (ex. « ajoute une règle MEM004 qui… »). Argument : la description de la règle.
---

# new-rule — ajouter une règle à claudoscope

Description de la règle demandée : $ARGUMENTS

Suis les étapes **dans l'ordre**. Les modèles de référence sont
[`examples/rule.md`](examples/rule.md) (fichier de règle) et
[`examples/test.md`](examples/test.md) (test unitaire). Lis-les avant l'étape 2.

## Étape 0 — Spec (bloquante)

**N'écris aucun code avant que chaque point ci-dessous soit tranché.** Pour
chaque point que la description ne tranche pas *explicitement*, pose la
question à l'utilisateur (AskUserQuestion, jusqu'à 4 questions par appel). Ne
déduis rien en silence, n'invente aucun texte de message.

1. **Id et sévérité** : id stable au format des règles existantes (`MEM001`…),
   et sévérité `error` / `warn` / `info`.
2. **Doc d'une ligne** (`docs`) et **texte exact du message** du finding, y
   compris les parties variables (`« ${section.title} »`, compteurs…).
3. **Localisation** : le finding porte-t-il une `line` ? Si oui, laquelle
   (en-tête de section, première ligne fautive, ligne du seuil…) ?
4. **Seuils** : valeurs exactes, exposées en constantes exportées
   (`export const MAX_LINES = 200;`), non configurables.
5. **Cas limites** : comportement attendu sur un fichier vide, un prologue
   sans en-tête, une section conteneur (suivie d'une sous-section plus
   profonde), du contenu dans un bloc de code.
6. **Fixture** : quelle modification de `fautif/` (et `avertissements/` si
   `warn`) déclenche la règle, et à quelle ligne.

Récapitule la spec tranchée en quelques lignes avant de passer à l'étape 1.

## Étape 1 — Fichier de règle

Créer `packages/core/src/rules/<id en minuscules>.ts` (ex. `mem004.ts`) qui
exporte un objet `Rule` (`packages/core/src/types.ts`) — voir
`examples/rule.md`.

- Réutiliser `parseMarkdown()` (`packages/core/src/markdown.ts`) : `lines`,
  `prologue`, `sections` (`level`, `title`, `headingLine`, `lines`), blocs de
  code déjà exclus de la détection d'en-têtes.
- S'il manque une information structurelle, **étendre `parseMarkdown()`**
  plutôt que reparser dans la règle, et compléter `markdown.test.ts`. Le
  parseur rapporte la structure sans la juger : la décision reste à la règle.
- Seuils en `export const`, documentés d'un commentaire `/** … */`.

## Étape 2 — Test unitaire

Créer `packages/core/src/rules/<id>.test.ts` — voir `examples/test.md`. Au minimum :

- un cas propre → `toEqual([])` ;
- un cas fautif vérifié par **`toEqual` sur le finding complet** (`ruleId`,
  `severity`, `message`, `line` si localisé) ;
- les cas limites tranchés à l'étape 0 (fichier vide, prologue sans en-tête,
  section conteneur, valeur pile au seuil et seuil + 1).

## Étape 3 — Enregistrement

- `packages/core/src/rules/index.ts` : importer la règle et l'ajouter **à la
  fin** du tableau `rules` (l'ordre fixe garantit le déterminisme de
  l'agrégation ; ne jamais réordonner).
- `packages/core/src/rules/rules.test.ts` :
  - ajouter l'id à la fin de la liste du test « ordre fixe » ;
  - test « Gate 1 » : faire déclencher la nouvelle règle par le contenu fautif
    et ajouter son finding attendu **en dernière position**. Si le contenu
    change, recalculer les findings existants (le compteur de lignes de MEM001,
    les `line` de MEM002).

## Étape 4 — Fixtures

Fichiers : `packages/cli/fixtures/{sain,fautif,avertissements}/CLAUDE.md`.

- Règle `warn` (ou `info`) : la faire déclencher dans `fautif/` **et** `avertissements/`.
- Règle `error` : **uniquement** dans `fautif/`. `avertissements/` ne doit
  contenir que des `warn` (son test attend exit 0 par défaut).
- `sain/` reste **sans aucun finding**.
- **Modifier des lignes existantes plutôt qu'en ajouter** : `fautif/` et
  `avertissements/` font 204 lignes, ce qui fixe le finding MEM001, et insérer
  une ligne décale les `line` de MEM002. Garder le nombre de lignes identique.

## Étape 5 — Snapshots et E2E

- `packages/cli/src/fixtures.test.ts` : ajuster les titres qui comptent les
  findings (« les trois findings » → « les quatre findings »…). **Les clés de
  snapshot dérivent des titres** : un titre renommé crée une nouvelle entrée
  et rend l'ancienne obsolète.
- Régénérer : `pnpm run build && pnpm exec vitest run -u`.
- **Relire le diff** de `packages/cli/src/__snapshots__/fixtures.test.ts.snap`
  (`git diff`) : seuls doivent apparaître le ou les nouveaux findings et les
  clés renommées ; les findings existants gardent leurs lignes ; aucune entrée
  obsolète ne doit subsister.
- `packages/cli/src/e2e.test.ts` : ajouter le `ruleId` à la liste attendue
  (ordre du tableau `rules`) et ajuster le titre (« les trois règles »…).

## Étape 6 — Dogfooding

Le `CLAUDE.md` du dépôt doit passer sans aucun finding. Deux garde-fous le
vérifient : `packages/core/src/rules/rules.test.ts` (via `analyze()`) et
`packages/cli/src/dogfooding.test.ts` (via `runScan`, `--fail-on warn`).

Si la nouvelle règle se déclenche sur le `CLAUDE.md` du dépôt : **corriger le
CLAUDE.md, pas la règle**. Note chaque correction pour le rapport.

## Étape 7 — Vérification

```bash
pnpm run typecheck
pnpm run test
pnpm run lint
node packages/cli/dist/index.js scan CLAUDE.md   # doit ne rien afficher
```

Puis contrôler les invariants ci-dessous. Tout échec se corrige avant de conclure.

## Invariants du core (à vérifier avant de conclure)

- **Pureté, zéro I/O** : la règle n'importe ni `node:fs`, ni réseau, ni
  `process` (`grep -rE "node:|process\." packages/core/src/rules` ne doit
  remonter que `rules.test.ts`).
- **Déterminisme** : même entrée, même sortie (pas de `Date`, d'aléatoire,
  d'ordre d'itération non garanti).
- **Ordre fixe** des règles : nouvelle règle en fin de tableau.
- **Id stable, sévérité, doc d'une ligne** ; `ruleId` et `severity` du finding
  identiques à ceux de la règle.
- **Dépendance à sens unique** : `packages/core` n'importe jamais
  `packages/cli` ni `claudoscope`.
- **Aucune modification des fichiers analysés** : `check` ne fait que lire
  `file.content`.

## Pièges

- Le dogfooding est vérifié **deux fois** (core et CLI) : les deux tests doivent passer.
- Les **noms de snapshots dépendent des titres** des tests : renommer = régénérer et relire.
- **Ajouter des lignes** à une fixture décale les findings MEM001 et MEM002.
- La fixture `avertissements` ne doit contenir que des **`warn`**.
- La fixture `sain` ne doit produire **aucun finding**.

## Fin — rapport

Termine par un message qui contient :

1. **Spec retenue** : id, sévérité, docs, message, localisation, seuils.
2. **Fichiers créés ou modifiés.**
3. **Vérification** : résultat de `pnpm run typecheck`, `pnpm run test`, `pnpm run lint`.
4. **Rapport de dogfooding** : findings éventuels de la nouvelle règle sur le
   `CLAUDE.md` du dépôt et corrections apportées (ou « aucun finding »).
5. **Rappel** : la liste des règles dans `README.md` et `design-doc.md` est à
   tenir à jour (à proposer, pas à faire d'office).
