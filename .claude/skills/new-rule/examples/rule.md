# Exemple annoté — fichier de règle (MEM002)

Copie de `packages/core/src/rules/mem002.ts`, annotée. C'est le modèle à
suivre pour `packages/core/src/rules/<id>.ts`.

```ts
// (1) Imports : uniquement des types du core et le parseur interne.
//     Jamais de `node:fs`, de réseau, de `process`, ni d'import de la CLI.
import type { Finding, Rule } from '../types.js';
import { parseMarkdown } from '../markdown.js';

// (2) Commentaire d'une ligne : id + ce que la règle signale.
/** MEM002 : signale chaque section sans contenu propre ni sous-section directe. */
export const mem002: Rule = {
  // (3) Id stable : il apparaît dans les sorties, les snapshots et l'E2E.
  //     Ne jamais le renommer après publication.
  id: 'MEM002',
  // (4) Sévérité tranchée à l'étape Spec : 'error' | 'warn' | 'info'.
  severity: 'warn',
  // (5) Doc d'une ligne : décrit le problème, pas la correction.
  docs: 'Une section de CLAUDE.md est vide',

  // (6) `check` est pur : il reçoit le contenu en mémoire et retourne des
  //     findings, sans rien lire ni écrire. Même entrée → même sortie.
  check({ file }) {
    // (7) Réutiliser parseMarkdown() : lignes normalisées (CRLF ≡ LF, BOM
    //     retiré), prologue, sections avec `headingLine` 1-indexée, blocs
    //     de code exclus de la détection d'en-têtes.
    const { sections } = parseMarkdown(file.content);
    const findings: Finding[] = [];

    // (8) Itération dans l'ordre du document → ordre des findings déterministe.
    for (const [index, section] of sections.entries()) {
      // (9) Cas limite tranché à l'étape Spec, commenté dans le code :
      // Une section immédiatement suivie d'une sous-section plus profonde est
      // un conteneur légitime, pas une section vide.
      const next = sections[index + 1];
      if (next !== undefined && next.level > section.level) {
        continue;
      }
      if (section.lines.every((line) => line.trim() === '')) {
        findings.push({
          // (10) `ruleId` et `severity` répètent ceux de la règle.
          ruleId: 'MEM002',
          severity: 'warn',
          // (11) Message exact validé à l'étape Spec ; guillemets « » et
          //      partie variable entre eux.
          message: `La section « ${section.title} » est vide.`,
          // (12) Finding localisé : `line` 1-indexée (ici l'en-tête).
          //      Une règle non localisée (cf. MEM003) omet simplement `line`.
          line: section.headingLine,
        });
      }
    }
    // (13) Aucun problème → tableau vide (jamais `undefined`).
    return findings;
  },
};
```

## Encart — règle à seuil (modèle MEM001)

Un seuil s'expose en **constante exportée**, non configurable, pour que les
tests (et `rules.test.ts`) s'y réfèrent au lieu de dupliquer la valeur :

```ts
/** Seuil de longueur en lignes, non configurable en v1 (exporté pour les tests). */
export const MAX_LINES = 200;

// … dans check() :
const count = parseMarkdown(file.content).lines.length;
if (count <= MAX_LINES) {   // pile au seuil : pas de finding
  return [];
}
return [{
  ruleId: 'MEM001',
  severity: 'warn',
  message: `Le fichier compte ${count} lignes (maximum recommandé : ${MAX_LINES}).`,
  line: MAX_LINES + 1,      // première ligne au-delà du seuil
}];
```
