# Veille Emploi n8n — Analyse factory & stratégie d'extraction

**Subject** : Deux workflows n8n (W1 feeds, W2 emails) en MVP fonctionnel sur n8n 2.15.0 — l'objectif est de transformer le starter-kit en vraie "automation factory" et de préparer l'extraction des workflows matures vers leur propre projet standalone.

**Solution recommandée** : (1) refactorer trois patterns dupliqués vers des sub-workflows réutilisables avant d'ajouter W3 ; (2) externaliser tous les magic strings/numbers vers env vars n8n ; (3) pour l'extraction, suivre un plan en 5 phases avec export workflow JSON + Qdrant snapshot + OAuth re-grant côté cible.

---

## 1. Quality review — anti-patterns introduits cette session

### 1.1 Monster Code nodes

`Format Digest` W1 fait **10 433 caractères** (ligne 173-391 du dump). Il mélange :
- parsing/validation du retour LLM
- mode de dégradation (FAILED/DEGRADED/NO_NEW/OK)
- helpers `esc/htmlShell/jobCard/watchLine/buildOutput`
- composition markdown
- composition HTML inline
- assemblage subject/to/binary

C'est devenu **non-testable** : pour valider un changement (typo dans le HTML, nouveau champ score), tu dois rerun tout le workflow.

**Pattern souhaitable** : décomposer en 3 nodes :
- `Score Parser` (Code, ~30 lignes) : ne fait que `JSON.parse(content)` + mode + scores
- `Render Markdown` (Code, ~40 lignes) : `scored[]` → string md
- `Render HTML Email` (Code, ~60 lignes) : `scored[]` → subject + html (réutilisable W1+W2)

Idéalement, `Render HTML Email` devient un **sub-workflow** (`Execute Sub-workflow` node) consommable par W1, W2 et futurs workflows.

### 1.2 Duplication W1 ↔ W2

Trois zones quasi-identiques mais déjà divergentes :

| Pattern | W1 | W2 | Divergence |
|---|---|---|---|
| Filter keyword (roles inclus + exclus) | `Normalize & Filter` lignes 82-110 | `Filter by Keywords` lignes 647-675 | W2 a `VAGUE_LOC` autorisé, W1 non — `loc === ''` seulement |
| Prepare Scoring (prompt + format batch + requestBody) | `Prepare Scoring` 124-170 | `Prepare Scoring` 688-713 | Prompts légèrement différents, MAX = 40 dupliqué, modèle hardcodé |
| Format Digest HTML (jobCard, watchLine, htmlShell) | 173-391 | 715-808 | W1 a `_cv_score` badge, W2 non |

Risque concret : tu vas oublier de propager le prochain ajustement keywords vers W2 (ou inverse). C'est déjà arrivé cette session — j'ai dû te faire patcher W2 après W1.

**Pattern souhaitable** : un sub-workflow `Score & Filter Jobs` qui prend `{jobs[], profile}` en input et retourne `{filtered, scored, top, watch, subject, html, md}`. W1 et W2 l'appellent avec leur batch.

### 1.3 Magic numbers et strings éparpillés

| Constante | Lieu | Type |
|---|---|---|
| `MAX = 40` | W1+W2 Prepare Scoring | batch limit |
| `substring(0, 500)` | W1 Normalize & Filter | desc truncate |
| `substring(0, 300)` | Prepare Scoring | desc display |
| `substring(0, 6000)` | W2 Build Extract Request | body truncate |
| `substring(0, 2000)` | W1 Dedup + Embed (embedding input) | embed truncate |
| `timeout: 60000 / 30000 / 300000 / 180000` | 4 endroits différents | ms |
| `http://host.docker.internal:11434` | 2 endroits | Mac-specific |
| `http://qdrant:6333` | 2 endroits | docker network |
| `'you@example.com'` | 2 endroits | TO_EMAIL |
| `'/data/shared/veille/jobs/'` et `'/data/shared/veille/emails/'` | 2 endroits | paths |
| `'llama3.2:3b'` (model) | 3 endroits | model name |
| `'bge-m3'` (EMBED_MODEL) | W1 Dedup | model name |
| `'qwen3-coder:480b'` | W2 Build Extract | model name |

