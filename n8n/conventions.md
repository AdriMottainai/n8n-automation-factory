# n8n Production Conventions — Automation Factory

Conventions et patterns pour construire des workflows n8n **robustes, debuggables, maintenables et exportables** dans cette factory (n8n self-hosted v2.15.0, Code node v2, stack Mac M1 + Docker). Synthèse de la doc officielle n8n + retours communauté production, validée 2026-05-29.

Quand tu démarres un nouveau workflow ou refactores un workflow existant, **lis la section concernée avant** de coder, pas après.

---

## Philosophie en 5 lignes

1. **Local-first** : pas d'API payante (cf. CLAUDE.md). Ollama local par défaut, Cloud justifié uniquement.
2. **Aucune valeur métier hardcodée** : tout passe par `$env.X` ou est calculé.
3. **Aucune exécution silencieuse** : `errorWorkflow` configuré + logs JSON sur stdout.
4. **Aucune action irréversible sans clé d'idempotence** : gates Postgres/Qdrant/Remove Duplicates avant tout write/send.
5. **Aucune modification in-place sur un workflow actif en prod** : clone → modify → swap.

---

## 1. Architecture & maintenabilité

### 1.1 Naming conventions strictes

**Workflows** — `[Domain] Action – Detail – vN`
- `[Veille] Daily Job Digest – v2`
- `[RAG] Index Shared Docs – v1`
- `[Util] Send Notif Slack – v1`

**Nodes** — verbe + objet métier, jamais le défaut (`Code1`, `HTTP Request`).
- `Fetch RSS Feed (TechCrunch)` au lieu de `HTTP Request`
- `Format Digest Markdown` au lieu de `Code`
- `Filter Jobs by Location` au lieu de `IF`

