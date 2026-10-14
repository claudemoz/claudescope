# Exemple annoté — test de règle (MEM002)

Copie de `packages/core/src/rules/mem002.test.ts`, annotée. C'est le modèle à
suivre pour `packages/core/src/rules/<id>.test.ts`.

```ts
import { describe, it, expect } from 'vitest';
import { mem002 } from './mem002.js';

// (1) Helper local : appelle la règle seule sur un contenu en mémoire.
//     Pas de fichier sur disque, pas de moteur : on teste la règle isolée
//     (le passage par analyze() est couvert par « Gate 1 » dans rules.test.ts).
function check(content: string) {
  return mem002.check({ file: { path: 'CLAUDE.md', content } });
}

// (2) describe : « <ID> — <nom court> ».
describe('MEM002 — section vide', () => {
  // (3) Cas propre : aucune violation → toEqual([]).
  it('ne signale rien quand toutes les sections ont du contenu', () => {
    expect(check('# Un\ntexte\n## Deux\nautre texte\n')).toEqual([]);
  });

  // (4) Cas fautif localisé : toEqual sur le finding COMPLET (ruleId,
  //     severity, message exact, line). Fige le contrat de sortie.
  it('signale une section vide, localisée à son en-tête', () => {
    expect(check('# Un\ntexte\n## Vide\n\n# Trois\ntexte\n')).toEqual([
      {
        ruleId: 'MEM002',
        severity: 'warn',
        message: 'La section « Vide » est vide.',
        line: 3,
      },
    ]);
  });

  // (5) Plusieurs violations : un finding chacune, dans l'ordre du document.
  //     Ici on projette sur `line` : le finding complet est déjà figé en (4).
  it('émet un finding par section vide', () => {
    const findings = check('# A\n# B\ntexte\n# C');
    expect(findings.map((f) => f.line)).toEqual([1, 4]);
  });

  // (6) Variantes de la condition (lignes blanches = vide).
  it('considère comme vide une section ne contenant que des lignes blanches', () => {
    expect(check('# Un\n \n\t\n')).toHaveLength(1);
  });

  // (7) Cas limite : section conteneur — tranché à l'étape Spec.
  it('ne signale pas une section conteneur suivie d’une sous-section plus profonde', () => {
    expect(check('## A\n### B\ntexte\n')).toEqual([]);
  });

  it('signale une section vide suivie d’un en-tête de même niveau ou moins profond', () => {
    const findings = check('## A\n## B\ntexte\n### C\n# D\ntexte\n');
    expect(findings.map((f) => f.message)).toEqual([
      'La section « A » est vide.',
      'La section « C » est vide.',
    ]);
  });

  // (8) Cas limite : fin de fichier sans newline final.
  it('évalue la dernière section du fichier comme les autres', () => {
    expect(check('# Un\ntexte\n## Fin')).toEqual([
      {
        ruleId: 'MEM002',
        severity: 'warn',
        message: 'La section « Fin » est vide.',
        line: 3,
      },
    ]);
  });

  // (9) Cas limites obligatoires : prologue sans en-tête et fichier vide.
  it('ne signale jamais le prologue ni un fichier sans en-tête', () => {
    expect(check('du texte sans en-tête\n')).toEqual([]);
    expect(check('')).toEqual([]);
  });
});
```

## Pour une règle à seuil

Importer la constante plutôt que dupliquer la valeur, et tester **pile au
seuil** (pas de finding) et **seuil + 1** (finding) :

```ts
import { mem001, MAX_LINES } from './mem001.js';

const lignes = (n: number) => Array.from({ length: n }, () => 'x').join('\n');

it('accepte un fichier pile au seuil', () => {
  expect(check(lignes(MAX_LINES))).toEqual([]);
});

it('signale un fichier au-delà du seuil', () => {
  expect(check(lignes(MAX_LINES + 1))).toEqual([
    {
      ruleId: 'MEM001',
      severity: 'warn',
      message: `Le fichier compte ${MAX_LINES + 1} lignes (maximum recommandé : ${MAX_LINES}).`,
      line: MAX_LINES + 1,
    },
  ]);
});
```

## Pour une règle non localisée (modèle MEM003)

Le finding attendu **n'a pas** de clé `line` ; `toEqual` le vérifie (une
`line: undefined` explicite serait aussi acceptée par `toEqual`, mais ne
l'écris pas dans la règle).