**Pattern souhaitable** : extraire vers **n8n environment variables** (`$env.X`). Ça permet de tester en dev et déployer ailleurs sans patcher le workflow.

> ⚠️ Correction post-analyse (vérifiée empiriquement) : contrairement à ce que cette section supposait, `$env.X` n'est PAS accessible par défaut. Il faut `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, qui débloque `$env` **partout** (Code nodes *et* expressions, de façon identique — pas « expressions seulement »). Voir `n8n/conventions.md §4.1`.

À ajouter dans `docker-compose.yml` :
```yaml
environment:
  - OLLAMA_LOCAL_URL=http://host.docker.internal:11434
  - OLLAMA_CLOUD_URL=https://ollama.com
  - QDRANT_URL=http://qdrant:6333
  - VEILLE_OUTPUT_DIR=/data/shared/veille-emploi
  - VEILLE_TO_EMAIL=you@example.com
  - VEILLE_BATCH_MAX=40
  - VEILLE_EMBED_MODEL=bge-m3
  - VEILLE_SCORE_MODEL_LOCAL=llama3.2:3b
  - VEILLE_EXTRACT_MODEL_CLOUD=qwen3-coder:480b
```

Dans Code node : `const QDRANT = $env.QDRANT_URL;`. Dans expressions HTTP : `={{ $env.OLLAMA_LOCAL_URL }}/api/chat`.

### 1.4 onError pas systématique

Seuls les `HTTP Request` ont `onError: continueRegularOutput`. Les **Code nodes critiques** (Format Digest, Match CV, Normalize & Filter) n'en ont pas. Si Match CV plante (Qdrant down), tout le workflow casse et l'exec est perdu.

**Pattern souhaitable** : `onError: continueRegularOutput` sur les Code nodes qui ont déjà un mode de dégradation (Format Digest), et au minimum `onError: stopWorkflow` explicite sur ceux qui n'en ont pas.

### 1.5 Observabilité absente