**Credentials** — `<env>-<service>-<scope>` (préfixe env **critique** pour éviter qu'un import dev écrase prod).
- `prod-ollama-cloud`, `local-ollama-host`, `prod-gmail-veille`, `dev-postgres-veille`

**Pourquoi** : un workflow à 30 nodes avec `Code1...Code7` est mort à 3 mois. La `versionId` change à chaque save, le **nom est le seul ancrage stable** pour les MCP tools (`search_nodes`) et la recherche humaine.

### 1.2 Code node — limites de taille et décomposition

| Lignes | Action |
|---|---|
| < 80 | OK inline |
| 80–250 | Décomposer en 2-4 Code nodes nommés métier |
| > 250 | Extraire en sub-workflow `[Util] ...` |

Pas de limite technique dure (le moteur tient 1000+ lignes), mais 3 problèmes pratiques émergent dès ~100 lignes : éditeur Monaco lent, `Test step` couvre tout (impossible d'isoler un bug), pas de tests unitaires possibles.

**Exemple concret** — décomposer un Format Digest 10 KB :
- `Parse LLM Output` (Code, ~40 lignes) — extraction JSON, fallback regex
- `Validate Schema` (Code, ~30 lignes) — typage strict, coercion
- `Build HTML Helpers` (Code, ~50 lignes) — fonctions de rendu pures
- `Compose Markdown Digest` (Code, ~40 lignes) — assemblage final

Si plusieurs workflows partagent une étape → sub-workflow `[Util] Parse LLM JSON – v1`.

### 1.3 Modes Code node — signature d'intention

| Mode | Quand | Intention signalée |
|---|---|---|
| `runOnceForAllItems` | Opérations sur collection : agrégation, dédup, tri, lookups inter-items | "J'ai besoin d'état partagé" |
| `runOnceForEachItem` | Transformations 1:1 pures, validation, enrichissement item-local | "Cet item est indépendant" |

**Anti-pattern** : `runOnceForAllItems` qui fait `items.map(x => { ...HTTP call... })` séquentiellement. C'est `runOnceForEachItem` + `retryOnFail`. Le mode `forAllItems` qui ne fait que `.map()` est un signal de refactor.

### 1.4 Sub-workflows réutilisables

**Quand factoriser** : tout bloc utilisé dans 2+ workflows, ou tout sous-graphe de >10 nodes encapsulant une responsabilité unique.

**Pattern correct** :
- **Côté trigger** (sub-workflow appelé) : `Execute Sub-workflow Trigger` en mode `Define using fields below` avec schéma typé (`emailBody: string`, `userId: number`). Activer `Attempt to convert types`. C'est le contrat d'entrée.
- **Côté sortie** : finir par `Edit Fields (Set)` en mode `Define using JSON` avec schéma stable (`{ status, result, errors[] }`). C'est le contrat de sortie.
- **Côté appelant** : `Execute Sub-workflow` en mode `From list` (inputs auto-complétés dans le formulaire).

**À éviter** :
- Sub-workflow appelé via HTTP Request sur l'API n8n locale (double sérialisation, perte du debug visuel — réservé au cross-instance).
- Sub-workflow en mode `Accept all data` (annule le typage, inputs invisibles dans l'appelant).
- Cascades > 3 niveaux (stack trace illisible).

### 1.5 Découpage IF/Switch vs sub-workflow

| Situation | Outil |
|---|---|
| Branche binaire avec 2-5 nodes de chaque côté | `If` |
| 3-7 branches mutuellement exclusives, courtes | `Switch` |
| Branches longues (>10 nodes) ou réutilisables | Sub-workflow par branche |
| Branches qui partagent leurs derniers nodes (merge) | `If`/`Switch` + `Merge` |
| Logique récursive | Sub-workflow self-call |

**Pattern orchestrateur + workers** pour un workflow >15 nodes : principal devient diagramme métier lisible (~10 nodes), workers `[Domain:util] ...` testables isolément via `n8n_test_workflow`.

---

## 2. Robustesse

### 2.1 `errorWorkflow` — non négociable avant activation

Avant d'`active=true` un workflow, configurer dans **Workflow Settings → Error Workflow** un workflow dédié qui démarre par un node `Error Trigger`. Un seul Error Workflow `_global_error_handler` réutilisé par tous tes workflows.

Payload reçu par l'Error Trigger :
```json
{
  "execution": { "id": "231", "url": "https://...", "lastNodeExecuted": "...", "error": { "message": "...", "stack": "..." }, "mode": "manual" },
  "workflow": { "id": "1", "name": "..." }
}
```

**Template Slack/Discord minimal** :
```
[FAIL] {{ $json.workflow.name }}
Node: {{ $json.execution.lastNodeExecuted }}
Error: {{ $json.execution.error.message }}
Open: {{ $json.execution.url }}
```

Cas particulier : erreur au trigger node (ex: Schedule Trigger activation) → pas d'`execution.id/url`, structure différente avec `trigger: { name: "WorkflowActivationError" }`. Le template doit gérer les deux (`$json.execution?.url ?? 'no execution saved'`).

### 2.2 `onError` au niveau node — explicite, jamais implicite

| Setting | Quand |
|---|---|
| `stopWorkflow` (défaut) | Nodes critiques dont l'échec invalide tout le run |
| `continueErrorOutput` | **Recommandé** pour HTTP Request, Gmail, OpenAI, AI Agent — crée une 2e sortie qu'on branche vers logging/fallback |
| `continueRegularOutput` | À éviter (masque le bug, propage des items vides) |

Dans le JSON workflow :
```json
{
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.4,
  "onError": "continueErrorOutput",
  "retryOnFail": true,
  "maxTries": 5,
  "waitBetweenTries": 5000
}
```

### 2.3 Defensive Code nodes

```js
// 1. Guard vide en début de Code node
const items = $input.all();
if (!items.length) {
  return [{ json: { skipped: true, reason: 'no input' } }];  // sentinelle, pas []
}

// 2. Accès défensifs
const email = item.json?.user?.contact?.email ?? null;
if (!email) return [{ json: { error: 'missing email', raw: item.json } }];

// 3. JSON.parse toujours wrapped
function safeParse(raw, fallback = null) {
  if (typeof raw !== 'string') return raw ?? fallback;
  try { return JSON.parse(raw); } catch { return fallback; }
}

// 4. Préserver pairedItem pour traçabilité downstream
return $input.all().map((item, i) => ({
  json: { ...item.json, processed: true },
  pairedItem: { item: i }
}));

// 5. Helper HTTP : this.helpers.httpRequest (pas $helpers), require interdit
```

Voir `feedback_n8n_code_node_helpers` pour les helpers et le hash BigInt FNV-1a (pas de `require('crypto')`).

### 2.4 Timeouts à 3 couches

n8n n'a **AUCUN timeout par défaut** (`EXECUTIONS_TIMEOUT=-1`). En prod c'est inacceptable.

**docker-compose.yml** :
```yaml
environment:
  - EXECUTIONS_TIMEOUT=900            # 15 min default
  - EXECUTIONS_TIMEOUT_MAX=3600       # 1h plafond
```

**Par workflow** (Settings → Timeout Workflow) :
- Webhook synchrone : 25s (client raccroche à 30s)
- Pipeline RAG / batch nocturne : 20 min
- LLM Cloud long-context : 5 min

**Par node HTTP Request** (Options → Timeout, ms) :
- Ollama local : 120000 (M1 8GB peut prendre 30-60s)
- Ollama Cloud : 180000
- Qdrant / Postgres locaux : 5000
- APIs externes (Gmail, RSS) : 30000

**Piège** : `EXECUTIONS_TIMEOUT_MAX < EXECUTIONS_TIMEOUT` → cap silencieux. Toujours `MAX >= TIMEOUT`.

### 2.5 Rate limiting et retry

**Batching natif HTTP Request** (Options → Batching) — préféré :
```
Items per Batch: 10
Batch Interval (ms): 1000
```

**Retry on Fail** (Settings du node) :
```json
"retryOnFail": true,
"maxTries": 5,
"waitBetweenTries": 5000
```
Pour exponential backoff custom : `Loop Over Items` + `Wait` avec expression `Math.pow(2, $('Loop').context.currentRunIndex) * 1000`.

**Respecter `Retry-After`** :
```
[HTTP] (continueErrorOutput)
  → [IF: status==429 && headers.retry-after exists]
      → [Wait: {{ $json.headers['retry-after'] }} s]
      → retry
```

**Anti-patterns** :
- `Loop Over Items` batchSize=1 sans Wait → pas de throttle, juste sérialisation
- Retry sur 4xx non-429 (400/401/403) → gaspillage, fail définitif

### 2.6 Idempotency — patterns du plus simple au plus robuste

Tous les triggers réseau sont at-least-once. n8n ne fournit **aucune garantie exactly-once**. Sans dédup explicite : retry Stripe → double charge, retry Gmail → double envoi.

| Pattern | Cas |
|---|---|
| `Idempotency-Key` header passé à l'API | API supporte (Stripe, Square) — zéro state côté n8n |
| `Remove Duplicates` node, scope `workflow` | Polling/RSS simple — persistance dans DB n8n, single-instance OK |
| `staticData` curseur (`$getWorkflowStaticData('global')`) | Polling timestamped — ⚠️ **uniquement en mode actif/trigger**, pas en manuel ; race conditions en haute fréquence |
| **Gate Postgres `INSERT ON CONFLICT DO NOTHING`** | Pattern de production robuste, distribué, auditable |
| Gate Qdrant (query par `payload.source_hash`) | Pipelines RAG/embeddings |

**Gate Postgres** (le plus solide) :
```sql
CREATE TABLE idempotency_keys (
  event_id TEXT PRIMARY KEY,
  workflow_id TEXT NOT NULL,
  seen_at TIMESTAMPTZ DEFAULT now()
);
```
```
[Trigger] → [Postgres: INSERT ON CONFLICT DO NOTHING RETURNING event_id]
         → [IF: rows_affected = 1] → branche "nouveau" → side effects
                                  → branche "déjà vu" → no-op
```

**Pièges** :
- `{{ $now }}` ou `Date.now()` comme clé d'idempotence → garantie de doublons
- `staticData` en exécution manuelle de test → vide, faux positifs/négatifs
- `Remove Duplicates` AVANT une étape qui peut échouer en milieu de batch → items marqués "vus" sans avoir été traités. Solution : dédup **côté action** (gate juste avant le write/send), pas en amont.

### 2.7 OAuth refresh / Google credentials

Sur n8n self-hosted, les creds OAuth2 Google (Gmail, Drive, Sheets, Calendar) **expirent tous les 7 jours** tant que ton app GCP est en `Testing`. Cf. `feedback_gmail_oauth_publish` pour la procédure.

Pattern de monitoring proactif (workflow utilitaire daily) :
```
[Schedule daily 06:00]
  → [Gmail: get labels (ping read-only)]
  → [IF: error 401/403] → [Slack alert: "Re-auth Gmail required"]
```
Détecte l'expiration J-2 avant que le workflow critique casse en silence.

---

## 3. Observabilité & debug

### 3.1 Logs custom dans Code nodes — `CODE_ENABLE_STDOUT=true`

Par défaut, `console.log()` dans un Code node **n'apparaît pas** dans `docker logs n8n` (uniquement console DevTools en test UI). Pour le faire ressortir côté serveur :

```yaml
# docker-compose.yml
environment:
  - CODE_ENABLE_STDOUT=true
  - N8N_LOG_FORMAT=json
  - NO_COLOR=1
```

Helper logs JSON structurés :
```js
const log = (level, event, data = {}) => console.log(JSON.stringify({
  ts: new Date().toISOString(),
  level,
  workflow: $workflow.name,
  exec: $execution.id,
  event,
  ...data,
}));

log('info', 'job_fetched', { count: items.length, source: 'remotive' });
log('warn', 'rate_limit', { retry_after: 60 });
```

Puis grep depuis le host :
```bash
docker logs n8n --since 9h | jq 'select(.event == "veille_done")'
```

### 3.2 `EXECUTIONS_DATA_SAVE_*` — équilibre rétention vs DB size

```yaml
environment:
  - EXECUTIONS_DATA_SAVE_ON_ERROR=all
  - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all          # sinon impossible de savoir si tournait
  - EXECUTIONS_DATA_SAVE_ON_PROGRESS=true        # post-mortem possible si crash
  - EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=false # exec UI pollue pendant dev
  - EXECUTIONS_DATA_PRUNE=true
  - EXECUTIONS_DATA_MAX_AGE=168                  # 7 jours
  - EXECUTIONS_DATA_PRUNE_MAX_COUNT=5000
```

**Tradeoff clé** : `ON_PROGRESS=true` double le volume de `execution_data` mais permet le debug post-mortem rapide. Si SQLite/Postgres gonfle au-delà de quelques GB, passer `ON_PROGRESS=false` et `ON_SUCCESS=none` (rester sur logs stdout + staticData pour savoir que ça a tourné).

### 3.3 Logs n8n core — env vars

```yaml
environment:
  - N8N_LOG_LEVEL=info                  # error / warn / info / debug
  - N8N_LOG_OUTPUT=console              # console / file (comma-sep)
  - N8N_LOG_FORMAT=json                 # JSON parse-able (recommandé prod)
  - N8N_LOG_CRON_ACTIVE_INTERVAL=60     # log les crons armés toutes les 60 min
```

### 3.4 `pinData` — itérer sans burn les APIs

Le pinning sauvegarde l'output JSON d'un node et le réutilise au lieu de re-exécuter. Activable uniquement sur les nodes à une seule sortie main.

**Cas d'usage** :
- Trigger webhook : ping une fois, pin, rejouer 50 fois sans relancer le webhook externe
- HTTP qui appelle une API payante : pin la réponse représentative, itérer sur les nodes en aval sans burner le quota
- Tests de régression : pin un set de jobs known-good, valider que le parser ne casse pas

**UI** : ouvrir un node → icône pin en haut à droite du panneau output. Sauvé dans `workflow.pinData` (visible en DB `workflow_entity.pinData`), **versionné avec le workflow** donc commitable.

**Garde-fou** : pinData est ignoré en production (trigger réel ou schedule), n'agit qu'en mode test. Aucun risque de geler la prod.

### 3.5 MCP `test_workflow` (n8n 2.15.0+)

Lance une exécution test avec un pinData fourni, **sans déclencher les triggers réels** et sans appeler les credentials externes.

```json
{
  "workflowId": "abc123",
  "pinData": { "Webhook": [{ "json": {...} }], "HTTP Remotive": [{ "json": {...} }] },
  "triggerNodeName": "Schedule Trigger"
}
```

**Workflow d'usage** :
1. `prepare_test_pin_data` MCP → retourne un JSON Schema indiquant les nodes à pinner
2. Construire le pinData avec un sample réaliste (exporte d'une exec OK passée)
3. `test_workflow` avec ce pinData → valide la logique sans burner d'APIs

### 3.6 `staticData` — stats cumulées workflow-scoped

`$getWorkflowStaticData('global')` retourne un objet persisté en DB entre exécutions. **Limitations** : (a) ne persiste qu'en mode actif/trigger (pas en manuel), (b) non fiable en haute fréquence (race conditions).

```js
// Dernier node "Update stats"
const stats = $getWorkflowStaticData('global');
stats.runs = (stats.runs || 0) + 1;
stats.last_run_at = new Date().toISOString();
stats.last_items_count = $input.all().length;
stats.cumulative_items = (stats.cumulative_items || 0) + $input.all().length;

// Rolling history sur 30 derniers
stats.history = (stats.history || []).slice(-29);
stats.history.push({ ts: stats.last_run_at, items: stats.last_items_count, ok: true });

return [{ json: stats }];
```

**Exposer sans UI** : workflow tiers Webhook qui lit via `Execute Workflow` ou lit directement `SELECT "staticData" FROM workflow_entity WHERE id = ...` en Postgres.

### 3.7 Healthcheck endpoints

| Endpoint | Signification |
|---|---|
| `GET /healthz` | Liveness — process up |
| `GET /healthz/readiness` | Readiness — DB connectée, migrations OK |

**Monitoring externe — Uptime Kuma** (zero-dep, self-host) : monitor HTTP sur `/healthz/readiness` toutes les 60s + alerte Slack/email. Ajouter un 2ᵉ monitor keyword sur un webhook `/stats` qui check que `last_run_at > now - 24h` → détecte "le schedule n'a pas tourné".

**Prometheus optionnel** : `N8N_METRICS=true` expose `/metrics` (OpenMetrics). Variables utiles : `N8N_METRICS_INCLUDE_WORKFLOW_ID_LABEL=true`, `N8N_METRICS_INCLUDE_QUEUE_METRICS=true`.

### 3.8 Checklist "savoir si Veille Emploi a tourné sans UI"

1. `errorWorkflow` global Slack → KO appris en push
2. `CODE_ENABLE_STDOUT=true` + `N8N_LOG_FORMAT=json` → logs custom dans `docker logs`
3. Dernier node : `$getWorkflowStaticData('global')` met à jour `last_run_at` + `last_items_count`
4. Webhook `/stats` (workflow tiers) ou SQL direct → curl pour interroger
5. Uptime Kuma sur `/healthz/readiness` + monitor keyword sur `/stats`
6. `EXECUTIONS_DATA_SAVE_ON_PROGRESS=true` + rétention 7j → debug rapide en UI
7. Pour alerting volume : `IF items.length < seuil` → throw artificiel → retombe dans Error Workflow Slack (détecte les KO métier vs KO techniques)

---

## 4. Exportabilité & portabilité

### 4.1 `$env.X` et `N8N_BLOCK_ENV_ACCESS_IN_NODE` — comportement réel (vérifié empiriquement)

Depuis n8n v2.0, `N8N_BLOCK_ENV_ACCESS_IN_NODE=true` est le défaut.

> ⚠️ **Correction (test empirique n8n 2.15.0, 2026-06-01)** : contrairement à ce que cette section affirmait auparavant, `BLOCK=true` ne bloque **pas** seulement les Code nodes — il bloque `$env` **partout**, Code nodes *et* expressions de nodes natifs, de façon identique (la garde est posée sur le provider `$env` commun aux deux chemins). Sous `BLOCK=true`, `={{ $env.X }}` dans un champ URL d'un HTTP Request échoue tout autant qu'un `$env.X` en Code node (`access to env vars denied`).

**Table de vérité** (sonde Code node + sonde expression Set, croisées) :

| Node | `BLOCK=true` | `BLOCK=false` |
|---|---|---|
| Code node (`$env.X`) | ❌ `access to env vars denied` | ✅ résout |
| Expression native (`={{ $env.X }}`) | ❌ `access to env vars denied` | ✅ résout |

**Conséquences pratiques :**
- Pour utiliser `$env` **du tout** dans des workflows (Code node OU expression), il faut `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. Il n'existe pas de demi-mesure « expressions seulement ».
- La distinction **Code node pur vs expression** reste pertinente pour l'**auditabilité** (un Code node = JS arbitraire, vrai vecteur d'import malveillant ; une expression = fragment court lisible), **pas** pour l'accès `$env`.

**État de la factory (instance locale dev)** : `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` est appliqué **par choix assumé** (Mac M1, mono-utilisateur, localhost, pas de webhook public, Adrien seul auteur de workflows). C'est ce qui rend les conventions §4.2/§4.3 ci-dessous opérationnelles.

**Pré-requis d'extraction vers une instance prod** (= déclencheurs : instance dédiée prod, webhook public, multi-utilisateur, import de workflows tiers) : il faut alors `BLOCK=true` (défaut sécurisé), donc :
1. **Code nodes purs** : aucun `$env` en Code node → un node natif « Config » en tête lit la config et les Code nodes lisent `$('Config').first().json.x`.
2. **Config hors `$env`** : sortir les constantes vers les **Variables UI n8n** ou des credentials typés (les deux axes — pureté Code node et compat `BLOCK=true` — sont orthogonaux).
Effort estimé W1+W2 : ~3,5–4,5 h. Voir [[feedback_n8n_block_env_access]].

### 4.2 Magic strings → env vars

**Toute** valeur dépendante de l'environnement sort du workflow et passe par `$env.PROJECT_KEY_NAME`. Préfixe **par projet/domaine**, jamais par usage technique.

```yaml
# docker-compose.yml — anchor x-n8n, section environment (noms réels)
environment:
  # Infra factory (partagée, STABLE, ne voyage jamais)
  - OLLAMA_LOCAL_URL=http://host.docker.internal:11434
  - OLLAMA_CLOUD_URL=https://ollama.com
  - QDRANT_URL=http://qdrant:6333
  - N8N_BLOCK_ENV_ACCESS_IN_NODE=false   # requis pour $env partout (cf. §4.1)

  # Config-projet (préfixe <PROJ>_, reste en factory, voyage à l'archivage)
  - VEILLE_OUTPUT_DIR=/data/shared/veille-emploi
  - "VEILLE_TO_EMAIL=${VEILLE_TO_EMAIL:?defined in .env}"
  - VEILLE_BATCH_MAX=40
  - VEILLE_EMBED_MODEL=bge-m3
  - VEILLE_EXTRACT_MODEL_CLOUD=qwen3-coder:480b
```

⚠️ Toute var référencée en `${...}` (ex. `VEILLE_TO_EMAIL`) **doit rester dans le `.env` racine** : le compose s'en sert aussi comme source d'interpolation au boot. La déplacer ailleurs casse le démarrage.

Dans une expression node natif : `={{ $env.VEILLE_OUTPUT_DIR }}/{{ $now.toFormat('yyyy-MM-dd') }}.md`.

Les **secrets** (POSTGRES_PASSWORD, N8N_ENCRYPTION_KEY, N8N_API_KEY, OLLAMA_API_KEY) restent dans `~/.zshrc` (jamais `.env` commit) et sont injectés via interpolation `${VAR}` dans `docker-compose.yml`. Voir CLAUDE.md "Sécurité".

### 4.3 Path portability — bannir les paths hardcodés

| Niveau | Pattern |
|---|---|
| **Recommandé** | Expression `$env` dans node natif : `={{ $env.VEILLE_OUTPUT_DIR }}/jobs/{{ $now.toFormat('yyyy-MM-dd') }}.md` |
| Acceptable | Code node qui calcule depuis input upstream (sans hardcode) |
| À proscrire | `/data/shared/veille/jobs/` en dur dans plusieurs nodes |

> Note : « Recommandé » > « Acceptable » pour l'**auditabilité** (Code node pur), pas pour l'accès `$env` — les deux nécessitent `BLOCK=false` (cf. §4.1). En prod `BLOCK=true`, basculer vers le pattern Config node + Variables UI.

**Garde-fou côté n8n** : `N8N_RESTRICT_FILE_ACCESS_TO=/data/shared` (déjà en place dans docker-compose.yml) verrouille la racine et rend les paths explicites.

### 4.4 Export workflow JSON — 3 voies

| Méthode | Quand | Commande |
|---|---|---|
| `n8n_get_workflow` (MCP) | Dev itératif, snapshot ponctuel depuis Claude | MCP tool |
| API publique `GET /api/v1/workflows/{id}` | CI/CD, sync multi-instance, scripts | `curl -H "X-N8N-API-KEY: $N8N_API_KEY" ...` |
| CLI `n8n export:workflow --backup` | Backup massif, snapshot reproductible git | `docker compose exec n8n n8n export:workflow --backup --output=/data/shared/exports/$(date +%F)/` |

**Pattern factory** — snapshot quotidien versionnable :
```bash
docker compose exec n8n n8n export:workflow \
  --backup --pretty --separate \
  --output=/data/shared/exports/$(date +%F)/
# Un fichier par workflow, format pretty, diffs git propres
```

**Piège** : tous les exports incluent les credentials **par référence** (id + nom), pas leurs valeurs. Le JSON est inutile sur une autre instance tant que les credentials n'y existent pas avec le même nom (ou avec leur id réassigné manuellement).

### 4.5 Export credentials — `--decrypted` ou rien

L'API publique `GET /api/v1/credentials` ne renvoie **jamais** les champs sensibles. Inutile pour migrer.

| Cas | Commande |
|---|---|
| Même instance ou clone avec même `N8N_ENCRYPTION_KEY` | `n8n export:credentials --backup` puis réimport direct |
| Nouvelle instance avec autre clé | `n8n export:credentials --all --decrypted --output=...` puis réimport (chiffre avec nouvelle clé) |

**Règle d'or** : **ne jamais commiter** un fichier `--decrypted`. `.gitignore` bloque `**/credentials*.json`. Stocker dans `~/secrets/` ou password manager, puis `shred -u` après réimport.

### 4.6 `N8N_ENCRYPTION_KEY` — discipline absolue

**Ce que la clé chiffre** : tous les champs sensibles de `credentials_entity` (API keys, tokens OAuth, passwords). Pas les workflows.

**Si non définie explicitement** : n8n en génère une au 1er démarrage et la stocke dans `~/.n8n/config` (dans le volume Docker), **invisible**, dangereux : tu ignores qu'elle existe jusqu'au jour où tu détruis le volume.

**Règles** :
1. **Définir explicitement** dans `~/.zshrc` AVANT le 1er `docker compose up` : `openssl rand -base64 42`
2. **Backup hors machine** : password manager (1Password/Bitwarden) avec item dédié par instance
3. **Une clé par instance PHYSIQUE** : en solo M1 il n'y a qu'UNE instance (la factory) → une seule clé. Clé distincte seulement en cas d'EXTRACTION physique réelle vers une autre machine (rare).
4. **Si perdue** : tous les credentials illisibles **définitivement**. Workflow tourne côté logique mais chaque node `credentials.id=...` plante à l'exécution. Seul remède : recréer chaque credential à la main.

### 4.7 Versioning git — Postgres = source of truth opérationnelle, Git = audit humain

| Aspect | Postgres SoT | Git SoT |
|---|---|---|
| Édition | UI n8n fluide | Conflits sur JSON gigantesques |
| Historique | Table `workflow_history` (Enterprise feature) | `git log`, diffs lisibles |
| Rollback | UI restore version | `git checkout` + `n8n import:workflow` |
| Code review | Aucun | PR github |

**Modèle hybride recommandé** : Postgres opérationnel + Git snapshots datés. Workflow :
1. Édite dans UI n8n
2. Milestone atteint → `n8n export:workflow --id=X --output=workflows/<slug>.json --pretty`
3. `git add workflows/ && git commit -m "veille-emploi: add LLM filter step"`
4. Pas de pull-from-git automatique : git est lecture, pas écriture vers n8n

**Convention nommage** : `<kebab-case-workflow-name>.json` (jamais l'ID Postgres `sFfERYppMeBnFNeA.json` illisible).

### 4.8 Multi-instance sync — dev → prod

| Pattern | Cas |
|---|---|
| **Git + CLI** (solo dev) | `export:workflow --backup` → `git push` → autre instance `git pull` + `import:workflow --separate` |
| API publique POST | CI/CD, scripts — nettoyer `id/active/createdAt/updatedAt/tags` avant POST, activation séparée via `POST .../activate` |
| Source control intégré Enterprise (n8n v1.16+) | Repo Git lié, push/pull UI, unidirectionnel dev→prod |

### 4.9 Cycle de vie d'un projet : ARCHIVER (défaut) vs EXTRAIRE (exception)

Voir CLAUDE.md "Cycle de vie" pour le détail. **Principe** : l'instance n8n est unique et permanente ; **on ne déplace JAMAIS un projet hors du repo** — déplacer `shared/<projet>` hors de `./shared/` casse le bind-mount → le workflow écrit dans le vide (split-brain).

- **ARCHIVER** (défaut, projet stable ~2-3 sem) : snapshot JSON figé → `projets/<projet>/workflows/<id>.json` ; **retirer le bloc `## PROJECT:<nom>` de `MEMORY.md`** (le vrai levier de lisibilité) ; (option) `mv project_<nom>_*.md` vers archive mémoire. Le workflow continue de tourner. **Avant** : promouvoir toute leçon `durable` généralisable en `feedback_*` factory (sinon perdue avec le projet).
- **Données** (`archiver ≠ purger`) : si l'automatisation est terminée (plus aucun run), snapshot Qdrant puis purge `shared/<projet>/inputs/`.
- **EXTRAIRE** (exception rare, sur déclencheur EXTERNE seul : déploiement ailleurs / produit / isolation données client) : repo + instance n8n séparés, nouvelle `N8N_ENCRYPTION_KEY`. Jamais la maturité seule (8 Go ne tiennent pas 2× n8n + Ollama).

---

## 5. Performance & scaling

### 5.1 Quand rester en `regular mode` vs passer en `queue mode`

**Règle solo Mac M1 8 GB** : reste en **regular mode** tant que tu es < ~50 exec/min soutenues. Queue mode est un piège à ton échelle (RAM doublée pour zéro gain).

| Mode | Cas |
|---|---|
| `regular` (défaut) | Single instance, solo, < 50 exec/min, Ollama local sur même machine |
| `queue` | > 50 exec/min soutenues, webhooks publics avec pics, workflow long bloque les autres, multi-workers utile |

**Queue mode env vars** (pour plus tard) :
```yaml
EXECUTIONS_MODE=queue
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_HEALTH_CHECK_ACTIVE=true
OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true
N8N_GRACEFUL_SHUTDOWN_TIMEOUT=30
```

**Tradeoff** : queue mode interdit `N8N_DEFAULT_BINARY_DATA_MODE=filesystem` (forcé en `database` → gonfle Postgres), double la RAM idle (main + Redis + ≥ 1 worker). Sur 8 GB partagés avec Ollama, c'est non. **Benchmark officiel** : 162 req/s queue vs 23 req/s regular sur c5.4xlarge — gain réel à partir de dizaines d'exec/s concurrentes.

### 5.2 `N8N_CONCURRENCY_PRODUCTION_LIMIT` — ton vrai garde-fou

Par défaut, n8n ne limite **rien**. Sur M1 8 GB partagés avec Ollama, c'est dangereux : 5 webhooks simultanés → swap explose.

```yaml
environment:
  - N8N_CONCURRENCY_PRODUCTION_LIMIT=3   # 2-3 sur 8 GB, 5-10 sur 16 GB
```

Au-delà de la limite, n8n met les exécutions en **file FIFO interne** et attend qu'un slot se libère. Comportement propre, pas d'OOM. Ne s'applique qu'aux execs production (triggers/webhooks/schedules), pas aux manuels ni aux sub-workflows enfants.

**Couplage avec Ollama** : `N8N_CONCURRENCY_PRODUCTION_LIMIT` ≤ nb modèles simultanés que la RAM peut tenir. Pour llama3.2:3b (~2 GB chargé) sur M1 8 GB : 1 seul modèle à la fois, donc concurrence n8n = 1-2 si tous les workflows hittent Ollama.

### 5.3 Vrai parallélisme avec `executionOrder: v1`

Pour avoir des branches **vraiment parallèles**, le workflow doit être en `executionOrder: v1` (Settings → Execution order). v0 entrelace par profondeur et donne des résultats imprévisibles sur branches longues.

```
[Trigger] → [HTTP A]
         → [HTTP B]   ← 3 branches indépendantes
         → [Transform C]
         → [Merge "Combine by Position"] → [Output]
```

**Pièges** :
- IF qui split sans `Merge` avant la suite → duplication d'items
- `Loop Over Items` rebranché sur lui-même sans `Merge` → double comptage
- `Merge` en mode "Wait for All" obligatoire en fan-in (sinon fire N fois)

**Node.js mono-thread** : pas de parallélisme CPU intra-workflow. Le gain vient des I/O concurrents (await) et de la rotation entre branches pendant les attentes réseau.

### 5.4 Caching LLM — Ollama natif + staticData + Qdrant sémantique

**(a) Ollama prompt caching (gratuit, automatique)**. Ollama réutilise le KV cache si le **préfixe du prompt est identique byte-for-byte** ET si le modèle est encore chargé. Deux conditions :
- Ollama décharge après 5 min d'inactivité par défaut → forcer `keep_alive: 60m` (ou `-1` infini) sur les nodes Ollama
- Aucun timestamp/ID dynamique dans le system prompt — sortir tout ce qui varie vers le user prompt ou en queue

Gain : prefill 800 ms → 5 ms quand prefix en cache.

**(b) `getWorkflowStaticData` pour cache simple cross-execution** :
```js
const cache = $getWorkflowStaticData('global');
const key = `lookup_${$json.id}`;
if (cache[key] && Date.now() - cache[key].ts < 3600_000) {
  return [{ json: cache[key].data }];
}
const data = await this.helpers.httpRequest({...});
cache[key] = { data, ts: Date.now() };
return [{ json: data }];
```
Idéal pour : config, tokens refresh, dictionnaires lookup. **Ne pas y mettre des MB** (lu/écrit à chaque exec). Persiste seulement si exec OK.

**(c) Cache sémantique Qdrant** pour rattraper paraphrases :
- Embed la query → search Qdrant collection `llm_cache_semantic` cosine > 0.95 → réutilise la réponse cachée
- HIT rate observé en prod : 60-90 % avec semantic, 20-45 % avec hash exact

### 5.5 Batching HTTP — natif > Loop+Wait > sub-workflow per item

```
HTTP Request → Options → Add Option → Batching:
  - Items per Batch: 10        # parallélisme intra-batch (I/O concurrents)
  - Batch Interval: 1000       # ms entre batchs (rate-limit)
```

Sur endpoint REST simple : ~50-100 req/s avec batchSize=10. En boucle séquentielle : 5-10 req/s.

`Loop Over Items` (batchSize=50-200) à utiliser pour : rate-limit strict, gros volumes (>10k items, libération RAM entre batches), checkpoints. Pas par défaut.

### 5.6 Task runners — activés par défaut en v2

Depuis n8n v2.x, chaque Code node passe par un sub-process via task runner (`N8N_RUNNERS_ENABLED=true` défaut). Ajoute 5-50 ms d'overhead + JSON serialize/deserialize, mais **6x throughput** sur Code nodes lourds (main event loop libre).

```yaml
environment:
  - N8N_RUNNERS_ENABLED=true
  - N8N_RUNNERS_MODE=internal   # internal = sub-process partage uid/gid (OK local)
```

Conséquence pratique : **préférer expressions n8n natives** (`Set`/`Edit Fields`) aux Code nodes pour transforms simples. Le Code node n'est plus le no-cost option qu'il était.

### 5.7 Memory tuning Mac M1 8 GB

Trois leviers gratuits :

```yaml
n8n:
  environment:
    - NODE_OPTIONS=--max-old-space-size=2048   # cap heap n8n à 2 GB
    - N8N_DEFAULT_BINARY_DATA_MODE=filesystem  # binaires sur disque, pas RAM
    - N8N_PAYLOAD_SIZE_MAX=32                  # MB, limite webhook entrant
  mem_limit: 2.5g                              # marge ~512 MB au-dessus heap
```

Pourquoi :
- Heap Node.js défaut ≈ 1.7 GB, parfois 8192 imposé par le build script — sur 8 GB partagés avec Ollama, dangereux
- `filesystem` binary mode : nodes PDF/images passent de minutes à secondes (legacy `default` gardait en RAM toute l'exec)
- `mem_limit` < heap = SIGKILL immédiat. Toujours `mem_limit = heap + 512 MB`

**Docker Desktop M1** : check Settings → Resources → Memory. Si 4 GB alloués à la VM Linux et tu mets `max-old-space-size=4096` côté n8n, OOM garanti. Vise Docker VM = 6 GB, n8n heap = 2 GB, Postgres+Qdrant = 1.5 GB.

### 5.8 Sub-workflow overhead — pour la RAM, pas pour la perf CPU

Coût d'`Execute Sub-workflow` : ~50-200 ms init + écriture en `execution_entity`. **Justifié seulement si** : isolation mémoire (sub libère heap à la fin), réutilisation (DRY), changement de mode d'exécution.

**Quand inline gagne** : transforms simples < 10 ms, pas de réutilisation, volume < 1k items, logging dans la même exec.

**Quand sub-workflow gagne** : loop batchs sur gros volumes (libération RAM entre batches), bloc partagé 3+ workflows, timeout/error policy différente, prêt à offloader sur worker en queue mode plus tard.

### 5.9 Postgres tuning solo

```yaml
environment:
  - DB_POSTGRESDB_POOL_SIZE=10        # défaut 4, monte à 10-20 si concurrence
  - EXECUTIONS_DATA_PRUNE=true
  - EXECUTIONS_DATA_MAX_AGE=168       # 7 jours
  - EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
```

**Postgres lui-même** (settings via `postgres.conf` ou env Postgres) — pour container limité 1 GB :
```
shared_buffers = 256MB         # 12% RAM container, pas RAM host
work_mem = 16MB
max_connections = 50
effective_cache_size = 1GB
```
N'alloue **pas** 25% de 8 GB à Postgres : Ollama a besoin de la RAM.

**Maintenance mensuelle** : `VACUUM ANALYZE` sur `execution_entity` et `execution_data` — sans ça, soft-delete du pruning ne libère pas l'espace disque (bug n8n #13375 connu).

---

## 6. Coût & efficacité (LOCAL-FIRST poussé)

### 6.1 Routage LLM par complexité (politique CLAUDE.md)

```
[Trigger] → [Compute complexity score]
  → [Switch on score]
      ├─ simple (< 1500 tokens, classif, extraction JSON) → Ollama local llama3.2:3b
      └─ complex (raisonnement multi-étapes, code, > 4k contexte) → Ollama Cloud
```

Switch route au **node**, pas au credential. Un seul prompt, deux credentials.

**Ordre de grandeur** : 1000 emails parsés (800 in, 200 out) = **0 €** local, ~0.72 € Qwen3-Coder API, ~1.80 € Claude Haiku. Sur 365j × 200 emails/j : économie annuelle 260-650 € minimum vs Cloud payant.

### 6.2 Token optimization

| Levier | Gain |
|---|---|
| Truncate input agressif (6000 chars body email) | −50 à −70 % input tokens |
| `format: 'json'` strict | −30 à −50 % output tokens (pas de prose) |
| Zero-shot bien rédigé > few-shot | −500 à −3000 tokens/call |
| Instructions concises | −20 à −40 % system prompt |
| Compactage historique conversationnel | linéaire vs N² sur agents multi-tours |

**Combiné** sur tâche d'extraction : −60 à −85 % tokens.

### 6.3 Court-circuiter AVANT le LLM (gain massif)

L'appel LLM est l'étape **la plus chère** (temps, RAM, tokens). Tout filtre placé après est du gaspi.

**Pipeline efficace** :
```
[Trigger] → [Dedup hash Postgres/Qdrant] (skip si déjà vu)
         → [Regex/keyword filter] (drop si pas dans scope)
         → [Pré-scoring rule-based] (drop si < seuil heuristique)
         → [Batching multi-items dans 1 prompt LLM]   ← seulement ici
         → [Post-traitement]
```

**Batching multi-items** : au lieu de 25 appels classif, **un seul prompt** avec 25 items numérotés et retour JSON `[{id, label}, ...]`. Économies : **50-80 %** tokens (instructions transmises une fois). Sweet spot 10-25 items pour llama3.2:3b (hallucinations sur les derniers au-delà).

**Ordre de grandeur Veille Emploi** : 200 emails/j, 30 % doublons + 50 % drop regex → ~70 LLM calls effectifs au lieu de 200 (**−65 %**). Avec cache 40 % hit → ~10-15 calls effectifs (**−98 %**).

### 6.4 Cache LLM exact + sémantique

```js
// Cache exact via Postgres
const key = crypto.subtle.digest('SHA-256', model + prompt + temp);
const cached = await pg.query('SELECT response FROM llm_cache WHERE key = $1', [key]);
if (cached.rows.length) return cached.rows[0].response;
const response = await callLLM(...);
await pg.query('INSERT INTO llm_cache (key, response, created_at) VALUES ($1, $2, now())', [key, response]);
```

Hit rates production : **20-45 % cache exact**, **60-90 % cache sémantique Qdrant** (cf. § 5.4).

### 6.5 Free tier monitoring + scheduling smart

Ollama Cloud Free reset toutes les 5h (quota GPU-time). Patterns :
- **Cronner les workloads gourmands hors heures actives** (03:00) — pas de conflit avec ta session de travail
- **Séquencer les Cloud calls en début de fenêtre** 5h pour bénéficier du quota fresh
- **Workflows consolidateurs** (1 run/jour vs 24 runs horaires) si la donnée le permet → −23 calls free tier
- **Compteur `staticData` par credential** avec reset cron pour visualiser conso live
- **Alerting** : workflow Schedule daily qui SELECT count failed → ping Discord si > seuil

### 6.6 Storage costs — pruning sans pitié

Sans pruning : 500 exec/j → 5-15 GB/mois. Avec pruning agressif (3j / 5000 max / pas de succès) : < 500 MB stable.

```yaml
environment:
  - EXECUTIONS_DATA_PRUNE=true
  - EXECUTIONS_DATA_MAX_AGE=72             # 3 jours si dev itératif
  - EXECUTIONS_DATA_PRUNE_MAX_COUNT=5000
  - EXECUTIONS_DATA_SAVE_ON_SUCCESS=none   # ne garder QUE les erreurs
  - EXECUTIONS_DATA_SAVE_ON_ERROR=all
  - EXECUTIONS_DATA_SAVE_ON_PROGRESS=false
  - EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=false
  - EXECUTIONS_DATA_HARD_DELETE_BUFFER=1
```

`VACUUM ANALYZE` mensuel obligatoire (cf. § 5.9).

### 6.7 Compute costs — quand sortir du Mac

Quand workflows toujours-on (24/7) + besoin d'indépendance machine perso :

| Provider | Specs | Prix |
|---|---|---|
| **Hetzner Cloud CAX11 (ARM)** | 2 vCPU / 4 GB / 40 GB | **~4.59 $/mois** |
| Hetzner CPX21 | 3 vCPU / 4 GB / 80 GB | ~8 $/mois |
| DigitalOcean Basic | 2 vCPU / 4 GB / 80 GB | ~24 $/mois |

**Hetzner CAX11 imbattable** rapport perf/€. n8n est plus sensible à la RAM qu'au CPU. **Ne pas descendre sous 4 GB** (1 GB = crash sous charge).

LLM sur VPS = à éviter (GPU dédié 200-500 €/mois). Garde Ollama local sur Mac OU passe Ollama Cloud Pro (20 $/mois flat).

---

## 7. CI/CD & déploiement

### 7.1 Stratégies de déploiement workflow

| Pattern | Quand | Implémentation n8n |
|---|---|---|
| **Blue/Green** | Tout workflow critique non-rejouable (envoi mail, paiement) | Deux workflows `v1`+`v2`, swap `active` via API |
| **Canary** | Validation graduelle nouveau code | Switch node route % du trafic vers v2 (modulo executionId), increment 10→50→100 % |
| **Shadow mode** | Validation de qualité d'une refonte | Duplique workflow, désactive nodes write (Postgres/email) ou remplace par Set qui logge dans `shared/shadow/`, compare outputs offline |

`N8N_GRACEFUL_SHUTDOWN_TIMEOUT=30` (queue mode) pour laisser les exécutions en cours finir avant swap.

### 7.2 Tests automatisés workflow

**Stack** :
- **pinData** (axe 3) — épingle outputs pour rejouer avec entrée déterministe
- **Golden dataset** : `tests/golden/<workflow>/inputs/*.json` + `expected/*.json` versionnés
- **MCP `n8n_test_workflow`** — exécution avec pinData injecté, sans toucher creds externes
- **MCP `n8n_validate_workflow`** — lint structure (orphans, typeVersions, expressions)
- **MCP `n8n_autofix_workflow`** — correction auto patterns LLM

**Tools externes** :
- [`n8n-workflow-validator`](https://github.com/yigitkonur/n8n-workflow-validator) (Yigit Konur) — validation via vrai moteur n8n, JSON output machine-readable, intégrable en pre-commit ou CI

**AI evaluation pattern** : workflow Schedule lance daily contre golden dataset, LLM-judge score chaque réponse, alert si moyenne < seuil. Inputs qui échouent en prod → ajoutés au golden dataset.

### 7.3 Pipeline GitHub Actions

Pattern de référence (3 jobs séquentiels + approval manuel) :

```yaml
name: n8n-deploy
on: { push: { branches: [main] } }

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: find workflows -name "*.json" -exec jq empty {} \;
      - run: npx n8n-workflow-validator workflows/ --fail-on warn
      - run: node ci/validate.js   # AJV schema + anti-patterns custom

  deploy-staging:
    needs: validate
    steps:
      - run: |
          for f in workflows/*.json; do
            curl -X POST "${{ secrets.STAGING_URL }}/api/v1/workflows" \
              -H "X-N8N-API-KEY: ${{ secrets.STAGING_API_KEY }}" \
              -H "Content-Type: application/json" -d @"$f"
          done
      - run: node ci/smoke.js --env staging

  deploy-prod:
    needs: deploy-staging
    steps:
      - uses: trstringer/manual-approval@v1
        with: { approvers: your-github-username }
      - run: |
          curl -X POST "${{ secrets.PROD_URL }}/api/v1/source-control/pull" \
            -H "X-N8N-API-KEY: ${{ secrets.PROD_API_KEY }}"
```

L'endpoint `/source-control/pull` est dispo en Enterprise. Sur Community : remplacer par `n8n import:workflow` via SSH.

### 7.4 Promotion dev → staging → prod

Modèle officiel : **3 instances physiquement séparées** (chacune sa DB, ses creds, son Redis), reliées via Git :
- `dev` ↔ branche Git `dev`
- `staging` ↔ branche `staging`
- `prod` ↔ branche `production`

**Crédentials ne sont jamais sync par Git**, créées manuellement par env avec **les mêmes noms** → workflow JSON portable. Tags `prod-ready` filtrent ce qui pull sur prod. Variables n8n (`$vars.X`) varient par env, workflow identique.

**Anti-pattern** : 1 instance + tags "dev/prod" — pas d'isolation des creds, un workflow "dev" planté peut sécher Redis/Postgres et tuer la prod.

### 7.5 Production checklist (avant `active=true`)

Avant toute promotion prod :

- [ ] `errorWorkflow` global Slack défini
- [ ] `executionTimeout` < `EXECUTIONS_TIMEOUT_MAX`
- [ ] Retry on Fail actif sur tous HTTP/DB externes avec `maxTries` explicite
- [ ] Idempotency gate sur tout node qui write/send
- [ ] Webhooks authentifiés (token, basic auth, HMAC) — pas de `authentication: none`
- [ ] Credentials nommées par env (`prod-X`, pas de token en dur)
- [ ] Pas de secrets dans les logs (`Set` redact si nécessaire)
- [ ] `/metrics` exposé + 3 alertes min (error rate > 1 %/1h, exec time p95 > 2x baseline, queue depth)
- [ ] `RUNBOOK.md` testé, rollback exécutable < 10 min
- [ ] Taille raisonnable : < 15-20 nodes, < 3 branches majeures (sinon sub-workflows)
- [ ] Premier run prod monitoré humainement

**3 alertes prod minimum** : error rate > 1 % sur 1h, exec time p95 > 2× baseline, freshness (`last_success_timestamp(workflow="X") < 2× period`).

### 7.6 Rollback à 3 niveaux

1. **Workflow JSON** : `git checkout <commit> workflows/X.json` puis `curl -X PUT /api/v1/workflows/{id} -d @workflows/X.json`. **L'ID doit rester stable** entre versions (préserve webhookUrl).
2. **Désactivation rapide** : `curl -X POST /api/v1/workflows/{id}/deactivate` — réponse la plus rapide à un workflow qui spam. Scripter `scripts/kill-switch.sh <id>`.
3. **Postgres restore** : `pg_restore --clean --if-exists --dbname=n8n_db backup.dump`. **`N8N_ENCRYPTION_KEY` identique** sinon creds illisibles.

**Backup pattern** : workflow n8n schedulé qui export tous les workflows tag=`prod` vers GitHub avec commit messages auto. Restore testé une fois par trimestre (un backup non-testé n'est pas un backup).

---

## 8. Pro vs perso — ce qui change quand on quitte le solo

Cette section anticipe la bascule. **Aujourd'hui solo, tu peux ignorer**. À lire quand tu envisages : 2ᵉ utilisateur, 1ʳᵉ donnée client, ou prod 24/7 critique.

### 8.1 Licensing tiers

| Tier | Features clés | Prix 2026 (Cloud / mois) |
|---|---|---|
| **Community** (self-host) | Tout moteur + 500 intégrations, exec illimitées | 0 € |
| **Starter** (Cloud) | 2 500 exec/mois | 24 € |
| **Pro** (Cloud) | 10 000 exec/mois | 60 € |
| **Business** (Cloud + self-host avec licence) | + SSO + Git natif + envs + RBAC | 800 € (− 50 % si < 20 employés) |
| **Enterprise** | + log streaming + audit + external secrets + multi-main HA + custom roles | Devis |

**Seuil de bascule** : 2ᵉ utilisateur ou 1ʳᵉ donnée client. En Community, le **sharing** est limité (owner + créateur seulement) — bloquant à 3+ users.

### 8.2 Features bloquées en Community

- **Projects** (conteneurs équipe + folders + variables projet)
- **Source control natif Git** (push/pull UI)
- **Variables `$vars.X` UI** (workaround : env vars)
- **External secrets** (Vault, AWS SM, Azure KV, GCP SM, 1Password Connect)
- **Log streaming structuré** (syslog/webhook/Sentry/Loki)
- **SSO** (SAML, LDAP, OIDC)
- **RBAC granulaire** (custom roles, Project Viewer/Editor)
- **Multi-main HA** (queue mode reste dispo)

### 8.3 Patterns pro qui mappent depuis tes pratiques perso actuelles

Tes pratiques solo qui **restent valides** en pro sans dette :
- Routing local-first / Cloud-fallback via Switch → architecture portable
- `N8N_ENCRYPTION_KEY` hors machine → précurseur Vault
- Secrets via `~/.zshrc` → précurseur External Secrets
- Export Git `shared/exports/` → précurseur Source Control Enterprise
- Postgres comme backend (pas SQLite) → déjà compatible queue mode
- Naming `[PROJET]_<workflow>` → mappe directement vers Projects
- `N8N_BLOCK_ENV_ACCESS_IN_NODE=true` → bonne hygiène

Tes pratiques solo qui **deviendraient dette** à réviser en pro :
- Workflows dans Personal Space → migrer en **Projects** dès > 1 user
- 1 instance "tout" → 3 instances dev/staging/prod + branches Git
- Credentials manuelles UI → External Secrets + rotation auto
- Logs `docker compose logs` → log streaming structuré vers SIEM
- Pas de RBAC → mapping rôles + SSO OIDC
- Mono-instance n8n → queue mode + workers + Redis + Postgres managé
- Backup ad-hoc → backup WAL horaire + test restore trimestriel

### 8.4 High Availability (target pro 24/7)

```
[Load Balancer] → [main-1, main-2 (leader election via Redis)] ← Multi-main = Enterprise
                       ↓ enqueue
                  [Redis Bull]
                       ↓ dequeue
                  [worker-1, worker-2, ..., worker-N]
                       ↓
                  [Postgres répliqué Patroni/managé]
                  [S3 (binary externalisé)]
```

**Queue mode seul** (1 main + N workers) = Community OK, mais SPOF sur le main.

**Targets typiques** : RPO < 5 min (replication streaming Postgres), RTO < 15 min (failover auto Patroni/RDS Multi-AZ), DR test trimestriel + `N8N_ENCRYPTION_KEY` backup séparé.

### 8.5 SLOs typiques pro

| Indicateur | SLI Prometheus | Cible |
|---|---|---|
| Success rate | `(1 - rate(failed[5m]) / rate(total[5m])) * 100` | > 99 % |
| Latency p95 webhook | `histogram_quantile(0.95, http_request_duration)` | < 500 ms |
| Queue depth | `n8n_queue_jobs_waiting` | < 100 sur 5 min |
| Freshness scheduled | `last_success_timestamp(workflow="X")` | < 2× période |
| Instance health | `up{job="n8n"}` | > 99.9 % |

Prometheus + Grafana + Alertmanager → Slack/PagerDuty. Runbooks `runbooks/<alert>.md` par alerte (symptômes, diagnostic, remédiation).

**Important** : `/metrics` Prometheus est **dispo en Community**. C'est l'absence de log streaming structuré qui force à reconstruire à la main.

### 8.6 Synthèse perso → pro

| Phase | Volume | Pratiques minimales |
|---|---|---|
| **Bootstrap** (toi maintenant) | 1-2 workflows | Sticky Notes "why", export JSON quotidien Git, kill-switch scripté, backup Postgres quotidien, `N8N_ENCRYPTION_KEY` en PM |
| **Consolidation** | 3-5 workflows prod | + Production checklist intégrale, errorWorkflow global, validator pre-commit, pinData + golden dataset, conventional commits |
| **Industrialisation** | 5+ workflows ou 2ᵉ user | + Instance staging séparée, pipeline GitHub Actions complet, Flyway pour Postgres, blue/green sur critiques |
| **Échelle** | 10+ workflows ou users externes | + Canary/shadow, Prometheus+Grafana, runbooks par workflow, eval LLM-judge schedulée, Business licence |

**Règle d'or de promotion prod** : un workflow promu prod doit avoir une procédure de rollback exécutable en moins de 10 minutes. Si tu ne sais pas rollback, tu n'es pas prêt à promouvoir.

---

## Checklist nouveau workflow

Avant d'`active=true` un nouveau workflow :

- [ ] Nom respecte `[Domain] Action – vN`
- [ ] Tous les nodes ont un nom verbe + objet (pas de `Code1`, `HTTP Request`)
- [ ] Credentials nommées `<env>-<service>-<scope>`
- [ ] Aucune valeur métier hardcodée — `$env.X` partout dans les expressions
- [ ] `onError` explicite sur chaque HTTP Request / Gmail / OpenAI / AI Agent (`continueErrorOutput` recommandé)
- [ ] `errorWorkflow` configuré dans Workflow Settings (pointe sur `_global_error_handler`)
- [ ] Workflow Timeout configuré (Workflow Settings)
- [ ] Code nodes < 100 lignes, ou décomposés
- [ ] Mode Code node cohérent avec l'intention (forEach pour 1:1, forAllItems pour agrégats)
- [ ] Idempotency gate avant tout side-effect (write DB, send email, charge)
- [ ] Helper log JSON structuré dans les Code nodes (`{ts, level, workflow, exec, event, ...}`)
- [ ] Dernier node met à jour `staticData` (`runs`, `last_run_at`, `last_items_count`)
- [ ] Tag du workflow (`prod`, `dev`, `wip`) pour filtre exports
- [ ] Snapshot JSON commit dans `projets/<projet>/workflows/` après stabilisation

## Anti-patterns courants à fuir

- Code node `god` mélangeant I/O + parsing + transformation + rendu
- `runOnceForAllItems` qui fait `items.map(x => { ...HTTP... })` séquentiel sans Loop+Wait
- Magic strings/URLs répétés dans plusieurs nodes
- `JSON.parse(content)` direct sans `safeParse` + strip ` ```json...``` ` (cf. `feedback_llm_cloud_json_fence`)
- Workflow actif édité in-place sans clone → save → swap
- `continueRegularOutput` par paresse (masque les bugs en propageant des items vides)
- Commit d'un fichier credentials non chiffré dans git
- Hardcode de `/data/shared/foo/` au lieu de `$env.VEILLE_OUTPUT_DIR`
- Pas de monitoring proactif des OAuth Google → expiration silencieuse à J+7

---

## Références mémoire factory

Patterns transversaux capitalisés en feedback memories :

- `feedback_n8n_code_node_helpers` — `this.helpers.httpRequest`, `require` interdit, hash BigInt FNV-1a
- `feedback_n8n_gmail_get_pattern` — `getAll IDs + get full` au lieu de base64url manuel
- `feedback_n8n_credential_types` — typées vs génériques httpHeaderAuth
- `feedback_n8n_api_put_workflow` — quirks PUT API + renommage nodes
- `feedback_ollama_cloud_naming` — convention `:480b/:397b/...` 2026
- `feedback_llm_cloud_json_fence` — strip code fence avant `JSON.parse`
- `feedback_gmail_oauth_publish` — publier app Google en Production
- `feedback_security_transparency` — expliciter avant toute manip de secrets

## Sources d'autorité consultées

Doc officielle n8n (docs.n8n.io) sections : Best Practices, Error handling, Workflow Settings, Sub-workflows, Code node, Logging, Monitoring, CLI commands, Export/import, Environment variables, Source control, MCP tools reference.

Blog officiel n8n.io : "15 best practices for deploying AI agents in production".

Community (patterns validés) : "Never let your automations silently fail", "Stopping duplicate Stripe charges (idempotency)", "Google credentials expire every week", "Best practices for structuring workflows at scale".

Validé 2026-05-29.
