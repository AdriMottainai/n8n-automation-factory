# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Automation factory built on the self-hosted AI starter kit. Every new workflow/agent starts here, gets built and tested locally via n8n-mcp, then can be extracted to its own project when mature.

## Environnements & cibles

Cette factory est l'**instance LOCALE de dev** (Mac M1, mono-utilisateur, localhost, pas de webhook public) :
- **Pas de staging/prod séparés** : tous les workflows cohabitent dans la même instance n8n/Postgres. Un workflow « gradué » (`~/projets/n8n/<projet>/`) continue de tourner dans CETTE instance.
- **Cible = perso/pro local** : automations pour l'utilisateur, données personnelles (CV, emails). Aucune API LLM payante.
- **Conséquences assumées** : `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (OK en local mono-user) ; secrets en `~/.zshrc` + `.env` gitignoré.
- **Dépôt public** : `github.com/AdriMottainai/n8n-automation-factory` (portfolio) — infra + conventions + doc seulement ; jamais de secrets, CV ni données (`shared/`, `.env*`, `.claude/tasks/` gitignorés).
- **Bascule prod / multi-user** : déclencheur = instance dédiée, webhook public, ou import de workflows tiers → repasser `BLOCK=true` + Code nodes purs (voir `n8n/conventions.md §4.1`).

## Factory structure

Ce repo est la **factory** : infra + conventions + leçons capitalisées. Les workflows vivent en Postgres (n8n DB), pas dans le repo. Chaque projet actif a un couloir dédié qui **reste dans la factory** même une fois mature (on archive, on ne déménage pas — voir Cycle de vie).

```
self-hosted-ai-starter-kit/             ← FACTORY
├── docker-compose.yml                  ← infra (n8n, postgres, qdrant, ollama)
├── CLAUDE.md / README.md               ← conventions + porte d'entrée
├── .env / .env.example                 ← config non-sensible (secrets en ~/.zshrc)
├── .mcp.json / .claude/settings.json   ← MCP + permissions Claude
├── n8n/                                ← conventions.md + snapshots/config n8n
├── shared/                             ← bind mount → /data/shared (container n8n)
│   └── <nom-projet>/                   ← inputs + outputs projet (gitignored)
└── projets/                            ← métadonnées projets actifs
    └── <nom-projet>/
        ├── README.md                   ← 1-pager projet
        ├── analysis.md                 ← analyses & audits projet
        └── workflows/                  ← snapshots JSON optionnels
