# Sandcastle x skills de Matt Pocock: exploration

Note d'exploration interne (SPHERE-DI), rédigée le 2026-09-04 sur la branche
`claude/sandcastle-skill-pocok-exploration-5sm0en`. Elle répond à une question:
qu'est-ce que ce repo de skills, qu'est-ce que Sandcastle, et qu'est-ce qu'on
obtient en branchant les deux.

Sources: le repo lui-même pour la partie skills; pour Sandcastle, son `README.md`
et son `package.json` récupérés le 2026-09-04 sur
[mattpocock/sandcastle](https://github.com/mattpocock/sandcastle)
(paquet `@ai-hero/sandcastle`, v0.12.0, MIT). Attention à l'homonymie: il existe
d'autres projets nommés "Sandcastle" sans rapport (par exemple un paquet PyPI
`sandcastle-ai`). Celui qui nous intéresse est la librairie TypeScript de Matt
Pocock, cohérente avec ce repo de skills.

## 1. En une phrase

Ce sont deux couches orthogonales du même problème (faire produire du vrai
travail d'ingénierie à un agent), et elles ne se recouvrent pas:

- **Les skills disent ce que l'agent fait pendant une session**: le processus,
  la discipline, les questions posées, les artefacts produits.
- **Sandcastle dit où et comment la session tourne**: isolation, stratégie de
  branche git, parallélisme, exécution AFK ("away from keyboard") sans humain
  devant.

L'une sans l'autre reste utile. Ensemble, elles couvrent la chaîne complète:
cadrer avec un humain, découper en tickets, puis lâcher N agents isolés sur le
frontier de tickets prêts.

## 2. Le repo de skills

### Organisation

37 `SKILL.md` répartis en buckets sous `skills/`:

| Bucket | Nombre | Statut |
| --- | --- | --- |
| `engineering/` | 18 | promu (livré dans le plugin) |
| `productivity/` | 7 | promu |
| `misc/` | 4 | gardé, non promu |
| `in-progress/` | 8 | bêta publique, non promu |
| `deprecated/` | 0 | vide par convention (un skill retiré est supprimé) |

Les 25 skills promus ont trois obligations couplées, décrites dans
[`CLAUDE.md`](../../CLAUDE.md): une entrée dans le `README.md` racine, une entrée
dans le tableau `skills` de `.claude-plugin/plugin.json`, et une page de doc
humaine dans `docs/<bucket>/<nom>.md`. Les non promus n'ont aucune des trois.
C'est un invariant simple à vérifier en revue et facile à casser en ajoutant un
skill.

### L'axe qui structure tout: qui peut invoquer

Voir [`.agents/invocation.md`](../invocation.md). Chaque skill est soit:

- **user-invoked**: seul l'humain le déclenche en tapant son nom
  (`disable-model-invocation: true` côté Claude Code,
  `policy.allow_implicit_invocation: false` dans `agents/openai.yaml` côté
  Codex). Sa `description` s'adresse à un humain. Ce sont les orchestrateurs.
- **model-invoked**: le modèle peut y aller tout seul. Sa `description` garde
  les formulations de déclenchement. Ce sont les disciplines réutilisables.

Invariant important pour la suite: **un skill user-invoked ne peut jamais être
appelé par un autre skill**, y compris via l'outil Skill. Les dépendances entre
skills s'écrivent explicitement (`Call the Skill tool with "grilling"`), jamais
par un lien relatif vers le fichier d'un autre skill.

### Le flux principal (routeur `ask-matt`)

`skills/engineering/ask-matt/SKILL.md` est la carte. Le chemin nominal, idée
vers production:

1. `/grill-with-docs` interroge l'humain jusqu'à ce que l'idée soit nette, et
   laisse une trace: `CONTEXT.md` (glossaire du domaine) et des ADR. Sans
   répertoire de travail, c'est `/grill-me`, stateless.
2. Détour optionnel par `/prototype` quand une question demande une réponse
   exécutable, avec `/handoff` dans les deux sens. Le prototype est conservé
   comme source primaire sur une branche `prototype/<nom>`.
3. Si le chantier tient dans une session: `/implement` directement. Sinon
   `/to-spec` puis `/to-tickets`, qui produit des tickets tracer bullet, chacun
   déclarant ses **blocking edges**, donc un graphe de tâches et non une liste.
4. `/implement` construit chaque ticket en pilotant `/tdd` en interne, puis
   clôt avec `/code-review` (deux axes, Standards et Spec, en sous-agents
   parallèles) avant de committer.

Trois on-ramps rejoignent ce flux: `/triage` (issues entrantes),
`/diagnosing-bugs` (bug dur, exige d'abord une boucle de feedback rouge sur ce
bug précis), `/wayfinder` (chantier trop gros pour une session, résolu comme une
carte de decision tickets). En dessous tourne une couche de vocabulaire:
`/domain-modeling` pour le langage du domaine, `/codebase-design` pour la forme
des modules (deep modules, seams).

Contrainte explicite du routeur: garder les étapes 1 à 3 dans **une seule
fenêtre de contexte**, sans compact ni clear avant `/to-tickets`, et ne pas
dépasser la "smart zone" (environ 150k tokens). Chaque `/implement` repart
ensuite d'un contexte neuf. Retenir ce point: c'est exactement la couture où
Sandcastle devient utile.

### Distribution

Deux entrées, deux philosophies (voir
[`.agents/adr/0002-ship-as-a-claude-code-plugin.md`](../adr/0002-ship-as-a-claude-code-plugin.md)):
le plugin Claude Code (`claude plugins install mattpocock-skills`), bundle en
lecture seule qui se met à jour tout seul, ou `npx skills@latest add
mattpocock/skills`, qui copie des fichiers éditables dans le projet. Installer
les deux donne chaque skill en double. Un plugin Codex natif est différé: le
manifeste Codex n'accepte qu'un seul chemin de skills, incompatible avec la
structure en buckets.

Le prérequis opérationnel: `/setup-matt-pocock-skills`, à lancer une fois par
repo. Il écrit la configuration que `to-tickets`, `to-spec`, `triage`,
`implement` et `code-review` lisent ensuite (issue tracker, labels de triage,
emplacement des docs), typiquement sous `docs/agents/`.

## 3. Sandcastle

Librairie TypeScript, `npm i -D @ai-hero/sandcastle`, plus une CLI
(`npx @ai-hero/sandcastle init`) qui génère un dossier `.sandcastle/`
(Dockerfile, `.env`, template de prompt).

API centrale, un appel:

```typescript
import { run, claudeCode } from "@ai-hero/sandcastle";
import { docker } from "@ai-hero/sandcastle/sandboxes/docker";

await run({
  agent: claudeCode("claude-opus-4-8"),
  sandbox: docker(),
  promptFile: ".sandcastle/prompt.md",
});
```

Ce qu'il faut retenir pour notre usage:

- **Providers d'agents**: `claudeCode()`, `codex()`, `cursor()`, `pi()`,
  `opencode()`, `copilot()`, avec le niveau d'effort et le mode de permission en
  options. Les providers resumables exposent `resume()` et `fork()`.
- **Sandboxes**: `docker`, `podman`, `vercel`, `daytona`, `no-sandbox` (les
  sous-chemins d'export du paquet le confirment). Docker et Podman sont en
  bind-mount, Vercel en microVM isolée.
- **Stratégies de branche**: `head` (écrit directement dans le working
  directory), `merge-to-head` (worktree temporaire fusionné à la fin), ou
  `{ type: "branch", branch: "agent/fix-42" }` (commits sur une branche nommée,
  worktree réutilisé et fast-forward si on relance). C'est la pièce qui rend le
  parallélisme sûr.
- **Prompts dynamiques**: un `promptFile` accepte la substitution `{{CLE}}` via
  `promptArgs`, et l'expansion shell `` !`commande` `` exécutée dans la sandbox
  (par exemple `` !`gh issue view 42` ``). `{{SOURCE_BRANCH}}` et
  `{{TARGET_BRANCH}}` sont injectés d'office.
- **Sortie structurée**: `Output.object({ tag, schema: z.object(...) })` extrait
  et valide du JSON émis par l'agent, avec `maxRetries`. Contrainte:
  `maxIterations: 1`. C'est ce qui permet de brancher un agent dans un pipeline
  et de décider en TypeScript plutôt qu'en lisant du texte.
- **Sandbox longue durée**: `createSandbox()` garde le conteneur chaud pour
  plusieurs `sandbox.run()` successifs (implémenter puis relire, par exemple),
  et `sandbox.exec("npm test")` permet de faire tourner les checks en dehors de
  l'agent, donc de gater sur un code de sortie plutôt que sur une affirmation
  du modèle.
- **Worktree indépendant**: `createWorktree()` puis `wt.interactive()` pour une
  session humaine, `wt.run()` pour l'AFK, sur la même branche. C'est le pont
  entre les deux mondes.
- **Hooks de cycle de vie** (`npm install`, copie de `.env`, paquets système),
  **completion signals** et timeouts d'inactivité.

## 4. Les coutures entre les deux

C'est le point intéressant: les skills produisent exactement les objets que
Sandcastle sait consommer.

### 4.1 Le graphe de tickets est un plan d'exécution parallèle

`/to-tickets` ne rend pas une liste mais un graphe avec des blocking edges, donc
un **frontier** de tickets prêts à tout instant. Sandcastle sait lancer un agent
par ticket, chacun sur `{ type: "branch", branch: "agent/<ticket>" }`, dans son
propre worktree et son propre conteneur. La correspondance est directe: le
frontier calculé côté skills devient la liste des `run()` à lancer en parallèle,
et chaque ticket terminé rouvre le frontier.

Le skill `in-progress/implement-spec` décrit déjà ce pattern en prose
(implementer subagents, chacun dans son worktree et sa branche, merger subagent,
frontier rouvert à chaque complétion, `/code-review` à la fin, PR unique). Il le
fait aujourd'hui avec des sous-agents du harness, sans isolation réelle ni
contrôle hôte. Sandcastle est la version exécutable de ce même dessin, avec de
vraies frontières de processus et un résultat typé.

### 4.2 Un `/implement` par ticket, en AFK

La règle de contexte du routeur ("chaque `/implement` repart d'un contexte
neuf") est littéralement ce que fait un `run()`: un conteneur neuf, un contexte
neuf, un ticket. Le `promptFile` porte le pointeur plutôt que le contenu:

```markdown
Ticket #{{ISSUE_NUMBER}}:

!`gh issue view {{ISSUE_NUMBER}}`

Call the Skill tool with "implement" and build this ticket on {{TARGET_BRANCH}}.
```

Deux prérequis, faciles à oublier:

1. Les skills doivent être **dans l'image** (installer le plugin ou
   `npx skills add` au build du Dockerfile), sinon l'agent en sandbox n'a rien à
   invoquer.
2. La configuration écrite par `/setup-matt-pocock-skills` doit être
   **commitée** dans le repo, sinon `implement` et `code-review` s'arrêtent en
   demandant à l'humain de lancer le setup, ce qu'aucun humain ne verra dans un
   run AFK.

### 4.3 Relire dans le même conteneur chaud

`/code-review` tourne sur un diff depuis un point fixe et rend deux axes. Avec
`createSandbox()`, on enchaîne implémentation puis revue sans repayer le
démarrage, et avec `Output.object()` on récupère les findings en JSON validé par
zod. On peut alors gater: pas de PR tant qu'un finding bloquant reste ouvert, et
`sandbox.exec("npm test")` en garde-fou objectif à côté.

### 4.4 Les skills qui ne peuvent pas partir en AFK

C'est la limite nette, et elle tombe pile au même endroit que la couture du
flux. Tout ce qui est un interrogatoire ou une décision humaine ne se
parallélise pas: `/grill-me`, `/grill-with-docs`, `/grilling`, `/wayfinder`,
la phase de quiz de `/to-tickets` et `/triage`, `/to-questionnaire`, `/wizard`
(par définition, il existe pour les étapes que seul un humain peut faire).

Donc: **avant `/to-tickets`, humain et interactif; après, isolable et
parallélisable**. `wt.interactive()` puis `wt.run()` sur la même branche est
exactement ce découpage, offert par l'API.

### 4.5 Autres correspondances

- `/prototype` conserve son résultat sur une branche `prototype/<nom>`: c'est
  `branchStrategy: { type: "branch" }` sans autre effort.
- `/research` tourne déjà en agent d'arrière-plan et rend un fichier Markdown
  cité: candidat naturel à un `run()`, sous réserve d'accès réseau dans la
  sandbox.
- `/resolving-merge-conflicts` devient utile en régime parallèle: N branches
  d'agents qui fusionnent vers une branche de PR produisent mécaniquement des
  conflits, et ce skill est la discipline pour les résoudre par intention.
- `/handoff` et `in-progress/claude-handoff` traitent le passage de témoin entre
  sessions; Sandcastle traite le passage de témoin entre **processus**. Les deux
  se complètent, ils ne se remplacent pas.

## 5. Ce que ça vaut pour nous, et à quel prix

Pistes concrètes, par ordre de coût croissant:

1. **Rien à faire côté agents**: utiliser les skills tels quels, en interactif,
   dans un repo où `/setup-matt-pocock-skills` a tourné. Bénéfice immédiat sur
   l'alignement (`grill-with-docs`) et sur le vocabulaire (`CONTEXT.md`).
2. **Un `run()` sur un seul ticket**, en Docker, pour valider l'image, l'auth et
   les hooks. C'est le vrai coût de démarrage de Sandcastle: fabriquer une image
   où le projet build et teste.
3. **Le frontier en parallèle** sur une spec réelle, avec merge vers une branche
   de PR unique, puis `/code-review` en sortie structurée comme garde-fou.

Risques et points de vigilance:

- **Secrets**: `.sandcastle/.env` contient des tokens OAuth ou des clés API, et
  les hooks peuvent copier des `.env` dans le conteneur. À traiter comme du
  secret CI, pas comme un fichier de config.
- **Permissions**: les modes du type `acceptEdits` sont ce qui rend l'AFK
  possible et ce qui rend l'isolation obligatoire. Le sandbox n'est pas un
  confort, c'est la contrepartie.
- **API en 0.x**: `@ai-hero/sandcastle` est en 0.12.0. Épingler la version,
  s'attendre à des ruptures.
- **Coût en tokens**: N agents en parallèle multiplient la facture, et un ticket
  mal découpé la multiplie encore. Le découpage vertical de `/to-tickets` est ce
  qui rend le parallélisme rentable, pas un détail de forme.
- **Conflits de merge** proportionnels au parallélisme, voir 4.5.

## 6. Questions ouvertes

- Quelle image de base pour nos repos, et est-ce qu'on installe les skills au
  build (plugin en lecture seule) ou qu'on les copie (fork éditable)?
- Est-ce qu'on garde ce fork de skills aligné sur l'upstream, ou est-ce qu'on
  diverge avec nos propres skills maison?
- Politique réseau dans la sandbox: `/research` et `gh` en ont besoin, le reste
  non. Deux images plutôt qu'une?
- Est-ce que `implement-spec` (bêta, non promu) devient notre orchestrateur, ou
  est-ce qu'on l'écrit en TypeScript avec Sandcastle et qu'on laisse les skills
  au niveau du ticket?