Tu n'as aucun moyen de savoir si W1 ou W2 a tourné/échoué sans regarder l'UI n8n exec list. Pas de Slack alert sur failure, pas de métriques cumulées (combien d'offres en 30 jours, combien de scores top moyen).

**Pattern souhaitable** : un sub-workflow `Notify Pipeline Status` qui prend `{workflowName, status, errorMsg, stats}` et envoie sur Slack/Discord/email avec template uniforme. Branché sur le `errorWorkflow` setting de chaque workflow + appelé à la fin de chaque pipeline.

### 1.6 Inconsistances de version (intra-workflow)

- Gmail : v2.2 (Fetch IDs/Full W2) vs v2.1 (Send Digest W1+W2). Le v2.2 a un schéma payload différent pour `get` (encodé/décodé). Pas dramatique mais source de confusion.
- HTTP Request : v4.4 vs v4.2 (Fetch France Travail uniquement) — uniformiser.

**Action** : upgrader tous les Gmail à v2.2 et tous les HTTP Request à v4.4 (effet probable nul mais cohérence).

### 1.7 Pas de tests

Aucun usage de `pinData` ni du nouveau **`test_workflow` MCP tool** (disponible depuis n8n 2.15.0, doc context7 confirmée). Tu dois rerun à chaque ajustement, ce qui coûte des appels Ollama Cloud.

**Pattern souhaitable** : pinner les outputs de `Fetch Email Full` (3 emails) et `Score LLM` avec un dataset golden, puis lancer `n8n_test_workflow` après chaque modif côté Parse/Filter/Format. Test gratuit, instantané.

### 1.8 Bug factuel détecté

`Match CV` (W1, ligne 526) utilise `this.helpers.httpRequest` direct alors que `Dedup + Embed + Index` (W1 ligne 416) a un wrapper `httpJSON` qui détecte `$helpers || this.helpers`. Le code dans Match CV était écrit avant de découvrir le pattern (cf. [[feedback_n8n_code_node_helpers]]). À aligner.

### 1.9 Discrepancies memory ↔ code

La memory précise `nomic-embed-text` (1024 dims) comme embed model, mais le code W1 utilise **`bge-m3`**. Le memory est obsolète sur ce point. À corriger dans la memory.

---

## 2. Version & docs — état n8n 2.15.0

### 2.1 typeVersions utilisés vs current

| Node | Utilisé | Latest n8n 2.15.0 | État |
|---|---|---|---|
| `scheduleTrigger` | v1.3 | v1.3 | À jour |
| `httpRequest` | v4.4 (et v4.2 pour FT) | v4.4+ | À jour, FT à upgrader |
| `code` | v2 | v2 | À jour |
| `merge` | v3.2 | v3+ | À jour (docs montrent encore v2.1 dans tutoriels) |
| `readWriteFile` | v1.1 | v1.1 | À jour |
| `rssFeedRead` | v1.2 | v1.2 | À jour |
| `gmail` | v2.2 (fetch) et v2.1 (send) | v2.2 | Send à upgrader v2.1 → v2.2 |

Conclusion : **on est bien sur n8n v2 (au sens Code node v2)**, et la quasi-totalité des nodes sont à jour. Deux toutes petites incohérences à corriger (FT v4.2 → v4.4, Gmail Send v2.1 → v2.2), sans effet bloquant.

### 2.2 Features manquées dans notre setup actuel

| Feature | Disponible depuis | Notre usage | Recommandation |
|---|---|---|---|
| `test_workflow` (MCP tool) | n8n 2.15.0 | **0** | Adopter pour itérer sans burn Ollama Cloud |
| `pinData` (UI + JSON) | < 2.0 | **0** | Pinner outputs LLM Cloud sur emails golden |
| `Execute Sub-workflow` node | < 2.0 | **0** | Factorer Score/Filter/Format en sub-workflows |
| `errorWorkflow` (workflow setting) | < 2.0 | **0** | Notifier Slack sur exec error |
| `EXECUTIONS_DATA_SAVE_*` env vars | < 2.0 | défaut (= tout sauvé) | Désactiver `MANUAL_EXECUTIONS` et `ON_SUCCESS=none` pour ne pas remplir la DB |
| `staticData` (workflow scope) | < 2.0 | **0** | Stocker stats cumulées (compteur runs, dernière exec) |
| `evaluations` (avec dataset) | n8n 2.x | **0** | Pour tester qualité scoring LLM avec dataset golden |
| LangChain nodes (Ollama Chat Model, Embeddings, etc.) | n8n 1.x AI nodes | partiellement (Ollama langchain cred) | Pourrait remplacer nos HTTP Request POST → /api/chat |

### 2.3 Patterns dépréciés ou sous-optimaux qu'on utilise

- **`Buffer.from(...)`** : OK, supporté dans sandbox vm2 mais pas documenté officiellement comme stable. Si on switche vers `n8n.eval.useNewerJsRuntime`, pas garanti.
- **Foreach manuel dans Code node** au lieu de `SplitInBatches v3` + Loop : OK pour 40 items, mais quand on passera à 200+, on aura intérêt à batcher pour permettre la reprise sur erreur partielle.

### 2.4 Documentation n8n — accès

Tu peux **toujours** vérifier l'état de l'art via :
```bash
# via context7 MCP (déjà branché)
mcp__context7__resolve-library-id "n8n"
mcp__context7__query-docs /n8n-io/n8n-docs "<question>"
```

Et via le **n8n-mcp** local que tu as déjà :
```
search_nodes / get_node / validate_node / search_templates / get_template
```

Donc oui, je peux vérifier en temps réel les schémas de nodes — la question est juste de prendre l'habitude de le faire avant d'écrire le code, pas après.

---

## 3. Pattern factory — ce qu'il faut mettre en place

### 3.1 Structure de repo cible (en restant dans le starter-kit)

```
self-hosted-ai-starter-kit/
├── docker-compose.yml          (existant — à enrichir avec env vars)
├── shared/
│   ├── veille/                 (outputs, current)
│   └── cv/                     (sources, current)
├── n8n/
│   └── workflows-exports/      (NEW: snapshots versionnés)
│       ├── 2026-05-28_veille-emploi-daily.json
│       └── 2026-05-28_veille-emploi-emails-daily.json
├── n8n/
│   └── sub-workflows/          (NEW: réutilisables)
│       ├── score-and-filter-jobs.json
│       ├── render-job-digest.json
│       └── notify-pipeline-status.json
└── .claude/
    ├── analysis/               (ce rapport)
    └── docs/
        └── n8n-conventions.md  (NEW: nos conventions)
```

### 3.2 Conventions à formaliser

Petit doc `n8n-conventions.md` à écrire (~1 page) :
- **Code node mode** : `runOnceForAllItems` par défaut. `runOnceForEachItem` réservé aux nodes qui appellent un service externe par item.
- **Helpers HTTP** : toujours `this.helpers.httpRequest({...})` avec fallback `$helpers` si besoin de portabilité.
- **Output shape** : toujours `[{ json: {...}, binary?: {...} }]`. Pas de keys au top-level à côté de `json`.
- **Constants** : jamais hardcoded, toujours via `$env.X`. Liste des env vars dans le doc.
- **onError** : `continueRegularOutput` sur Code nodes critiques avec mode de dégradation interne ; `stopWorkflow` partout ailleurs.
- **Naming** : nodes en français snake-case, workflows en `<Domaine> (<freq>)`.

### 3.3 Trois priorités d'amélioration

| Priorité | Action | Effort | Impact |
|---|---|---|---|
| **P0** | Externaliser magic strings vers env vars dans docker-compose.yml + Code nodes refactor | 2h | Préparation extraction + diminution dette |
| **P1** | Décomposer Format Digest W1 en 2-3 nodes + créer sub-workflow `Render HTML Email` partagé W1+W2 | 3h | Test/maintien facile, fin de la duplication |
| **P2** | Implémenter `errorWorkflow` + sub-workflow `Notify Pipeline Status` | 2h | Observability minimale |

À faire **avant** d'ajouter un W3 (ex. canal Slack communautaire) — sinon on triplera la dette.

---

## 4. Stratégie d'extraction — clarification du modèle

### 4.1 Le bon modèle (correction d'un postulat initial)

Ce que je décrivais en v1 de ce rapport (extraction = nouveau docker-compose + nouveau n8n cible) était faux. Le modèle correct :

- **Le repo `self-hosted-ai-starter-kit` EST la factory.** Il porte l'infra (docker-compose), les conventions (`CLAUDE.md`), les sub-workflows partagés (à venir), les feedback memories factory-level (gotchas n8n, patterns réutilisables).
- **L'instance n8n est unique et permanente.** Tous les workflows passés/présents/futurs tournent sur le même n8n + Postgres. Pas de duplication d'infra par projet.
- **Les workflows vivent dans Postgres.** Ils ne sont pas dans le repo (sauf snapshots versionnés optionnels).
- **"Extraire un workflow"** = déménager **uniquement les fichiers projet-spécifiques** hors du repo : memory file projet, brief docs, inputs (CV PDF), outputs accumulés. Le workflow continue de tourner dans la même instance n8n.
- **La factory accumule les améliorations.** Les gotchas découverts sur un projet (ex: pattern Gmail get singular, code fence cloud LLM, OAuth Gmail Production) sont **promus en feedback memories factory-level** pour bénéficier aux projets futurs.

### 4.2 Qu'est-ce qui est factory vs projet ?

| Type | Fichiers | Décision |
|---|---|---|
| **Infra & conventions** | `docker-compose.yml`, `.env.example`, `.mcp.json`, `.claude/settings.json`, `CLAUDE.md` | **Factory** — reste dans le repo |
| **Feedback memories transférables** | `feedback_n8n_*`, `feedback_security_*`, `feedback_gmail_oauth_publish`, etc. | **Factory** — restent dans `~/.claude/projects/.../memory/` |
| **Profil user** | `user_profile_job_search.md` | **Factory** (transversal au user, pas au projet) |
| **Sub-workflows partagés** (à créer) | `n8n/sub-workflows/render-html-digest.json`, etc. | **Factory** — sub-workflows réutilisables entre projets |
| **Project memory** | `project_veille_emploi_workflow.md` | **Projet** — déménage |
| **Project briefs / analyses** | `.claude/analysis/veille-emploi-n8n-factory-analysis.md` (ce rapport) | **Projet** — déménage |
| **Inputs projet** | `shared/cv/*.pdf` | **Projet** — déménage |
| **Outputs projet** | `shared/veille/jobs/`, `shared/veille/emails/` | **Projet** — déménagent (ou archivés) |
| **Workflow JSON (snapshot)** | Optionnel : `n8n/workflows-exports/<id>.json` versionnable | **Projet** — si pratiqué, déménage |

### 4.3 Workflow "graduation" — checklist quand veille-emploi sera mature

À jouer quand W2 sera stable depuis ~1 mois et que tu veux libérer la factory. Destination : `~/projets/n8n/veille-emploi/` (le sous-dossier `n8n/` regroupe les projets de stack n8n, laisse de la place à `~/projets/python/...` etc.).

1. **Snapshot final du workflow JSON** (utile pour rollback / lecture hors-ligne) :
   ```bash
   mkdir -p ~/projets/n8n/veille-emploi/{workflows,qdrant,memory}
   for id in q2crCIIaJhSsXfRx b8erSgFcfJ32W7vx; do
     curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" \
       "http://localhost:5678/api/v1/workflows/$id" \
       > ~/projets/n8n/veille-emploi/workflows/$id.json
   done
   ```
2. **Snapshot Qdrant** (collections projet-spécifiques) :
   ```bash
   curl -X POST "http://localhost:6333/collections/cv_profile/snapshots" \
     > ~/projets/n8n/veille-emploi/qdrant/cv_profile.snapshot.meta.json
   curl -X POST "http://localhost:6333/collections/job_listings/snapshots" \
     > ~/projets/n8n/veille-emploi/qdrant/job_listings.snapshot.meta.json
   # télécharger les fichiers .snapshot via GET /collections/{name}/snapshots/{name}
   ```
3. **Déménager les fichiers projet** :
   ```bash
   mv shared/cv ~/projets/n8n/veille-emploi/cv
   mv shared/veille ~/projets/n8n/veille-emploi/veille
   mv .claude/analysis/veille-emploi-n8n-factory-analysis.md \
      ~/projets/n8n/veille-emploi/analysis.md
   ```
4. **Déménager la project memory** :
   ```bash
   mv ~/.claude/projects/-Users-adrien-projets-self-hosted-ai-starter-kit/memory/project_veille_emploi_workflow.md \
      ~/projets/n8n/veille-emploi/memory/
   mv ~/.claude/projects/-Users-adrien-projets-self-hosted-ai-starter-kit/memory/feedback_veille_emploi_manual_run.md \
      ~/projets/n8n/veille-emploi/memory/
   ```
   Et **retirer** ces lignes de `MEMORY.md`.
5. **Vérifier que toutes les leçons transférables sont en factory** avant de déménager — la project memory ne doit contenir que de l'état projet, pas des gotchas réutilisables. (Promotion déjà faite cette session : Ollama Cloud naming, code fence, OAuth publish, API PUT, credential types.)
6. **Conserver les workflows actifs dans n8n.** Aucune action sur Postgres — l'instance n8n garde tout, continue à tourner. Si tu veux désactiver pour économiser : `POST /api/v1/workflows/{id}/deactivate`. Si tu veux vraiment supprimer : `DELETE /api/v1/workflows/{id}` (mais avec snapshot déjà fait).
7. **Retirer les sections projet-spécifiques de `CLAUDE.md`** si tu en as ajoutées (actuellement, `CLAUDE.md` est purement factory, aucune section veille emploi spécifique — rien à faire).

Résultat : la factory revient à un état "vierge" (infra + conventions + feedbacks transférables), prête à accueillir un nouveau projet, avec **toutes les leçons techniques du projet précédent intactes**.

### 4.4 Pré-requis avant de "graduer" veille emploi

Idéalement, avant de déménager :
1. **Refactor P0 (env vars)** — sinon le workflow JSON snapshot porte des paths/URLs hardcodés qui rendent le snapshot moins portable si un jour tu rejoues sur une autre machine.
2. **Refactor P1 (sub-workflows partagés)** — ces sub-workflows restent en factory (réutilisables par futurs projets). Les extraire après les avoir créés, c'est facile. Les créer après l'extraction, ça oblige à recréer.
3. **Stabilité W2** : minimum 2-3 semaines d'observation sans intervention.
4. **Memory cleanup** : déjà fait cette session (12 → 14 memory files, project memory passée de 11.6 KB à 5.2 KB, promotions de 5 lessons transférables).

### 4.5 Note sur "réutiliser la factory pour un nouveau projet"

Quand tu démarreras le prochain projet (ex: Slack #jobs, ou un workflow pour une autre tâche), la factory aura **automatiquement** :
- Les conventions de `CLAUDE.md` (LOCAL-FIRST, MCP, sécurité)
- Tous les patterns n8n appris (~7 feedback memories factory aujourd'hui)
- L'infra opérationnelle (docker-compose, MCP n8n branché)
- Le profil user

Donc démarrer un nouveau projet = créer **un seul nouveau fichier** `project_<nom>_workflow.md` dans la memory. Tout le reste est déjà là.

---

## Code References

- `q2crCIIaJhSsXfRx` (W1) — `Format Digest` Code node 10433c, à décomposer
- `q2crCIIaJhSsXfRx` (W1) — `Normalize & Filter` ligne 82-110, duplication à factorer avec W2 `Filter by Keywords` ligne 647-675
- `q2crCIIaJhSsXfRx` (W1) — `Dedup + Embed + Index` ligne 411 `helpersInfo` probe : code de debug à nettoyer maintenant que [[feedback_n8n_code_node_helpers]] est stable
- `b8erSgFcfJ32W7vx` (W2) — `Build Extract Request` model hardcodé `qwen3-coder:480b` ligne 584, à externaliser
- `docker-compose.yml` ligne 13-29 — section env n8n, où ajouter les nouvelles env vars
- `CLAUDE.md` ligne 52-72 — politique LOCAL-FIRST à conserver dans le projet extrait
- Memory `project_veille_emploi_workflow.md` — discrepancy embed model `nomic-embed-text` (memory) vs `bge-m3` (code W1)

## Recommendation Rationale

Le starter-kit a un usage hybride : (a) factory de prototypage où tu construis des workflows neufs, (b) hébergement de prod pour les workflows matures qui ne valent pas encore la peine d'être extraits. Cette double casquette est saine **tant que** la factory ne devient pas un dépotoir.

L'écart entre "ça marche" et "ça scale comme factory" est dans **trois pratiques** que tu n'as pas formalisées : externalisation des constantes, sub-workflows partagés, observabilité minimale. Ces trois pratiques sont **prérequis** à toute extraction propre — sans elles, tu vas dupliquer les bugs dans chaque projet extrait.

Pour la version : on est à jour. Pour les patterns dépréciés : aucun bloquant, juste des features modernes pas exploitées (`test_workflow`, `pinData`, `Execute Sub-workflow`, `errorWorkflow`). Les adopter coûte ~1 jour cumulé et te sauvera des semaines.

Pour l'extraction : **ne pas se précipiter**. W1 est en prod, W2 est jeune. La fenêtre idéale d'extraction est après le refactor P0+P1 et après 2-4 semaines de stabilité W2 — pas avant.