```

**Pourquoi cette séparation** : `shared/<projet>/` (que n8n voit comme `/data/shared/<projet>/`) porte les données ; `projets/<projet>/` porte les documents humains. Les deux **restent dans la factory** même une fois le projet mature (le runtime ne déménage pas — voir Cycle de vie).

### Démarrer un nouveau projet

**Brief — ce que l'utilisateur fournit** (le know-how n8n est DÉJÀ chargé : ce doc + `n8n/conventions.md` + mémoires factory ; l'utilisateur décrit juste l'automatisation) :
- **But** en 1 phrase · **Déclencheur** (schedule / webhook / fichier déposé / email / manuel)
- **Entrées** : sources + auth (API clé gratuite/OAuth/aucune ? RSS ? Gmail ? fichier `shared/` ? Postgres ?)
- **Traitement** : filtrer / classifier / extraire JSON / résumer / matcher / scorer — **modèle choisi selon le besoin** (coût / puissance / confidentialité / latence ; local Ollama, Ollama Cloud, ou API payante — voir « Choix du modèle »)
- **Sortie** : email / fichier `shared/<projet>/` / Slack / Postgres / Qdrant
- **Réussite** : à quoi ressemble un bon résultat · **Volume/run** (batching + plafond Ollama)
- **Contexte métier** : profil / préférences / mots-clés qui guident filtre & scoring · **`public_ok` ?** · **credentials déjà dispo**

**Process (Claude) — dans l'ordre, dès qu'une automatisation est demandée :**

0. **Vérifier l'env** : n8n up (`docker compose ps` / `curl localhost:5678/healthz`) + **n8n-mcp connecté** (présence des outils `n8n_*` / `search_nodes` ; au besoin `ToolSearch` "n8n"). → **Piloter n8n via les outils n8n-mcp, PAS l'API REST/curl.** Si MCP absent : le signaler + checklist « Si MCP ne se charge pas » (ci-dessous), proposer de le fixer **avant de coder** ; fallback REST seulement si l'utilisateur préfère avancer sans.
1. **Brief** : récupérer le brief ci-dessus ; demander ce qui manque (surtout *Réussite* et *Contexte métier*).
2. **Clarifier + concevoir** : reformuler le but, proposer l'**archi** (trigger → nodes → sortie), **choix du modèle** (arbitrer coût/puissance/confidentialité — voir « Choix du modèle »), sources/auth — **valider avec l'utilisateur avant de coder**.
3. **✅ CHECKPOINT conventions — AVANT de coder** : lire `n8n/conventions.md` (§1 archi/naming, §2 robustesse, §4 exportabilité + la section du type de workflow visé). Objectif : concevoir juste du 1er coup (naming, Code node < 100 lignes, `errorWorkflow`, idempotency, env vars préfixées) — jamais construire sans avoir relu les conventions.
4. **Créer les couloirs** : `mkdir projets/<nom>/` (+ `shared/<nom>/` si fichiers) ; `README.md` (front-matter `public_ok` défaut non, mission, IDs, état) ; project memory `project_<nom>_workflow.md` (`type: project`, `volatility: volatile`, `public_ok`) + bloc `## PROJECT:<nom>` dans `MEMORY.md`.
5. **Build via n8n-mcp** : `search_templates` → `n8n_deploy_template`, ou `search_nodes` + `get_node_essentials`/`validate_node` + `n8n_create_workflow`. Config-projet en env vars **préfixées `<PROJ>_`** (`${...}` → `.env` racine) ; credentials via `n8n_manage_credentials`. Valider en boucle : `n8n_validate_workflow` / `n8n_autofix_workflow`. Paths des nodes → `/data/shared/<nom>/...`.
6. **Tester sans casser** : sondes jetables (executeWorkflowTrigger + DELETE, cf. [[feedback_n8n_cli_execute_testing]]) pour $env/shapes/décodage ; puis l'utilisateur lance un **run UI** → j'analyse le funnel (volumes par étape + sortie).
7. **Itérer** jusqu'au critère de *Réussite* du brief.
8. **✅ CHECKPOINT conventions — FIN d'itérations** : repasser la « Checklist nouveau workflow » + « Anti-patterns à fuir » de `conventions.md`. Combler les manques (errorWorkflow, observabilité logs JSON, idempotency, timeouts) **avant** de déclarer le projet stable.
9. Stabilité ~2-3 semaines → **ARCHIVER** (voir Cycle de vie).

### Cycle de vie : ARCHIVER (défaut) vs EXTRAIRE (exception)

**Principe** : l'instance n8n est **unique et permanente**. Le runtime (workflow en Postgres) et la config (`<PROJ>_*` dans docker-compose) **restent dans la factory pour toujours**. On ne déplace **jamais** un projet hors du repo : déplacer `shared/<nom>` hors de `./shared/` casse le bind-mount → le workflow écrit dans le vide (split-brain).

**ARCHIVER** (par défaut, projet stable depuis ~2-3 semaines) = ranger la mémoire, pas déménager :
1. Snapshot JSON figé : `n8n_get_workflow` → `projets/<nom>/workflows/<id>.json` (strip `availableInMCP`/`binaryMode`).
2. **Retirer le bloc `## PROJECT:<nom>` de `MEMORY.md`** (le vrai levier de lisibilité du contexte ; le filesystem n'est pas le problème).
3. (Optionnel) `mv project_<nom>_*.md` vers une archive mémoire.

Le workflow continue de tourner. Désactiver si besoin : `POST /api/v1/workflows/{id}/deactivate`.

**Avant d'archiver** : promouvoir toute leçon `durable` généralisable (gotcha n8n réutilisable) d'un `project_<nom>_*` vers un `feedback_*` factory — sinon elle part avec le projet et est perdue pour les suivants. Un `project_<nom>_*` ne devrait contenir, à l'archivage, que de l'**état** (`volatile`).

**Données** (`archiver ≠ purger`) : si l'automatisation est **terminée** (plus aucun run prévu), snapshot Qdrant (`POST /collections/<col>/snapshots`) puis purge `shared/<nom>/inputs/` ; garder JSON + README.

**EXTRAIRE** (exception rare) = repo + instance n8n **séparés**. **Uniquement sur déclencheur externe** : déploiement ailleurs, projet devenu produit, ou isolation de données client/sensibles. Jamais la maturité seule (sur 8 Go, 2× n8n + Ollama = exclu). Le jour venu : factory privée + portfolio public séparé, et nouvelle `N8N_ENCRYPTION_KEY` / credentials pour l'instance extraite.

### Conventions n8n production

`n8n/conventions.md` contient les conventions factory pour construire des workflows robustes, debuggables, maintenables et exportables — **à lire avant de démarrer un nouveau workflow ou refactorer**. Couvre architecture/maintenabilité (naming, Code node limits, sub-workflows), robustesse (errorWorkflow, onError, timeouts, idempotency, OAuth refresh), observabilité (logs JSON, EXECUTIONS_DATA_SAVE_*, pinData, test_workflow MCP, staticData, healthz), et exportabilité ($env discipline, magic strings, CLI export/import, N8N_ENCRYPTION_KEY). Inclut checklist nouveau workflow et anti-patterns à fuir.

## Stack

Mac M1 8 GB : Ollama natif (`localhost:11434`, GPU Metal) + Docker (n8n `:5678`, postgres interne, qdrant `:6333`).

### Connexions depuis les nodes n8n (Docker)

| Cible | URL depuis n8n | URL depuis le host |
|---|---|---|
| Ollama | `http://host.docker.internal:11434` | `http://localhost:11434` |
| Qdrant | `http://qdrant:6333` | `http://localhost:6333` |
| Postgres | `postgres:5432` (env vars) | `localhost:5432` |
| Fichiers partages | `/data/shared` | `./shared/` |

### Docker lifecycle

| Commande | Effet |
|---|---|
| `docker compose stop` / `start` | Pause/reprise, conserve containers + volumes |
| `docker compose down` | Detruit containers, **conserve** volumes (donnees OK) |
| `docker compose down -v` | **Destructif** — detruit volumes, reset complet |
| `docker compose up -d` | Recree containers, **necessite** env vars dans le shell |

## n8n Credentials (premiere utilisation)

- **Ollama local** : type "Ollama", base URL = `http://host.docker.internal:11434/`
- **Ollama Cloud** : type "Ollama", base URL = `https://ollama.com`, API key = `$OLLAMA_API_KEY`
- **Qdrant** : type "Qdrant", URL = `http://qdrant:6333` (pas d'API key)

## Modeles

### Ollama local (gratuit, illimite)

| Modele | Taille | Usage |
|---|---|---|
| `llama3.2:3b` | 2 GB | Generation, classification, extraction, resume court |
| `nomic-embed-text` | 274 MB | Embeddings pour RAG/Qdrant (**a pull** : `ollama pull nomic-embed-text`) |

Contrainte M1 8 GB : **un seul modele charge a la fois**. Decharge auto apres 5 min.

### Ollama Cloud (free tier)

Meme API, endpoint `https://ollama.com`. Modeles avec suffixe `:cloud` (liste : ollama.com/search?c=cloud).

| Modele | Parametres | Force |
|---|---|---|
| `deepseek-v3.1:cloud` | 671B | Raisonnement complexe, long contexte |
| `qwen3-coder:cloud` | 480B | Generation et analyse de code |
| `qwen3.5:cloud` | variable | Generaliste puissant |

Free tier : 1 modele concurrent, usage "light" (resets 5h/7j).

## Choix du modèle / routage — ADAPTATIF (rien n'est imposé)

Le modèle **et** l'hébergement se choisissent **par automatisation**, en arbitrant 5 axes : **coût · puissance/qualité · confidentialité (sensibilité des données) · latence · volume**. `llama3.2:3b` local n'est qu'un **défaut commode du setup actuel** (M1, coût 0, données privées) — **pas une règle**.

| Option | Coût | Puissance | Confidentialité | Quand |
|---|---|---|---|---|
| **Ollama local** (`llama3.2:3b`…) | gratuit | faible (M1 = petits modèles, 1 à la fois) | max (rien ne sort) | simple, données sensibles, gros volume, hors-ligne |
| **Ollama Cloud** (free tier, gros modèles) | gratuit (light) | élevée | données → Ollama | raisonnement long, code, gros docs, sans budget |
| **API payante** (Claude, OpenAI…) | $ | max | données → fournisseur | quand la qualité justifie le coût **et** que la confidentialité l'autorise |

**Règle** : à la conception, déduire/demander les contraintes du projet (budget ? données sensibles ? besoin de puissance ?) et choisir en conséquence. Local-first reste un bon défaut **quand rien ne le contredit**, mais une API payante est légitime si le besoin la justifie. **Dans n8n** : un Switch peut router selon la complexité ; un credential par fournisseur utilisé. Voir [[feedback_llm_model_choice_adaptive]].

## MCP

### n8n-mcp (pilotage n8n)

| Outil | Usage |
|---|---|
| `n8n_health_check` | Sante de l'instance |
| `n8n_list_workflows` / `n8n_get_workflow` | Lister / inspecter |
| `n8n_create_workflow` / `n8n_generate_workflow` | Creer / generer depuis description |
| `n8n_update_full_workflow` / `n8n_update_partial_workflow` | Modifier |
| `n8n_validate_workflow` / `n8n_autofix_workflow` | Valider / corriger |
| `n8n_test_workflow` / `n8n_executions` | Tester / voir executions |
| `n8n_manage_credentials` | Gerer credentials |
| `n8n_deploy_template` | Deployer un template |
| `search_nodes` / `search_templates` | Chercher nodes ou templates |
| `get_node` / `validate_node` | Info / validation node |

### Context7 (documentation a jour)

`resolve-library-id` puis `query-docs`. Utiliser des qu'on touche a une lib/SDK/framework.

### Skills n8n (`/n8n-*`)

`/n8n-workflow-patterns`, `/n8n-node-configuration`, `/n8n-expression-syntax`, `/n8n-code-python`, `/n8n-code-javascript`, `/n8n-mcp-tools-expert`, `/n8n-validation-expert`

### Configuration MCP

n8n-mcp est en scope **project** (`.mcp.json`, gitignoré). `${N8N_API_KEY}` est une reference resolue au runtime depuis l'env shell. Pour que ca marche : la variable doit etre exportee dans `~/.zshrc` AVANT de lancer `claude`.

Context7 est en scope **user** (`~/.claude.json`), disponible dans tous les projets.

Si MCP ne se charge pas : verifier le trust dialog au premier lancement, que `$N8N_API_KEY` est dans l'env, que n8n tourne (`docker compose ps`).

## Patterns d'automatisation courants

_(Le process complet est dans « Démarrer un nouveau projet » ci-dessus. Voici des archétypes de départ.)_

**RAG local** : Local File Trigger (`/data/shared`) → chunk → embed (nomic-embed-text) → Qdrant. Chat Trigger → query Qdrant → llama3.2:3b.

**Classification/triage** : Trigger → llama3.2:3b classifie en JSON → Switch → actions par categorie → Postgres.

**Veille web** : Schedule Trigger → HTTP Request (RSS) → llama3.2:3b resume → `shared/veille/YYYY-MM-DD.md`.

**Pipeline docs** : fichier dans `shared/` → extraction → llama3.2:3b → `shared/output/`.

## Commandes

```bash
# Stack
docker compose ps                     # Statut
docker compose logs -f n8n            # Logs n8n
docker compose restart n8n            # Redemarrer n8n
docker compose up -d                  # Demarrer (env vars requises)
docker compose stop                   # Pause

# Ollama
ollama list                           # Modeles installes
ollama pull nomic-embed-text          # Modele d'embedding
curl http://localhost:11434/api/tags  # Check API

# Sante
curl http://localhost:5678/healthz    # n8n
curl http://localhost:6333/           # Qdrant
```

## Securite

**Secrets** dans `~/.zshrc` (jamais dans le repo) : `POSTGRES_USER`, `POSTGRES_PASSWORD`, `N8N_ENCRYPTION_KEY`, `N8N_USER_MANAGEMENT_JWT_SECRET`, `N8N_API_KEY`, `OLLAMA_API_KEY`.

**`.env`** : uniquement config non-sensible (`POSTGRES_DB`, `OLLAMA_HOST`, `N8N_DEFAULT_BINARY_DATA_MODE`).

**Permissions Claude Code** (`.claude/settings.json`) : whitelist stricte pour Docker/git/curl localhost. Deny sur `.env`, destructif, exfil API. Ask sur commit/push/install.

**`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`** (overridé dans `docker-compose.yml` ; défaut n8n v2 = `true`) : requis pour lire `$env` — `true` bloque `$env` **partout** (Code nodes ET expressions, vérifié empiriquement, pas seulement les Code nodes). Voir `n8n/conventions.md §4.1`.

**Backup critique** : `N8N_ENCRYPTION_KEY` — si perdue, tous les credentials n8n deviennent illisibles. Sauvegarder hors machine (password manager).

### Hardening a faire avant d'exposer des webhooks

- Verifier la protection SSRF (`host.docker.internal:11434` peut etre bloque)
- Signature HMAC-SHA256 sur webhooks entrants (node Crypto)
- Reverse proxy (Traefik/nginx/Caddy) pour rate limiting si exposition publique

## Extraire une automatisation

1. `n8n_get_workflow` → JSON dans `shared/exports/<nom>.json`
2. Documenter les dependances (credentials, modeles, services)
3. README minimal (quoi, comment configurer, comment lancer)
4. Si standalone : extraire un `docker-compose.yml` minimal
5. Partager : galerie templates n8n ou repo independant
